---
title: "MCPサーバーでAIエージェントにナレッジベースを持たせる方法"
emoji: "🧠"
type: "tech"
topics: ["mcp", "claude", "ai", "typescript", "postgresql"]
published: true
---

## はじめに

AIエージェントに「知識」を持たせたいとき、あなたならどうしますか？

RAG（Retrieval-Augmented Generation）でベクトル検索する方法が一般的ですが、もう一つのアプローチがあります。**MCP（Model Context Protocol）** を使って、AIエージェントのツールとしてナレッジベースを統合する方法です。

この記事では、私が開発しているマルチエージェントシステム「Argus」で実際に採用した MCP ベースのナレッジ管理アーキテクチャを紹介します。エージェントが `knowledge_search` や `personal_context` といったツールを、Bash や Read と同じ感覚で使えるようにした設計と、その実装を解説します。

## MCP とは何か

**MCP（Model Context Protocol）** は、Anthropic が策定したオープンプロトコルで、AIモデルと外部ツール・データソースを標準的なインターフェースで接続する仕組みです。

MCP では、ツールを提供する側を **MCP サーバー**、それを利用する側を **MCP クライアント**（ここでは Claude Agent SDK）と呼びます。通信は JSON-RPC over stdio で行われ、MCP サーバーは子プロセスとして起動されます。

```
┌─────────────────────────────────┐
│   Claude Agent SDK (Client)     │
│                                 │
│  ┌────────┐  ┌───────────┐     │
│  │ Bash   │  │ Read/Write│     │  ← ネイティブツール
│  └────────┘  └───────────┘     │
│                                 │
│  ┌────────────────────────┐     │
│  │ MCP Server (stdio)     │     │  ← MCP ツール
│  │ - knowledge_search     │     │
│  │ - knowledge_add        │     │
│  │ - personal_context     │     │
│  └────────────────────────┘     │
└─────────────────────────────────┘
```

ポイントは、MCP ツールがネイティブツールとまったく同じ見え方でエージェントに提供されることです。エージェントは「これは MCP 経由のツールだ」と意識する必要がなく、必要に応じて自然に呼び出します。

## なぜ REST API でなく MCP を選んだか

ナレッジベースへのアクセス方法として、いくつかの選択肢を検討しました。

### 選択肢A: REST API

```
Agent → Bash("curl http://localhost:3950/api/knowledge?q=...") → JSON
```

標準的な HTTP パターンですが、**エージェントが curl コマンドを構築する必要がある**という問題があります。Claude Code エージェントは「ツール」で思考します。HTTP クライアントではありません。curl の引数を組み立てさせるのは不自然な間接化で、エラーの温床になります。

### 選択肢B: システムプロンプトに知識を注入

```
System Prompt: "以下はナレッジベースの全内容です: ..."
```

レイテンシゼロでアクセスできますが、**コンテキストウィンドウを汚染**します。ナレッジが増えるほどトークン消費が増え、O(n) でスケールしません。「ID で参照して必要時に取得する」設計原則に反します。

### 選択肢C: MCP サーバー（採用）

```
Agent → knowledge_search(query: "デプロイ手順") → 結果
```

エージェントにとって最も自然な形です。ツールとして呼び出すだけで、HTTP の知識もコンテキストの消費も不要。必要なときに必要な情報だけを取得できます。

## Knowledge MCP サーバーの設計

Argus のナレッジシステムは、2つの MCP サーバーで構成されています。

| MCP サーバー         | 対象                 | ツール数           |
| -------------------- | -------------------- | ------------------ |
| `knowledge`          | 組織の共有ナレッジ   | 2〜5（ロール依存） |
| `knowledge-personal` | 個人のメモ・性格情報 | 6                  |

### ロールベースのツール公開（Collector / Executor）

最も重要な設計判断は、**エージェントのロールに応じて公開するツールを制限する**ことです。

```
Collector エージェント（情報収集担当）:
  knowledge_search, knowledge_list, knowledge_add, knowledge_update, knowledge_archive

Executor エージェント（タスク実行担当）:
  knowledge_search, knowledge_list
```

Executor は検索と一覧取得しかできません。これにより、タスク実行中のエージェントがナレッジベースを誤って変更するリスクをゼロにしています。**最小権限の原則を AI エージェントに適用した**形です。

### サーバーの実装

Knowledge MCP サーバーのコアは約130行です。

```typescript
// packages/knowledge/src/server.ts
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  ListToolsRequestSchema,
  CallToolRequestSchema,
  type Tool,
} from "@modelcontextprotocol/sdk/types.js";

export class KnowledgeMcpServer {
  private server: Server;
  private tools: Tool[];

  constructor(
    private service: KnowledgeService,
    private role: KnowledgeRole,
  ) {
    // ロールに応じてツールを初期化
    this.tools = this.initializeTools();
    this.server = new Server(
      { name: "knowledge-server", version: "0.1.0" },
      { capabilities: { tools: {} } },
    );
    this.setupHandlers();
  }

  private initializeTools(): Tool[] {
    const commonTools = getCommonTools(); // search, list
    if (this.role === "collector") {
      return [...commonTools, ...getCollectorTools()]; // + add, update, archive
    }
    return commonTools; // executor は search, list のみ
  }

  private setupHandlers(): void {
    // ツール一覧を返す
    this.server.setRequestHandler(ListToolsRequestSchema, async () => ({
      tools: this.tools,
    }));

    // ツール呼び出しをルーティング
    this.server.setRequestHandler(CallToolRequestSchema, async (request) => {
      const { name, arguments: args } = request.params;
      const result = await this.handleToolCall(name, args ?? {});
      return {
        content: [{ type: "text", text: JSON.stringify(result, null, 2) }],
      };
    });
  }

  async start(): Promise<void> {
    const transport = new StdioServerTransport();
    await this.server.connect(transport);
  }
}
```

`@modelcontextprotocol/sdk` の `Server` クラスに、`ListToolsRequestSchema`（ツール一覧）と `CallToolRequestSchema`（ツール呼び出し）の2つのハンドラーを登録するだけです。

### ツール定義

各ツールは JSON Schema で入出力を定義します。

```typescript
// packages/knowledge/src/tools/common-tools.ts
export function getCommonTools(): Tool[] {
  return [
    {
      name: "knowledge_search",
      description:
        "Search knowledge entries by name or content. Returns matching entries.",
      inputSchema: {
        type: "object",
        properties: {
          query: {
            type: "string",
            description: "Search query to match against name or content",
          },
        },
        required: ["query"],
      },
    },
    {
      name: "knowledge_list",
      description: "List all knowledge entries ordered by last updated date.",
      inputSchema: {
        type: "object",
        properties: {},
      },
    },
  ];
}
```

`description` がエージェントにとっての「マニュアル」になります。エージェントはこの説明を読んで、いつ・どのツールを使うべきか判断します。

### 二重の権限チェック

MCP サーバーがツールの可視性を制御する一方で、サービス層でも権限チェックを行います。

```typescript
// packages/knowledge/src/service.ts
export class KnowledgeServiceImpl implements KnowledgeService {
  constructor(private role: KnowledgeRole) {}

  private requireCollector(): void {
    if (this.role !== "collector") {
      throw new PermissionError("write", "collector");
    }
  }

  async add(name: string, content: string, description?: string) {
    this.requireCollector(); // MCP層を迂回されても防御
    const [newKnowledge] = await db
      .insert(knowledges)
      .values({ name, content, description })
      .returning();
    return newKnowledge;
  }

  async search(query: string) {
    return db
      .select()
      .from(knowledges)
      .where(
        or(
          ilike(knowledges.name, `%${query}%`),
          ilike(knowledges.content, `%${query}%`),
        ),
      );
  }
}
```

防御の深さ（Defense in Depth）の考え方です。MCP サーバーが Executor にツールを見せなくても、万が一サービス層が直接呼ばれた場合に備えて `requireCollector()` で二重に防御します。

### エントリーポイント

MCP サーバーの起動は環境変数でロールを受け取る CLI です。

```typescript
// packages/knowledge/src/cli.ts
const role = process.env.KNOWLEDGE_ROLE as KnowledgeRole;
const service = new KnowledgeServiceImpl(role);
const server = new KnowledgeMcpServer(service, role);
await server.start();
```

このシンプルさが MCP の美点です。stdio トランスポートで通信するため、HTTP サーバーのセットアップもポート管理も不要です。

## Personal Knowledge の DB 移行

Argus には組織の共有ナレッジとは別に、「個人ナレッジ」の仕組みがあります。ユーザーの性格・価値観・習慣・キャリア目標などを保持し、エージェントの応答をパーソナライズするためのものです。

### ファイルベースからの脱却

初期実装ではローカルの Markdown ファイルで管理していました。

```
data/
├── personality/
│   └── value.md          # 価値観、強み、思考スタイル
├── areas/
│   └── habits/
│       ├── index.md      # 習慣の概要
│       └── value.md      # 習慣の詳細
└── career/
    └── goals.md          # キャリア目標
```

しかし、この方式にはいくつかの問題がありました。

1. **デプロイ時にデータが消える** — Docker コンテナを再起動するとファイルが失われる
2. **複数プロセスからの同時アクセス** — Slack Bot、Orchestrator、Dashboard が同じファイルを読み書きする
3. **アーキテクチャの不一致** — 他のデータはすべて PostgreSQL なのに、ここだけファイルベース

### PostgreSQL への移行

`personal_notes` テーブルを設計し、ファイルパスの構造をそのまま保持しました。

```typescript
// packages/db/src/schema.ts
export const personalNotes = pgTable("personal_notes", {
  id: uuid("id").primaryKey().defaultRandom(),
  path: varchar("path", { length: 500 }).notNull().unique(), // "personality/value.md"
  category: varchar("category", { length: 255 }).notNull(), // "personality"
  name: varchar("name", { length: 255 }).notNull(), // "value"
  content: text("content").notNull(),
  updatedAt: timestamp("updated_at").defaultNow().notNull(),
});
```

`path` をユニークキーにすることで、元のファイルパスでそのまま検索できます。`category` はフィルタリング用に非正規化しています。

### データ移行ツール

ファイルから DB への移行は、冪等な seed スクリプトで行います。

```typescript
// packages/knowledge-personal/src/seed.ts
for (const filePath of files) {
  const relPath = relative(dataDir, filePath).split(sep).join("/");
  const category = relPath.split("/")[0] ?? "uncategorized";
  const name = parse(relPath).name;
  const content = await readFile(filePath, "utf-8");

  await db
    .insert(personalNotes)
    .values({ path: relPath, category, name, content, updatedAt: new Date() })
    .onConflictDoUpdate({
      target: personalNotes.path,
      set: {
        content: sql`excluded.content`,
        category: sql`excluded.category`,
        updatedAt: sql`excluded.updated_at`,
      },
    });
}
```

`ON CONFLICT DO UPDATE`（upsert）パターンにより、何度実行しても安全です。ファイルの追加や変更があっても、このスクリプトを再実行するだけで DB が最新になります。

### MCP サーバー層は変更ゼロ

最も重要なポイントは、**MCP サーバーのインターフェースを一切変更せずに移行できた**ことです。`PersonalMcpServer` は `PersonalService` インターフェースに依存しており、その実装が `fs.readFile` から `db.select()` に変わっただけです。

```typescript
// インターフェースは同じ
export interface PersonalService {
  list(
    category?: string,
  ): Promise<{ path: string; name: string; category: string }[]>;
  read(path: string): Promise<NoteEntry>;
  search(query: string): Promise<SearchResult[]>;
  getPersonalityContext(section?: PersonalitySection): Promise<string>;
  add(category: string, name: string, content: string): Promise<NoteEntry>;
  update(
    path: string,
    content: string,
    mode: "append" | "replace",
  ): Promise<NoteEntry>;
}
```

MCP がサービス層とプロトコル層を分離していたおかげで、ストレージの移行がクリーンに行えました。

## Gmail / Google Calendar MCP の実装

ナレッジだけでなく、外部サービスとの統合にも同じ MCP パターンを適用しています。

### Google Calendar MCP サーバー

```typescript
// packages/google-calendar/src/server.ts
export class CalendarMcpServer {
  constructor() {
    this.tools = getCalendarTools();
    this.server = new Server(
      { name: "google-calendar-server", version: "0.1.0" },
      { capabilities: { tools: {} } },
    );
    this.setupHandlers();
  }

  async handleToolCall(name: string, args: Record<string, unknown>) {
    switch (name) {
      case "create_event":
        return calendarClient.createEvent(args);
      case "list_events":
        return calendarClient.listEvents(args);
      case "update_event":
        return calendarClient.updateEvent(args);
      case "delete_event":
        await calendarClient.deleteEvent(args.eventId as string);
        return { success: true };
      default:
        throw new Error(`Unknown tool: ${name}`);
    }
  }
}
```

Knowledge サーバーとまったく同じ構造です。`Server` + `StdioServerTransport` + ハンドラー登録という3つのステップは、すべての MCP サーバーで共通です。

### Claude Agent SDK への登録

MCP サーバーを Claude Agent SDK に登録するのは、設定オブジェクトに追加するだけです。

```typescript
// apps/slack-bot/src/session-manager.ts
const SLACK_SDK_OPTIONS = {
  mcpServers: {
    "google-calendar": {
      command: "node",
      args: ["packages/google-calendar/dist/cli.js"],
      env: {
        GMAIL_CLIENT_ID: process.env.GMAIL_CLIENT_ID || "",
        GMAIL_CLIENT_SECRET: process.env.GMAIL_CLIENT_SECRET || "",
        DATABASE_URL: process.env.DATABASE_URL || "",
      },
    },
    "knowledge-personal": {
      command: "node",
      args: ["packages/knowledge-personal/dist/cli.js"],
      env: {
        DATABASE_URL: process.env.DATABASE_URL || "",
      },
    },
    gmail: {
      command: "node",
      args: ["packages/gmail/dist/mcp-cli.js"],
      env: {
        /* ... */
      },
    },
  },
};
```

`command` + `args` で子プロセスとして起動され、`env` で環境変数が分離されます。エージェントの `query()` が呼ばれるたびに MCP サーバーが起動し、セッション終了とともに終了します。

### MCP サーバーの一貫したパターン

Argus で稼働している MCP サーバーをまとめると、こうなります。

| MCP サーバー         | ツール                                                                                         | 用途                |
| -------------------- | ---------------------------------------------------------------------------------------------- | ------------------- |
| `knowledge`          | search, list, add, update, archive                                                             | 共有ナレッジの CRUD |
| `knowledge-personal` | personal_search, personal_read, personal_list, personal_context, personal_add, personal_update | 個人メモと性格情報  |
| `gmail`              | send_email                                                                                     | メール送信          |
| `google-calendar`    | create_event, list_events, update_event, delete_event                                          | カレンダー管理      |
| `playwright`         | browser\_\*                                                                                    | ブラウザ自動化      |

5つのサーバーがすべて同じパターンで実装されています。新しい外部サービスを統合するときも、このテンプレートに沿うだけです。

## MCP サーバーのテスト方法

MCP サーバーはプロセス間通信を行うため、テストには工夫が必要です。Argus では、サービス層と MCP 層を分離してテストしています。

### サービス層のユニットテスト

サービス層は純粋なデータアクセスロジックなので、DB のモックだけでテストできます。

```typescript
// packages/knowledge/src/service.test.ts
describe("KnowledgeServiceImpl", () => {
  it("executor ロールで add を呼ぶと PermissionError", async () => {
    const service = new KnowledgeServiceImpl("executor");
    await expect(service.add("test", "content")).rejects.toThrow(
      PermissionError,
    );
  });

  it("collector ロールで add できる", async () => {
    const service = new KnowledgeServiceImpl("collector");
    const result = await service.add("test", "content");
    expect(result.name).toBe("test");
  });
});
```

### MCP ハンドラーのテスト

MCP 層は `handleToolCall` メソッドを直接テストします。stdio トランスポートを介さずに、ツール呼び出しのルーティングを検証できます。

```typescript
// packages/knowledge/src/server.test.ts
describe("KnowledgeMcpServer", () => {
  it("executor はツール2個のみ", () => {
    const server = new KnowledgeMcpServer(mockService, "executor");
    expect(server.getTools()).toHaveLength(2);
    expect(server.getTools().map((t) => t.name)).toEqual([
      "knowledge_search",
      "knowledge_list",
    ]);
  });

  it("collector はツール5個", () => {
    const server = new KnowledgeMcpServer(mockService, "collector");
    expect(server.getTools()).toHaveLength(5);
  });

  it("knowledge_search が正しくルーティングされる", async () => {
    const server = new KnowledgeMcpServer(mockService, "executor");
    await server.handleToolCall("knowledge_search", { query: "test" });
    expect(mockService.search).toHaveBeenCalledWith("test");
  });
});
```

`getTools()` と `handleToolCall()` を public にしておくことで、MCP プロトコルの詳細に触れることなくロジックをテストできます。

## 設計の振り返り: 良かった点と注意点

### 良かった点

**1. ネイティブなツール統合**

エージェントにとって、ナレッジ検索もカレンダー操作も「ただのツール」です。プロンプトで「curl で API を叩いてください」と指示する必要がありません。

**2. 権限分離がシンプル**

ロールに応じてツールの可視性を変えるだけで、最小権限を実現できます。JWT やAPIキーの管理は不要です。

**3. プロセス隔離**

各 MCP サーバーは独立した Node.js プロセスとして動作します。ナレッジサーバーでメモリリークが起きても、メインのエージェントプロセスには影響しません。

**4. 一貫したパターン**

5つの MCP サーバーがすべて同じ構造なので、新しいサーバーの追加が容易です。学習コストも低く抑えられます。

### 注意点

**1. 起動コスト**

各 `query()` 呼び出しのたびに複数の MCP サーバーが子プロセスとして起動されます。短い対話では起動コストが相対的に大きくなります。

**2. 設定の重複**

MCP サーバーの設定（パス、環境変数）を `session-manager.ts` と `executor.ts` の両方で持つ必要があります。変更時は両方を同期する必要があり、ここは改善の余地があります。

**3. デバッグの複雑さ**

ツール呼び出しが失敗したとき、MCP プロトコル層、サービス層、データベース層のどこで起きたのかを特定する必要があります。ログの構造化が重要です。

## まとめ

MCP サーバーを使ったナレッジベース統合は、AI エージェントにとって最も自然な「知識アクセスの形」を提供します。

要点をまとめると:

1. **MCP は AI エージェントのツール拡張プロトコル** — stdio ベースの子プロセスで、ネイティブツールと同等の体験を提供する
2. **ロールベースのツール公開** — `initializeTools()` でロールに応じてツールを出し分け、最小権限を実現する
3. **二重の権限チェック** — MCP 層でツールを隠し、サービス層で再検証する Defense in Depth
4. **一貫したサーバーパターン** — `Server` + `StdioServerTransport` + ハンドラーの3点セットをテンプレート化する
5. **インターフェース分離** — サービス層を抽象化しておけば、ストレージの移行も MCP 層に影響しない

MCP はまだ新しいプロトコルですが、AI エージェントシステムにおけるツール統合の標準になる可能性を感じています。「エージェントにとって自然なインターフェース」を考えたとき、MCP はとても良い選択肢です。

---

記事中のコード例は [Argus](https://github.com/ryusuke-ai/argus) プロジェクトの実装を簡略化したものです。
