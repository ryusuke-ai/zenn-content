---
title: "11パッケージの pnpm monorepo を Turborepo なしで運用する"
emoji: "📦"
type: "tech"
topics: ["pnpm", "monorepo", "TypeScript", "AgentSDK"]
published: true
---

## はじめに

AI エージェントシステム「Argus」を個人開発する中で、コードベースが 11 パッケージの monorepo に成長しました。Turborepo のような専用ツールは導入せず、**pnpm のネイティブ機能だけ**で依存解決・ビルド・テストを回しています。

本記事では、実際のプロジェクト構成をベースに、pnpm monorepo の設計判断とつまずきポイントを共有します。

## プロジェクト構成

```
argus/
├── apps/
│   ├── slack-bot/              # Slack 連携（Socket Mode）
│   ├── dashboard/              # Next.js 16 監視UI
│   └── agent-orchestrator/     # Cron スケジューラ + REST API
├── packages/
│   ├── agent-core/             # Claude Agent SDK ラッパー
│   ├── db/                     # Drizzle ORM スキーマ（16 テーブル）
│   ├── knowledge/              # Knowledge MCP Server
│   ├── knowledge-personal/     # Personal Knowledge MCP Server
│   ├── gmail/                  # Google API（OAuth2 + Gmail）
│   ├── google-calendar/        # Google Calendar MCP Server
│   ├── tiktok/                 # TikTok Content Posting API
│   ├── r2-storage/             # Cloudflare R2 クライアント
│   └── slack-canvas/           # Slack Canvas API
├── pnpm-workspace.yaml
├── package.json
└── tsconfig.json
```

`pnpm-workspace.yaml` は至ってシンプルです。

```yaml
packages:
  - "packages/*"
  - "apps/*"
```

## なぜ Turborepo を入れなかったか

理由は 3 つあります。

**1. pnpm -r が依存順を自動解決する**

`pnpm -r build` は内部的にワークスペースの依存グラフをトポロジカルソートし、正しい順序でビルドします。11 パッケージ程度なら、これだけで十分でした。

**2. キャッシュの恩恵が薄い**

Turborepo の最大の売りはリモートキャッシュですが、個人開発で CI もローカル実行がメインの環境では、キャッシュヒット率が低く恩恵を感じにくいです。

**3. 設定の複雑さを避けたい**

`turbo.json` にパイプラインを定義し、各タスクの入出力を宣言する必要があります。11 パッケージの設定を正確に書いて維持するコストに対して、リターンが見合いませんでした。

## 依存グラフの設計

パッケージ間の依存関係はレイヤー構造になっています。

```
Layer 0（基盤）:  agent-core, db, r2-storage
                   └─ 内部依存なし。外部ライブラリのみに依存

Layer 1（統合）:  knowledge, knowledge-personal, gmail,
                   slack-canvas, tiktok
                   └─ db に依存

Layer 2（派生）:  google-calendar
                   └─ gmail の OAuth2 インフラを再利用

Layer 3（Apps）:  slack-bot, dashboard, agent-orchestrator
                   └─ 複数パッケージに依存
```

この「上位レイヤーは下位にのみ依存する」ルールを守ることで、循環依存を防いでいます。

たとえば `slack-bot` は最も依存が多く、7 つの内部パッケージを使います。

```json
{
  "dependencies": {
    "@argus/agent-core": "workspace:*",
    "@argus/db": "workspace:*",
    "@argus/gmail": "workspace:*",
    "@argus/google-calendar": "workspace:*",
    "@argus/r2-storage": "workspace:*",
    "@argus/slack-canvas": "workspace:*",
    "@argus/tiktok": "workspace:*"
  }
}
```

`workspace:*` プロトコルにより、ローカルではシンボリックリンクで即座に反映され、`pnpm publish` 時には実バージョンに自動変換されます。

## TypeScript Project References の活用

ルートの `tsconfig.json` は **ビルドハブ** として機能させています。

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "files": [],
  "references": [
    { "path": "./packages/agent-core" },
    { "path": "./packages/db" },
    { "path": "./packages/knowledge" }
    // ... 全 11 パッケージ
  ]
}
```

`"files": []` でルート自体はコンパイルせず、`references` で全子パッケージを指定します。各パッケージは `composite: true` + `extends` で設定を継承します。

```json
// packages/agent-core/tsconfig.json
{
  "extends": "../../tsconfig.json",
  "compilerOptions": {
    "composite": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

こうすることで `tsc --build` でインクリメンタルビルドが効き、変更のないパッケージは再コンパイルされません。

## ESM 統一の決断

全 11 パッケージで `"type": "module"` を採用しています。CJS は一切使いません。

```json
{
  "name": "@argus/agent-core",
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "default": "./dist/index.js"
    }
  }
}
```

ESM 統一のメリットは **設定のシンプルさ** です。CJS/ESM の dual package を考慮する必要がなく、`exports` フィールドも `types` と `default` の 2 行で済みます。

ただし、ESM では `node:` プレフィックスやファイル拡張子の明示が必要です。

```typescript
// ESM では node: プレフィックス必須
import { readFile } from "node:fs/promises";
import { join } from "node:path";

// パッケージ内インポートは .js 拡張子
import { createSession } from "./session.js";
```

ソースは `.ts` で書いているのにインポートは `.js` — これが ESM + TypeScript の最大のつまずきポイントでした。`moduleResolution: "bundler"` なら拡張子なしでも通りますが、`tsc` 単体でビルドする場合は `.js` が必要です。

## サブパスエクスポートで API を整理する

`@argus/db` パッケージでは、用途別にエントリポイントを分けています。

```json
{
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "default": "./dist/index.js"
    },
    "./schema": {
      "types": "./dist/schema.d.ts",
      "default": "./dist/schema.js"
    },
    "./client": {
      "types": "./dist/client.d.ts",
      "default": "./dist/client.js"
    }
  }
}
```

消費側はこう書けます。

```typescript
import { sessions, messages } from "@argus/db/schema";
import { getClient } from "@argus/db/client";
```

`import { everything } from "@argus/db"` ではなく、サブパスで必要なものだけを公開することで、パッケージの責務が明確になります。

## dev スクリプトの工夫

開発時に一番ハマったのは **ビルド順序** です。

```json
{
  "scripts": {
    "dev": "pnpm build && pnpm --parallel --filter './apps/*' dev"
  }
}
```

`pnpm build` で全パッケージをトポロジカル順にビルドし、完了後に `--parallel` で 3 つのアプリを同時起動します。

なぜ `--parallel` だけではダメかというと、アプリが `workspace:*` で参照するパッケージの `dist/` が存在しない状態で起動すると、import エラーになるからです。

個別起動時も同様です。

```json
{
  "dev:slack": "pnpm --filter @argus/slack-bot dev"
}
```

`slack-bot` の `dev` スクリプト内で、依存パッケージのビルドを先行実行するようにしています。

## テスト戦略

テストは Vitest で統一し、**各パッケージが独立して実行** する方式です。

```json
// ルート package.json
{ "scripts": { "test": "pnpm -r test" } }

// 各パッケージ
{ "scripts": { "test": "vitest run" } }
```

`vitest.workspace.ts`（ワークスペースモード）は使っていません。各パッケージが独立して `vitest run` を呼ぶだけです。

dashboard だけは `jsdom` 環境が必要なので、専用の `vitest.config.ts` を持っています。

```typescript
// apps/dashboard/vitest.config.ts
export default defineConfig({
  test: {
    environment: "jsdom",
    globals: true,
    setupFiles: ["./src/test-setup.ts"],
  },
});
```

テストファイルはソースと同ディレクトリに置く **コロケーション方式** を採用しています。

```
packages/agent-core/src/
├── agent.ts
├── agent.test.ts      # ← 隣に置く
├── hooks.ts
└── hooks.test.ts
```

この方式のメリットは「テストが書かれているか一目でわかる」ことです。`__tests__/` ディレクトリに分離すると、対応するソースファイルを探す手間が増えます。

現在 11 パッケージで **1,360+ テスト** が通っています。

## まとめ

| 判断                          | 理由                               |
| ----------------------------- | ---------------------------------- |
| Turborepo なし                | pnpm -r のトポロジカルソートで十分 |
| レイヤー構造                  | 循環依存を構造的に防ぐ             |
| TypeScript Project References | インクリメンタルビルドで高速化     |
| ESM 統一                      | dual package の複雑さを排除        |
| サブパスエクスポート          | パッケージ API の責務明確化        |
| テストのコロケーション        | 未テストファイルの可視化           |

11 パッケージ程度の規模なら、pnpm のネイティブ機能だけで十分に運用できます。Turborepo や Nx を入れるかどうかは「キャッシュの恩恵が実感できるか」で判断すればよいと思います。
