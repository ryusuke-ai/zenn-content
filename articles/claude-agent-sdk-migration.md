---
title: "Claude Agent SDK でマルチエージェントシステムを作った話 ― CLI spawn からの脱却"
emoji: "🤖"
type: "tech"
topics: ["claude", "agent-sdk", "typescript", "ai", "anthropic"]
published: true
---

## はじめに

AI エージェントを本番運用していると、「CLI を子プロセスで叩く」アプローチの限界にぶつかります。出力パースが壊れる、リアルタイムにツール実行を観測できない、セッション管理が煩雑——。

この記事では、個人で開発・運用している **Argus** というマルチエージェントシステムにおいて、Claude Code CLI のプロセス起動から **Claude Agent SDK** (`@anthropic-ai/claude-agent-sdk`) へ移行した設計と実装を紹介します。

Argus は Slack をインターフェースとして、タスク実行・メール管理・カレンダー操作・SNS コンテンツ生成などを自律的に行うシステムです。pnpm monorepo で 12 パッケージ、1,165 以上のテストを持つ TypeScript プロジェクトで、Railway VPS 上で 24 時間稼働しています。

## 移行前の課題: CLI spawn の何がつらかったか

移行前のアーキテクチャはこうでした。

```
Slack メッセージ
  → slack-bot が受信
  → child_process.spawn("claude", [...args])
  → stdout を行ごとにパース (stream-parser.ts)
  → JSON-lines から結果を組み立て
```

これには 4 つの本質的な問題がありました。

### 1. 壊れやすいストリームパース

CLI の出力フォーマットはドキュメント化されておらず、バージョンアップのたびに変わりました。JSON-lines を 1 行ずつパースして状態機械で組み立てるコードは、CLI のアップデートが来るたびにメンテナンスが必要でした。

### 2. リアルタイム観測の不可能さ

Argus の設計思想は **"Observation-First"** ——エージェントのすべてのツール実行を記録し、後から何が起きたかを再構成できることを最重要視しています。しかし CLI spawn では `PreToolUse` / `PostToolUse` のフックが存在せず、ログ行を事後パースするしかありませんでした。

### 3. セッション再開の複雑さ

Slack の 1 スレッド = 1 セッションというモデルを実現するには、ディスク上のセッションファイルを管理して `--resume` フラグを渡す必要がありました。プログラム的にセッション継続を保証する手段がなく、エッジケースの処理が増え続けました。

### 4. PATH の罠

`which claude` で CLI バイナリの場所を探すアプローチは、子プロセスから呼ぶと PATH が正しく継承されず失敗することがありました。

## Claude Agent SDK の登場

Anthropic がリリースした `@anthropic-ai/claude-agent-sdk` (v0.2.34) は、これらの問題をすべて解決するプログラマティック API を提供します。

- `query()` が `AsyncGenerator<SDKMessage>` を返す
- 型付きメッセージ (`SDKSystemMessage`, `SDKAssistantMessage`, `SDKResultMessage`)
- ネイティブ Hook サポート (`HookCallbackMatcher[]`)
- セッション再開が `resume` オプション 1 つで完結

## アーキテクチャ設計

### レイヤー構成

移行後のアーキテクチャは 3 層構造です。

```
┌─────────────────────────────────────────────┐
│  消費側 (slack-bot / orchestrator / dashboard)│
│  → import { query, resume } from agent-core  │
│  → ArgusHooks で DB 書き込みを注入            │
├─────────────────────────────────────────────┤
│  @argus/agent-core                           │
│  → buildOptions(): AgentOptions → SDK Options│
│  → consumeSDKStream(): AsyncGenerator → Result│
│  → buildSDKHooks(): ArgusHooks → SDK Hooks   │
├─────────────────────────────────────────────┤
│  @anthropic-ai/claude-agent-sdk              │
│  → query() / resume (AsyncGenerator)         │
│  → HookCallbackMatcher[]                     │
└─────────────────────────────────────────────┘
```

設計のポイントは **消費側のインポートを一切変えない** こと。`agent-core` パッケージが SDK の複雑さを吸収し、slack-bot や orchestrator は移行前と同じ `query()` / `resume()` を呼ぶだけです。

### Session-per-Thread モデル

Slack スレッドと Claude セッションを 1:1 で対応させる設計は SDK 移行でシンプルになりました。

```
新規スレッド → query(prompt, options)
               → sessionId を DB に保存

スレッド返信 → resume(sessionId, message, options)
               → 既存セッションを継続
               → 失敗時は query() にフォールバック
```

## 実装の核心

### `consumeSDKStream()`: AsyncGenerator の消費

SDK の `query()` は `AsyncGenerator<SDKMessage>` を返します。これを `AgentResult` に正規化するのが `consumeSDKStream()` です。

```typescript
async function consumeSDKStream(
  stream: AsyncGenerator<SDKMessage, void>,
): Promise<AgentResult> {
  let sessionId: string | undefined;
  let resultText = "";
  let totalCost = 0;
  let isError = false;
  let hasResult = false;
  const toolCalls: ToolCall[] = [];
  const contentBlocks: Block[] = [];

  for await (const msg of stream) {
    switch (msg.type) {
      case "system":
        // init メッセージから sessionId を取得
        sessionId = msg.session_id;
        break;

      case "assistant": {
        // テキストブロックとツール使用ブロックを収集
        sessionId = msg.session_id;
        if (msg.message?.content) {
          for (const block of msg.message.content) {
            if (block.type === "text") {
              contentBlocks.push({ type: "text", text: block.text });
            } else if (block.type === "tool_use") {
              contentBlocks.push({
                type: "tool_use",
                name: block.name,
                input: block.input,
                tool_use_id: block.id,
              });
              toolCalls.push({
                name: block.name,
                input: block.input as unknown,
                status: "success",
              });
            }
          }
        }
        break;
      }

      case "result": {
        // 最終結果: コスト、成功/失敗ステータス
        sessionId = msg.session_id;
        totalCost = msg.total_cost_usd;
        hasResult = true;
        if (msg.subtype === "success") {
          resultText = msg.result;
          isError = msg.is_error;
        } else {
          isError = true;
        }
        break;
      }
    }
  }

  return {
    sessionId,
    message: {
      type: "assistant",
      content: contentBlocks,
      total_cost_usd: totalCost,
    },
    toolCalls,
    success: !isError,
  };
}
```

SDK のメッセージは 3 種類です。

| 型                    | 発火タイミング          | 取得できる情報              |
| --------------------- | ----------------------- | --------------------------- |
| `SDKSystemMessage`    | セッション開始時        | `session_id`, モデル情報    |
| `SDKAssistantMessage` | テキスト/ツール実行ごと | `content` ブロック配列      |
| `SDKResultMessage`    | 完了時                  | `total_cost_usd`, 成功/失敗 |

### Hook Injection: `ArgusHooks` → SDK Hooks のブリッジ

Argus の設計で最も重要なのが **すべてのツール実行を記録する** ことです。SDK のフック機構は強力ですが、`HookCallbackMatcher[]` という複雑な型を消費側に直接使わせたくありません。

そこで、シンプルな `ArgusHooks` インターフェースを定義し、`buildSDKHooks()` で SDK 形式に変換するブリッジパターンを採用しました。

```typescript
// 消費側が実装するシンプルなインターフェース
interface ArgusHooks {
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

```typescript
// ArgusHooks → SDK の HookCallbackMatcher[] に変換
function buildSDKHooks(
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
  // ...

  return sdkHooks;
}
```

これにより、消費側は SDK の内部構造を知る必要がなくなります。

```typescript
// slack-bot での使い方
const result = await query("メールを送って", {
  hooks: {
    onPreToolUse: async ({ toolName }) => {
      await postSlackMessage(`🔧 ${toolName} を実行中...`);
    },
    onPostToolUse: async ({ toolName, toolResult }) => {
      await recordToDatabase(toolName, toolResult);
    },
  },
});
```

### 観測 Hooks の共通実装

実際の DB 書き込みロジックは `createDBObservationHooks()` として共通化しています。`agent-core` パッケージは `db` パッケージに直接依存しない設計なので、DB インターフェースを外部注入するパターンを使っています。

```typescript
interface ObservationDB {
  db: {
    insert: (table: any) => {
      values: (v: any) => { returning: () => Promise<any[]> };
    };
    update: (table: any) => {
      set: (v: any) => { where: (cond: any) => Promise<any> };
    };
  };
  tasks: any;
  lessons: any;
  eq: (column: any, value: any) => any;
}

function createDBObservationHooks(
  obsDB: ObservationDB,
  dbSessionId: string,
  logPrefix = "[ObservationHooks]",
): ArgusHooks {
  const { db, tasks, lessons, eq } = obsDB;
  const taskIds = new Map<string, { dbId: string; startTime: number }>();

  return {
    onPreToolUse: async ({ toolUseId, toolName, toolInput }) => {
      try {
        const [task] = await db
          .insert(tasks)
          .values({
            sessionId: dbSessionId,
            toolName,
            toolInput: toolInput as Record<string, unknown>,
            status: "running",
          })
          .returning();
        taskIds.set(toolUseId, { dbId: task.id, startTime: Date.now() });
      } catch (err) {
        console.error(`${logPrefix} Failed to record PreToolUse`, err);
      }
    },

    onPostToolUse: async ({ toolUseId, toolResult }) => {
      try {
        const tracked = taskIds.get(toolUseId);
        if (tracked) {
          await db
            .update(tasks)
            .set({
              toolResult: (typeof toolResult === "string"
                ? { text: toolResult }
                : toolResult) as Record<string, unknown>,
              durationMs: Date.now() - tracked.startTime,
              status: "success",
            })
            .where(eq(tasks.id, tracked.dbId));
          taskIds.delete(toolUseId);
        }
      } catch (err) {
        console.error(`${logPrefix} Failed to record PostToolUse`, err);
      }
    },

    onToolFailure: async ({ toolUseId, toolName, toolInput, error }) => {
      try {
        const tracked = taskIds.get(toolUseId);
        if (tracked) {
          await db
            .update(tasks)
            .set({
              toolResult: { error } as Record<string, unknown>,
              durationMs: Date.now() - tracked.startTime,
              status: "error",
            })
            .where(eq(tasks.id, tracked.dbId));
          taskIds.delete(toolUseId);
        }
        // エピソード記憶: 失敗パターンを学習
        await db.insert(lessons).values({
          sessionId: dbSessionId,
          taskId: tracked?.dbId,
          toolName,
          errorPattern: error,
          reflection: `Tool ${toolName} failed with input: ${JSON.stringify(toolInput).slice(0, 500)}`,
          severity: "medium",
        });
      } catch (err) {
        console.error(`${logPrefix} Failed to record tool failure`, err);
      }
    },
  };
}
```

Hook の中で DB 書き込みが失敗しても、エージェントの実行は止まりません。観測が壊れてもコア実行に影響しない——これは本番運用で非常に重要なパターンです。

ツール実行の失敗は `lessons` テーブルにも記録されます。この **エピソード記憶** は次回のセッション開始時にプロンプトに注入され、同じ失敗を繰り返さないように学習します。

## Dual-Mode 実行: Max Plan / API Key の自動切替

開発と本番で異なる認証方式を自動判定する仕組みも、この移行で実現しました。

```typescript
const CLAUDE_CLI_PATHS = [
  join(homedir(), ".local", "bin", "claude"),
  "/usr/local/bin/claude",
  "/opt/homebrew/bin/claude",
];

function isMaxPlanAvailable(): boolean {
  // macOS で CLI がインストール済み = ローカル開発
  if (process.platform !== "darwin") return false;
  return CLAUDE_CLI_PATHS.some((p) => existsSync(p));
}

function getDefaultModel(): string {
  if (isMaxPlanAvailable()) return "claude-opus-4-6";
  // サーバーでは API キーでコスト効率の良い Sonnet を使用
  return process.env.ANTHROPIC_API_KEY
    ? "claude-sonnet-4-5-20250929"
    : "claude-opus-4-6";
}
```

Max Plan（Claude Desktop のサブスクリプション）が検出された場合、環境変数から `ANTHROPIC_API_KEY` を除外して、SDK にローカル接続を強制します。

```typescript
function envForSDK(): Record<string, string | undefined> | undefined {
  if (!isMaxPlanAvailable() || !process.env.ANTHROPIC_API_KEY) {
    return undefined; // デフォルト環境をそのまま使用
  }
  // Max Plan 検出: API キーを除外してローカル接続に
  const { ANTHROPIC_API_KEY: _, ...rest } = process.env;
  return rest;
}
```

これにより、以下が自動的に切り替わります。

| 環境                   | 認証方式            | モデル | コスト   |
| ---------------------- | ------------------- | ------ | -------- |
| macOS + Claude Desktop | Max Plan (ローカル) | Opus   | 無料     |
| Linux (Railway VPS)    | API Key             | Sonnet | 従量課金 |

設定ファイルの変更は一切不要です。

## テスト: `fakeStream` によるモック

SDK の AsyncGenerator をテストするために `fakeStream` ヘルパーを用意しています。

```typescript
/** SDKMessage の AsyncGenerator を作るヘルパー */
async function* fakeStream(
  messages: SDKMessage[],
): AsyncGenerator<SDKMessage, void> {
  for (const msg of messages) {
    yield msg;
  }
}

// テストでの使い方
it("should successfully execute query", async () => {
  vi.mocked(sdk.query).mockReturnValue(
    fakeStream([
      {
        type: "system",
        subtype: "init",
        session_id: "test-session-123",
        // ... その他のフィールド
      },
      {
        type: "result",
        subtype: "success",
        result: "Hello",
        total_cost_usd: 0.001,
        is_error: false,
        session_id: "test-session-123",
        // ...
      },
    ]) as unknown as sdk.Query,
  );

  const result = await query("Test prompt");

  expect(result.sessionId).toBe("test-session-123");
  expect(result.success).toBe(true);
  expect(result.message.content).toEqual([{ type: "text", text: "Hello" }]);
});
```

ポイントは `session_id` を全メッセージで一致させること。`result` メッセージが最後に `sessionId` を上書きするため、不一致があるとテストが不安定になります。

## ハマりポイント

### SDK の CLI が result 後に exit code 1 で終了する

SDK は内部で CLI プロセスを起動しますが、画像の Read など一部のツール実行後に exit code 1 で終了することがあります。`result` メッセージを受信済みなら正常結果として扱う `hasResult` ガードを入れています。

```typescript
try {
  for await (const msg of stream) {
    // ... メッセージ処理
  }
} catch (streamError) {
  if (hasResult) {
    // result を受信済みなら正常として扱う
    console.warn("[agent-core] Process exited after result (ignoring)");
  } else {
    throw streamError;
  }
}
```

### `which` コマンドが子プロセスで失敗する

Max Plan の検出で `which claude` を使おうとしましたが、子プロセスでは PATH が異なるため見つからないことがありました。`fs.existsSync()` で既知のパスを直接チェックする方式に変更して解決しました。

### `@anthropic-ai/claude-code` と `@anthropic-ai/claude-agent-sdk` は別物

`@anthropic-ai/claude-code` は CLI パッケージであり、プログラマティック API は提供していません。SDK として使えるのは `@anthropic-ai/claude-agent-sdk` のみです。npm パッケージ名が紛らわしいので注意してください。

## 移行の成果

SDK 移行による改善をまとめます。

| 観点           | Before (CLI spawn)        | After (Agent SDK)     |
| -------------- | ------------------------- | --------------------- |
| ストリーム処理 | 自前パーサー (壊れやすい) | 型付き AsyncGenerator |
| ツール観測     | ログ事後パース            | リアルタイム Hook     |
| セッション管理 | ファイル + フラグ         | `resume` オプション   |
| 型安全性       | なし                      | 完全型付き            |
| 消費側の変更   | —                         | ゼロ                  |

消費側（slack-bot, orchestrator, dashboard）のインポートや呼び出しコードは **一切変更なし** で移行が完了しました。これは `agent-core` パッケージが SDK の複雑さを完全に隠蔽しているからです。

## まとめ

Claude Agent SDK への移行で得た教訓をまとめます。

1. **ラッパーレイヤーを作る** — SDK に直接依存するのは 1 パッケージだけにし、消費側は安定したインターフェースを使う。SDK のバージョンアップは 1 箇所だけ直せば済む。

2. **Hook でビジネスロジックを注入する** — 観測、通知、DB 記録などは Hook 経由で外部から注入する。コア実行エンジンはこれらの関心事を知らない。

3. **環境自動判定で設定をなくす** — 開発と本番で設定ファイルを切り替えるのではなく、ランタイムで検出して自動切替する。

4. **AsyncGenerator のテストは `fakeStream` で** — SDK モックは yield するメッセージ配列を定義するだけ。シンプルで堅牢。

AI エージェントの本番運用は、「動くデモ」から「壊れにくいシステム」への転換が必要です。Claude Agent SDK は、その転換を技術的に支える良い基盤だと感じています。

---

この記事で紹介した Argus のコードは [GitHub](https://github.com/ryusuke-ai/argus) で公開しています。質問やフィードバックがあれば、コメントや Issue でお気軽にどうぞ。
