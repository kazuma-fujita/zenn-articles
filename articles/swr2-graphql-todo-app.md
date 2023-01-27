---
title: "TodoアプリでSWR2の動きを確認する"
emoji: "🔧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - "nextjs"
  - "react"
  - "swr"
  - ""
  - ""
published: false
---

こんにちわ。 [ZUMA](https://twitter.com/zuma_lab) です。

GraphQLZero というフリーの GraphQL のモック API に対して Query を実行してみます。

https://graphqlzero.almansi.me/

# 実行環境

- Node
  - 16.13.0
- npm
  - 8.1.0
- Next.js
  - 13.1.1
- SWR
  - 2.0.0
- graphql
  - 16.6.0
- graphql-codegen/cli
  - 2.16.2

# Next.js プロジェクトを作成する

以下コマンドを実行して Next.js プロジェクトを作成します。

```
npx create-next-app@latest --ts --use-npm --eslint swr2-graphql-todo-app
```

プロジェクトルートディレクトリへ移動します。

```
cd swr2-graphql-todo-app
```

SWR と GraphQL 関連パッケージをインストールします。

```
npm install swr graphql-request graphql
```
