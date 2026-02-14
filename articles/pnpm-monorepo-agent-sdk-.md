---
title: "pnpm monorepo で始める Agent SDK 開発"
emoji: "🔧"
type: "tech"
topics: ["pnpm", "monorepo", "AgentSDK"]
published: true
---
## pnpm monorepo とは

pnpm の workspace 機能を使って複数パッケージを管理する手法です。

## Agent SDK との連携

Agent SDK を monorepo の1パッケージとして配置することで、再利用性が高まります。

## セットアップ手順

1. pnpm init
2. pnpm-workspace.yaml を作成
3. packages/ に SDK ラッパーを配置
