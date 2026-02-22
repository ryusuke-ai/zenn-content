---
title: "Claude Code を10倍使いこなす実践Tips 7選【2026年版】"
emoji: "🛠"
type: "tech"
topics: ["ClaudeCode", "AI", "開発効率化", "プロンプトエンジニアリング"]
published: true
---

## はじめに

Claude Code を日常的に使っていると「もっと効率よくできないか」と感じる場面が増えてきます。本記事では、11 パッケージの monorepo を Claude Code で開発・運用する中で効果を実感した実践 Tips を 7 つ紹介します。

## 1. CLAUDE.md を階層化する

ルートの `CLAUDE.md` だけでなく、サブディレクトリにも配置できます。

```
project/
├── CLAUDE.md              # プロジェクト全体のルール
├── apps/frontend/
│   └── CLAUDE.md          # フロントエンド固有の規約
└── packages/core/
    └── CLAUDE.md          # コアパッケージの設計方針
```

サブディレクトリの CLAUDE.md は、そのディレクトリで作業する時だけルートに**追記**されます。文脈に応じた指示が可能になり、不要な情報がコンテキストを圧迫しません。

ただし、CLAUDE.md が肥大化すると逆効果です。私のプロジェクトでは **ルート CLAUDE.md を 75 行に抑え**、詳細は `.claude/rules/` に分離しています。

```markdown
## Conventions

詳細: @.claude/rules/coding-conventions.md
```

`@` 記法で参照すると、Claude Code は必要な時だけそのファイルを読みに行きます。「段階的開示（Progressive Disclosure）」の考え方です。

## 2. 思考レベルを使い分ける

プロンプトに特定キーワードを含めると、Claude の思考の深さが変わります。

| キーワード   | トークン上限 | 用途               |
| ------------ | ------------ | ------------------ |
| `think`      | 約 10,000    | 通常の設計判断     |
| `think hard` | 約 20,000    | 複雑なバグ調査     |
| `ultrathink` | 約 32,000    | アーキテクチャ設計 |

```bash
# 例: 複雑なリファクタリング
$ claude "ultrathink このモジュールの依存関係を整理して"
```

体感として、`ultrathink` を使うと「なぜその設計にするのか」の説明が格段に丁寧になります。逆に単純なタスクで使うとトークンの無駄遣いになるので、使い分けが重要です。

## 3. サブエージェントでコンテキストを分離する

メインの会話にすべてを詰め込むとコンテキストウィンドウが圧迫されます。調査やコード探索はサブエージェントに委譲しましょう。

`.claude/agents/` にカスタムエージェントを定義できます。

```markdown
# .claude/agents/code-reviewer.md

---

name: code-reviewer
description: コード品質レビュー専用
tools: [Read, Glob, Grep]
model: sonnet

---

以下の優先順でレビューしてください:

1. 正確性
2. セキュリティ
3. パフォーマンス
4. 保守性
```

サブエージェントは独立したコンテキストで動くため、メインスレッドがクリーンに保たれます。

私のプロジェクトでは 4 つのカスタムエージェントを定義しています。

| エージェント  | 役割                       | 使えるツール                  |
| ------------- | -------------------------- | ----------------------------- |
| analyzer      | 週次ログ分析               | Read, Glob, Grep, Bash        |
| code-reviewer | コード品質レビュー         | Read, Glob, Grep              |
| collector     | 情報収集（書き込み可）     | Read, Glob, Grep, Bash, Write |
| executor      | タスク実行（読み取りのみ） | Read, Glob, Grep, Bash        |

注目すべきは **collector と executor の権限分離** です。collector は Knowledge ベースに書き込めますが、executor は検索しかできません。最小権限の原則をエージェントレベルで適用しています。

## 4. settings.json で危険な操作をブロックする

`.claude/settings.json` で deny-first の権限設定ができます。

```json
{
  "permissions": {
    "deny": [
      "Read(.env)",
      "Read(.env.*)",
      "Read(.secrets/**)",
      "Bash(rm -rf *)",
      "Bash(git push --force*)",
      "Bash(git reset --hard*)",
      "Bash(DROP TABLE*)",
      "Write(.env)",
      "Write(.env.*)"
    ],
    "allow": [
      "Read(packages/**)",
      "Read(apps/**)",
      "Write(.claude/agent-output/**)",
      "Write(.claude/agent-workspace/**)",
      "Bash(node .claude/skills/*/scripts/*)",
      "WebSearch(*)"
    ]
  }
}
```

ポイントは **deny を先に書く** ことです。

- `.env` は絶対に読ませない（API キーが入っている）
- 破壊的な git コマンドを禁止
- 書き込みは `agent-output/` と `agent-workspace/` のみ許可
- スキルのスクリプト実行は明示的に allow

これにより、エージェントが自律的に動いても「やってはいけないこと」が構造的に防がれます。

## 5. スキルでワークフローを再利用可能にする

繰り返し行う作業は `.claude/skills/` にスキルとして定義できます。スキルは `/skill-name` で呼び出せるショートカットです。

私のプロジェクトでは 32 のスキルを定義しています。その中でも効果が大きいのは**統合スキル**のパターンです。

```
.claude/skills/
├── sns-publisher/        # 統合エントリポイント
│   └── SKILL.md          # キーワードで個別スキルに分岐
├── sns-x-poster/         # X 投稿
├── sns-threads-poster/   # Threads 投稿
├── sns-youtube-creator/  # YouTube 投稿
└── ...                   # 11 プラットフォーム
```

`sns-publisher` が**ルーター**として機能し、「X に投稿」と言えば `sns-x-poster` に、「YouTube 動画」と言えば `sns-youtube-creator` に自動分岐します。

スキルのディレクトリ構成にもパターンがあります。

```
skills/{skill-name}/
├── SKILL.md          # エントリポイント（手順の概要のみ）
├── phases/           # フェーズ別の詳細手順
├── prompts/          # サブエージェント用プロンプト
├── schemas/          # Zod スキーマ（中間成果物の型定義）
└── scripts/          # 実行スクリプト（Node.js）
```

SKILL.md 本体は軽くして、詳細を `phases/` や `prompts/` に分離する — ここでも段階的開示の考え方が効きます。

## 6. 中間成果物の JSON でフェーズ間を繋ぐ

複数フェーズにまたがるタスク（例: 動画制作）では、フェーズ間を **型付き JSON** で連携します。

```
Phase 1: シナリオ生成 → scenario.json
Phase 2: 対話生成   → dialogue.json（scenario.json を入力）
Phase 3: 演出指示   → direction.json（dialogue.json を入力）
Phase 4: レンダリング → output.mp4（direction.json を入力）
```

各フェーズは独立して再実行可能です。Phase 3 の演出が気に入らなければ、Phase 1-2 はそのままに Phase 3 だけやり直せます。

中間成果物には Zod スキーマでバリデーションをかけています。

```typescript
const ScenarioSchema = z.object({
  title: z.string(),
  sections: z.array(
    z.object({
      heading: z.string(),
      points: z.array(z.string()),
    }),
  ),
});
```

こうすることで「Phase 1 の出力が壊れていて Phase 2 が失敗」というカスケード障害を早期検出できます。

出力先は日付プレフィックスで管理します。

```
.claude/agent-output/20260210-news-video/
├── scenario.json
├── dialogue.json
├── direction.json
└── output.mp4
```

## 7. サブエージェント委譲の 5 要素を守る

サブエージェントにタスクを委譲する際、曖昧な指示だと高確率で期待と違う結果が返ります。私のプロジェクトでは**5 つの要素を必ず指定する**ルールを設けています。

1. **作業ディレクトリの絶対パス** — サブエージェントは親の cwd を引き継がない場合がある
2. **入力ファイルのパス** — 何を読むべきかを明示
3. **出力ファイルのパス** — どこに書くべきかを明示
4. **参照すべきドキュメント** — 必要なコンテキストを指定
5. **完了条件** — 「何をもって完了とするか」を明文化

悪い例:

```
「テストを書いて」
```

良い例:

```
作業ディレクトリ: /path/to/project/packages/agent-core
入力: src/hooks.ts
出力: src/hooks.test.ts
参照: ../../.claude/rules/coding-conventions.md
完了条件: vitest run が全てパスし、カバレッジ 80% 以上
```

サブエージェントは「コンテキストを持たない他人」だと思って指示するのがコツです。

## まとめ

| Tips                     | 効果                       |
| ------------------------ | -------------------------- |
| CLAUDE.md 階層化 + @参照 | コンテキスト節約           |
| 思考レベルの使い分け     | タスクに応じた精度調整     |
| カスタムエージェント     | 役割ごとの権限分離         |
| deny-first 権限設定      | 安全な自律実行             |
| スキルの統合パターン     | ワークフローの再利用       |
| 中間成果物 JSON          | フェーズの独立再実行       |
| 委譲の 5 要素            | サブエージェントの精度向上 |

Claude Code は「対話型ツール」から「開発パートナー」に進化しています。設定ファイルとスキルを整備することで、プロジェクト固有のワークフローを Claude Code に教え込むことができます。
