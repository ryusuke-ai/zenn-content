---
title: "AIエージェントの全行動を記録する ─ Observation-First アーキテクチャの設計と実装"
emoji: "👁️"
type: "tech"
topics: ["AI", "LLMOps", "TypeScript", "エージェント", "オブザーバビリティ"]
published: true
---

## はじめに

AI エージェントを本番環境で運用して最初にぶつかる壁は、「**何が起きたのかわからない**」という問題です。

LLM ベースのエージェントは非決定論的に動作します。同じ入力でも、使うツール、呼び出し順序、中間結果が毎回異なります。ユーザーから「さっきの処理、何をしたの？」と聞かれたとき、ログが残っていなければ答えようがありません。

私が開発・運用している **Argus** は、Claude Agent SDK を基盤としたマルチエージェントシステムです。Slack を入り口に、タスクの自律実行、Deep Research、メール処理、SNS 投稿など、日常業務の自動化を担っています。開発初期、エージェントが「成功しました」と報告しながら実際には何もしていなかったり、途中で失敗したのに気づけなかったりする問題が頻発しました。

この経験から生まれたのが **Observation-First Architecture** ── 「まず記録する、次に実行する」を設計原則に据えたアーキテクチャです。本記事では、その設計思想から具体的な実装パターン、運用で得られた知見までを解説します。

---

## なぜ Observation-First なのか

### ブラックボックス問題

従来のソフトウェアでは、コードの実行パスは決定論的です。入力 A に対して関数 B が呼ばれ、結果 C が返る。デバッグも再現も容易です。

しかし AI エージェントは違います。

```
ユーザー: 「明日の会議をカレンダーに追加して」

エージェントの実際の動作（毎回異なる）:
  1. カレンダーの既存予定を確認 → list_events
  2. 明日の日付を計算
  3. イベントを作成 → create_event
  4. 確認メッセージを生成
```

この動作ログが残っていなければ、「なぜ会議が重複登録されたのか」「なぜ日付が間違っていたのか」を後から追跡できません。

### 3つの落とし穴

運用を通じて、AI エージェントには特有の落とし穴があることがわかりました。

| 落とし穴               | 説明                                                     | 例                                                              |
| ---------------------- | -------------------------------------------------------- | --------------------------------------------------------------- |
| **偽の成功**           | LLM が「完了しました」と報告するが、実際には失敗している | メール送信を試みたが認証エラー → 「メールを送信しました」と報告 |
| **サイレントフェイル** | エラーがどこにも記録されず、問題に気づけない             | ツール呼び出しが途中で中断されたが、次のステップに進んでしまう  |
| **再現不能**           | 同じ入力でも異なる動作をするため、バグの再現ができない   | 特定の条件でのみ発生するツール呼び出しの組み合わせ              |

これらを解決するために、**ツール呼び出しの全てを、実行と同時に記録する**アーキテクチャが必要でした。

---

## 設計の全体像

Observation-First Architecture は 3 つの層で構成されています。

```
┌─────────────────────────────────────────────────────┐
│  Application Layer (Slack Bot / Orchestrator)        │
│    ├── SessionManager  ─────── 進捗通知・UXフィード │
│    └── InboxExecutor   ─────── 失敗検出・入力待ち   │
├─────────────────────────────────────────────────────┤
│  Observation Layer (agent-core)                      │
│    ├── ArgusHooks      ─────── 抽象フックIF          │
│    ├── buildSDKHooks() ─────── SDK形式への変換       │
│    └── createDBObservationHooks() ── DB記録          │
├─────────────────────────────────────────────────────┤
│  Storage Layer (db)                                  │
│    ├── sessions        ─────── セッション管理        │
│    ├── tasks           ─────── ツール実行ログ        │
│    └── lessons         ─────── エピソード記憶        │
└─────────────────────────────────────────────────────┘
```

**設計原則**: agent-core は DB パッケージに直接依存しません。`ObservationDB` インターフェースで DB 操作を注入する Dependency Injection パターンを採用しています。これにより、core は純粋な SDK ラッパーとして保たれ、消費側（Slack Bot、Orchestrator）が観測データの保存先を自由に決定できます。

---

## Hook ベースの記録レイヤー

### ArgusHooks: シンプルな抽象インターフェース

Observation-First の核は、Claude Agent SDK の Hook メカニズムに乗る形で設計した `ArgusHooks` インターフェースです。

```typescript
// packages/agent-core/src/hooks.ts

export interface ArgusHooks {
  onPreToolUse?: (event: {
    sessionId: string;
    toolUseId: string;
    toolName: string;
    toolInput: unknown;
  }) => Promise<void>;

  onPostToolUse?: (event: {
    sessionId: string;
    toolUseId: string;
    toolName: string;
    toolInput: unknown;
    toolResult: unknown;
  }) => Promise<void>;

  onToolFailure?: (event: {
    sessionId: string;
    toolUseId: string;
    toolName: string;
    toolInput: unknown;
    error: string;
  }) => Promise<void>;
}
```

わずか 3 つのコールバック。これだけで、エージェントのあらゆるツール呼び出しの**開始・成功・失敗**を捕捉できます。

### SDK フォーマットへのブリッジ

Claude Agent SDK は独自の `HookCallbackMatcher[]` 形式を要求します。`buildSDKHooks()` がこの橋渡しを行います。

```typescript
// packages/agent-core/src/hooks.ts

export function buildSDKHooks(
  argusHooks: ArgusHooks,
): Partial<Record<HookEvent, HookCallbackMatcher[]>> {
  const sdkHooks: Partial<Record<HookEvent, HookCallbackMatcher[]>> = {};

  if (argusHooks.onPreToolUse) {
    const callback = argusHooks.onPreToolUse;
    const hook: HookCallback = async (input, toolUseId, _options) => {
      const preInput = input as PreToolUseHookInput;
      await callback({
        sessionId: preInput.session_id,
        toolUseId: toolUseId ?? "",
        toolName: preInput.tool_name,
        toolInput: preInput.tool_input,
      });
      return {};
    };
    sdkHooks.PreToolUse = [{ hooks: [hook] }];
  }

  // PostToolUse, PostToolUseFailure も同様のパターン
  return sdkHooks;
}
```

**なぜこの変換層が必要か**: SDK の Hook 型は複雑で、`HookCallback`, `HookCallbackMatcher`, `HookEvent` といった多数の型が絡みます。消費側のアプリケーションコードがこれらの型を直接扱うと、SDK のバージョンアップ時の影響範囲が大きくなります。`ArgusHooks` で抽象化することで、SDK の内部構造から消費側を保護しています。

### DB 記録の実装

実際に全ツール呼び出しをデータベースに記録する `createDBObservationHooks()` の実装です。

```typescript
// packages/agent-core/src/observation-hooks.ts

export function createDBObservationHooks(
  obsDB: ObservationDB,
  dbSessionId: string,
  logPrefix: string,
): ArgusHooks {
  // toolUseId → { dbId, startTime } のマッピング
  const taskIds = new Map<string, { dbId: string; startTime: number }>();

  return {
    onPreToolUse: async (event) => {
      // ツール実行開始を即座に記録
      const [task] = await obsDB.db
        .insert(obsDB.tasks)
        .values({
          sessionId: dbSessionId,
          toolName: event.toolName,
          toolInput: event.toolInput,
          status: "running",
        })
        .returning();
      taskIds.set(event.toolUseId, {
        dbId: task.id,
        startTime: Date.now(),
      });
    },

    onPostToolUse: async (event) => {
      // 成功時: 結果と所要時間を記録
      const entry = taskIds.get(event.toolUseId);
      if (!entry) return;
      await obsDB.db
        .update(obsDB.tasks)
        .set({
          toolResult:
            typeof event.toolResult === "string"
              ? { text: event.toolResult }
              : event.toolResult,
          status: "success",
          durationMs: Date.now() - entry.startTime,
        })
        .where(obsDB.eq(obsDB.tasks.id, entry.dbId));
    },

    onToolFailure: async (event) => {
      // 失敗時: エラーを記録 + lessons に教訓を自動保存
      const entry = taskIds.get(event.toolUseId);
      if (!entry) return;
      await obsDB.db
        .update(obsDB.tasks)
        .set({
          toolResult: { error: event.error },
          status: "error",
          durationMs: Date.now() - entry.startTime,
        })
        .where(obsDB.eq(obsDB.tasks.id, entry.dbId));

      // エピソード記憶に自動記録
      await obsDB.db.insert(obsDB.lessons).values({
        sessionId: dbSessionId,
        taskId: entry?.dbId,
        toolName: event.toolName,
        errorPattern: event.error,
        reflection: `Tool ${event.toolName} failed with input: ${JSON.stringify(
          event.toolInput,
        ).slice(0, 500)}`,
        severity: "medium",
      });
    },
  };
}
```

**重要なポイント**:

- `onPreToolUse` の時点で DB に `running` ステータスで INSERT しています。もしエージェントプロセスがクラッシュしても、「実行中のまま止まったツール呼び出し」が DB に残り、何が起きていたかを追跡できます。
- `onPostToolUse` では `toolResult` が文字列の場合に `{ text: toolResult }` に変換しています。JSONB カラムに格納するために、常にオブジェクト形式に正規化するためです。
- `onToolFailure` では失敗時の入力情報も `reflection` に含めています。後から「何を渡したときに失敗したか」を特定できるようにするためです。

---

## データモデル: ツール実行ログとエピソード記憶

### tasks テーブル ── ツール実行の全記録

```typescript
// packages/db/src/schema.ts

export const tasks = pgTable("tasks", {
  id: uuid("id").primaryKey().defaultRandom(),
  sessionId: uuid("session_id")
    .references(() => sessions.id)
    .notNull(),
  toolName: varchar("tool_name", { length: 255 }).notNull(),
  toolInput: jsonb("tool_input"),
  toolResult: jsonb("tool_result"),
  durationMs: integer("duration_ms"),
  status: varchar("status", { length: 50 }).notNull(), // running | success | error
  createdAt: timestamp("created_at").defaultNow().notNull(),
});
```

このテーブルには、エージェントが実行した**全てのツール呼び出し**が記録されます。ファイルの読み書き、Web 検索、メール送信、カレンダー操作 ── あらゆる操作の入力・出力・所要時間・成否が後から参照可能です。

### lessons テーブル ── エピソード記憶

```typescript
export const lessons = pgTable("lessons", {
  id: uuid("id").primaryKey().defaultRandom(),
  sessionId: uuid("session_id")
    .references(() => sessions.id)
    .notNull(),
  taskId: uuid("task_id").references(() => tasks.id),
  toolName: varchar("tool_name", { length: 255 }).notNull(),
  errorPattern: text("error_pattern").notNull(),
  reflection: text("reflection").notNull(),
  resolution: text("resolution"),
  severity: varchar("severity", { length: 50 }).notNull().default("medium"),
  tags: jsonb("tags").$type<string[]>().default([]),
  createdAt: timestamp("created_at").defaultNow().notNull(),
});
```

lessons テーブルは、エージェントの「**経験から学ぶ**」仕組みの基盤です。ツール呼び出しが失敗するたびに、エラーパターン・考察・解決策が自動的に蓄積されます。

### セッション ↔ ツール ↔ 教訓の関係

```
sessions (1:N) → tasks (ツール呼び出しログ)
                    ↓
                 lessons (失敗から生まれた教訓)
```

1 セッション（= 1 Slack スレッド）に複数のツール呼び出しが紐づき、その中の失敗から教訓が自動生成される。この構造により、「あるセッションで何が起きて、何を学んだか」を完全に再構成できます。

---

## Episodic Memory: 教訓のフィードバックループ

### formatLessonsForPrompt: 過去の失敗を未来に活かす

記録した教訓は、蓄積するだけでは意味がありません。Argus は過去の教訓を**次のセッションのシステムプロンプトに自動注入**します。

```typescript
// packages/agent-core/src/lessons.ts

export function formatLessonsForPrompt(lessons: LessonEntry[]): string {
  if (lessons.length === 0) return "";

  const entries = lessons.map((lesson, i) => {
    const resolution = lesson.resolution
      ? `  Resolution: ${truncate(lesson.resolution, 500)}`
      : "  Resolution: (未解決)";
    return [
      `${i + 1}. [${lesson.severity.toUpperCase()}] ${lesson.toolName}`,
      `  Error: ${truncate(lesson.errorPattern, 500)}`,
      `  Reflection: ${truncate(lesson.reflection, 500)}`,
      resolution,
    ].join("\n");
  });

  return [
    "",
    "# Past Lessons (avoid repeating these mistakes)",
    "",
    ...entries,
  ].join("\n");
}
```

### セッション開始時の注入フロー

```typescript
// apps/slack-bot/src/session-manager.ts (簡略化)

async handleMessage(session, messageText) {
  // 1. 観測フックを生成
  const hooks = this.createObservationHooks(session.id);

  // 2. 直近の教訓を DB から取得
  const recentLessons = await db
    .select()
    .from(lessons)
    .orderBy(desc(lessons.createdAt))
    .limit(10);

  // 3. 教訓テキストをシステムプロンプトに追加
  const lessonsText = formatLessonsForPrompt(recentLessons);
  const sdkOptions = {
    systemPrompt: {
      append: basePrompt + (lessonsText || ""),
    },
  };

  // 4. 実行（hooks で全ツール呼び出しを記録）
  const result = await query(messageText, { hooks, sdkOptions });
}
```

このフィードバックループにより、エージェントは以下のようなサイクルで「学習」します。

```
セッション A: メール送信失敗（Gmail トークン期限切れ）
  → lessons に記録: "Gmail API 401 - トークンリフレッシュが必要"
       ↓
セッション B: システムプロンプトに教訓が注入される
  → エージェントが事前にトークン状態を確認してから送信を試みる
```

### 教訓の鮮度管理

注意点として、**古くなった教訓は害になる**ことがあります。例えば「Gmail トークンが見つかりません」という教訓が残っていると、トークンを再設定した後もエージェントがメール機能をスキップし続けます。

対策として、Argus では以下を実施しています。

- `limit(10)` で直近の教訓のみを注入（古い教訓は自動的に押し出される）
- `resolution` フィールドに解決策が記録されたら、次回以降は「解決済み」として注入される
- 定期クリーンアップで一定期間を超えた教訓を削除

---

## 消費側のデコレータパターン

`createDBObservationHooks()` は基本的な DB 記録機能を提供しますが、各アプリケーションはこの上に**独自の関心事をデコレータとして重ねる**ことができます。

### Slack Bot の例: 進捗通知の追加

```typescript
// apps/slack-bot/src/session-manager.ts

private createObservationHooks(
  dbSessionId: string,
  onProgress?: (message: string) => Promise<void>,
): ArgusHooks {
  // ベース: DB 記録フック
  const obsDB: ObservationDB = { db, tasks, lessons, eq };
  const baseHooks = createDBObservationHooks(obsDB, dbSessionId, "[SessionManager]");

  if (!onProgress) return baseHooks;

  // デコレータ: 進捗通知を追加（5秒スロットル）
  let lastProgressTime = 0;
  return {
    ...baseHooks,
    onPreToolUse: async (event) => {
      await baseHooks.onPreToolUse!(event);  // 元の DB 記録は必ず実行

      // Slack への進捗通知（スロットル付き）
      const progressMsg = formatToolProgress(event.toolName, event.toolInput);
      if (progressMsg) {
        const now = Date.now();
        if (now - lastProgressTime >= 5000) {
          lastProgressTime = now;
          await onProgress(progressMsg);
        }
      }
    },
  };
}
```

ここでのポイントは、`formatToolProgress()` がツール名と入力から自然な日本語メッセージ（例: 「パッケージをインストールしています」「ファイルを検索しています」）を生成していることです。ユーザーには内部のツール名ではなく、理解しやすい進捗メッセージが届きます。

この設計の美しさは、**DB 記録とアプリケーション固有の処理が疎結合**であることです。Slack Bot は進捗通知を追加し、Dashboard は WebSocket で状態を配信し、Orchestrator はメトリクスを収集する ── すべて同じ `ArgusHooks` インターフェースの上に成り立っています。

---

## Inbox Agent の失敗検出

Observation-First の恩恵が最も顕著に表れるのが、Inbox Agent の失敗検出機構です。

### 問題: SDK は「成功」と言うが、実際は失敗

Claude Agent SDK は、エージェントがレスポンスを返せば `success: true` を報告します。しかし、エージェントのテキスト出力が「認証エラーが発生しました」「メールを送信できませんでした」と言っている場合、これは本当に成功でしょうか？

### 解決: テキスト末尾のパターンマッチ

```typescript
// apps/slack-bot/src/handlers/inbox/executor.ts

private detectTaskFailure(resultText: string): boolean {
  // 結論部分（末尾500文字）のみ検査
  const tail = resultText.slice(-500);

  const failurePatterns = [
    /失敗しました/,
    /できません/,
    /できませんでした/,
    /エラーが発生/,
    /認証.{0,10}(?:エラー|未設定|必要)/,
    /アクセスできない/,
    /トークンが.{0,10}(?:ない|未設定|見つから)/,
    /No .{0,20} tokens? found/i,
    /authentication (?:failed|required|error)/i,
  ];

  return failurePatterns.some((p) => p.test(tail));
}
```

**末尾 500 文字に限定する理由**: エージェントは途中で一時的なエラーに遭遇してリトライし、最終的に成功することがあります（例: API のレートリミットに当たった → リトライ → 成功）。全文を検査すると、途中の失敗報告を拾って偽陽性が発生します。「結論」が書かれる末尾に絞ることで、最終結果のみを判定します。

### ユーザー入力待ちの検出

もうひとつの重要な検出は、エージェントが「質問を返してきた」場合です。

```typescript
private detectPendingInput(resultText: string): boolean {
  const questionMarks = (resultText.match(/？|\?/g) || []).length;
  return questionMarks >= 3;
}
```

Inbox Agent は「メッセージを投げたら結果が返る」ことを期待するシステムです。エージェントが「どの形式がいいですか？」「対象範囲はどこまでですか？」「優先度はどれですか？」と質問を返してきた場合、タスクを「入力待ち」(waiting) 状態にしてユーザーに通知する必要があります。

### 統合: executeTask の判定ロジック

```typescript
async executeTask(task): Promise<ExecutionResult> {
  const result = await query(task.executionPrompt, { hooks, timeout, sdkOptions });
  const resultText = this.extractText(result);

  const taskFailed = this.detectTaskFailure(resultText);
  const needsInput = this.detectPendingInput(resultText);

  return {
    // SDK 成功 かつ テキスト上も失敗していない
    success: result.success && !taskFailed,
    needsInput,
    resultText,
    sessionId: result.sessionId,
    costUsd: result.message.total_cost_usd,
  };
}
```

この二重チェックにより、「SDK は成功と言っているが実際は失敗」「成功したが追加入力が必要」という、LLM エージェント特有のグレーゾーンを正しくハンドリングしています。

---

## Dashboard での可視化

記録したデータは、Next.js 16 ベースのダッシュボードで可視化しています。

### セッション詳細ページ

各セッションの詳細ページでは、以下を時系列で確認できます。

- **メッセージ**: ユーザーとエージェントのやり取り
- **ツール呼び出し**: 各ツールの名前・入力・出力・所要時間・成否
- **コスト**: セッション全体の API 使用料

### エージェント実行履歴

Orchestrator が定期実行するエージェント（Gmail チェック、デイリープランナー、コードパトロール等）の実行履歴を一覧で確認できます。

```
ダッシュボードのページ構成:

/              ── トップページ
/sessions      ── セッション一覧
/sessions/[id] ── セッション詳細（メッセージ + ツール呼び出し）
/knowledge     ── ナレッジ管理
/agents        ── エージェント実行履歴
/files         ── 生成ファイルブラウザ
```

ダッシュボードは「エージェントが何をしたか」を人間が後から確認するためのインターフェースです。ツール呼び出しの全記録があるからこそ、「このセッションで何が起きたか」を完全に再構成できます。

---

## クラッシュリカバリ

Observation-First は、異常系でもデータの整合性を保つ設計になっています。

### 孤立タスクの復旧

エージェントプロセスがクラッシュすると、`running` ステータスのまま放置されたタスクが DB に残ります。Inbox Agent は起動時に `recoverAndResumeQueue()` でこれを検出・復旧し、未処理のキューも再開します。

```typescript
// apps/slack-bot/src/handlers/inbox/index.ts

async function recoverAndResumeQueue() {
  // running のまま放置されたタスクを queued に戻す
  const orphaned = await db
    .select()
    .from(inboxTasks)
    .where(eq(inboxTasks.status, "running"));

  for (const task of orphaned) {
    await db
      .update(inboxTasks)
      .set({ status: "queued", startedAt: null })
      .where(eq(inboxTasks.id, task.id));
  }
}
```

これが可能なのは、`onPreToolUse` で**ツール実行開始時に即座に DB に記録している**からです。「記録してから実行」という順序が、クラッシュ時のリカバリを可能にしています。

---

## 運用で得た知見

### 1. 教訓の鮮度管理は必須

lessons テーブルに古い教訓が残り続けると、すでに解決済みの問題に対してエージェントが過剰に慎重になります。

**実例**: 「Gmail トークンが見つかりません」という教訓が残っていた結果、トークン再設定後もエージェントがメール送信をスキップし続けた。

**対策**: 定期クリーンアップスクリプトで古い教訓を削除。`resolution` が埋まっている教訓は一定期間後に自動アーカイブ。

### 2. スロットルは必須

Slack への進捗通知を全ツール呼び出しで送ると、1 セッションで数十件の通知が飛びます。5 秒のスロットルを入れることで、ユーザー体験とシステム負荷のバランスを取っています。

### 3. 末尾検査の文字数は調整が必要

`detectTaskFailure()` の検査対象を末尾 500 文字にしていますが、エージェントが長い「まとめ」を書く場合、失敗パターンが 500 文字の外に出ることがあります。一方、範囲を広げると偽陽性が増えます。この数値は運用しながら調整を続けている部分です。

### 4. Observation は「コスト」ではなく「投資」

全ツール呼び出しの DB 記録は、1 セッションあたり数十の INSERT/UPDATE を生みます。パフォーマンスへの影響を心配しましたが、実際には PostgreSQL が十分に高速で、体感できるオーバーヘッドはありませんでした。一方で、問題発生時の調査時間が劇的に短縮されました。

### 5. DI パターンが拡張を支える

`createDBObservationHooks()` が `ObservationDB` インターフェースを受け取る設計のおかげで、テストでは本物の DB の代わりにモックを注入できます。また、将来的に別のストレージ（例: ClickHouse でのメトリクス記録）を追加する場合も、core を変更せずに対応できます。

---

## まとめ

Observation-First Architecture のエッセンスを整理します。

| 原則                 | 実装                                                             |
| -------------------- | ---------------------------------------------------------------- |
| **記録してから実行** | `onPreToolUse` で開始時に即座に DB INSERT                        |
| **全てを記録**       | 成功・失敗・所要時間を漏れなく記録                               |
| **失敗から学ぶ**     | lessons テーブル + formatLessonsForPrompt でフィードバックループ |
| **二重チェック**     | SDK の成否 + テキストパターンマッチの組み合わせ                  |
| **疎結合**           | ArgusHooks で SDK と消費側を分離、デコレータで拡張               |

AI エージェントの運用において、「何が起きたかわからない」は致命的です。Observation-First は、エージェントの全行動を記録し、そこから学び、その学びを次のセッションにフィードバックする ── この循環を設計の根幹に据えるアプローチです。

LLM ベースのエージェントを本番で運用する方にとって、何かの参考になれば幸いです。

---

## 参考

- [Claude Agent SDK ドキュメント](https://docs.anthropic.com/en/docs/agents/claude-code-sdk-reference)
- [Drizzle ORM](https://orm.drizzle.team/)
- [OpenTelemetry ─ Observability フレームワーク](https://opentelemetry.io/)
