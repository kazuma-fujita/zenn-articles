---
title: "GraphQL Code GeneratorでTypeScriptの型を自動生成してSWRでQueryを実行する"
emoji: "🔧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - "nextjs"
  - "react"
  - "amplify"
  - "aws"
  - "transcribe"
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
npx create-next-app@latest --ts --use-npm --eslint swr2-graphql-todo-app-with-codegen
```

プロジェクトルートディレクトリへ移動します。

```
cd swr2-graphql-todo-app-with-codegen
```

SWR をインストールします。

```
npm install swr
```

GraphQL Code Generator 関連パッケージをインストールします。

```
npm install graphql
npm install -D @graphql-codegen/cli
```

```
npm install -D @graphql-codegen/near-operation-file-preset
```

# GraphQL Code Generator の Initialization Wizard を実行する

今回は公式のセットアップ手順に沿って Initialization Wizard を使ってみたいと思います。

以下コマンドを実行します。

```
npx graphql-code-generator init
```

以下全問デフォルトのまま回答します。

```
    Welcome to GraphQL Code Generator!
    Answer few questions and we will setup everything for you.

? What type of application are you building? Application built with React
? Where is your schema?: (path or url) http://localhost:4000
? Where are your operations and fragments?: src/**/*.tsx
? Where to write the output: src/gql
? Do you want to generate an introspection file? No
? How to name the config file? codegen.ts
? What script in package.json should run the codegen? codegen
```

ウィザードが終了すると自動で package.json に `codegen` コマンドが追加されます。

```json:package.json
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "codegen": "graphql-codegen --config codegen.ts"
  },
```

また、ルートディレクトリに `codegen.ts` が生成されます。

```ts:codegen.ts
import type { CodegenConfig } from '@graphql-codegen/cli';

const config: CodegenConfig = {
  overwrite: true,
  schema: "http://localhost:4000",
  documents: "src/**/*.tsx",
  generates: {
    "src/gql": {
      preset: "client",
      plugins: []
    }
  }
};

export default config;
```

# GraphQL Code Generator のプラグインを追加する

schema の URL は適宜環境に合わせて変更します。

## near-operation-files preset を追加する

near-operation-files preset は GraphQL Operation に対応する GraphQL Code Generator の出力ファイルを、その Operation が記述されてるファイルの近くに書き出してくれる。

https://the-guild.dev/graphql/codegen/plugins/presets/near-operation-file-preset

Next.js 環境の場合、自動的にページで使用する js ファイルが分割されるので、GraphQL Code Generator で出力されたファイルも分割されて欲しいです。

https://hiroppy.me/blog/nextjs-chunk-optimization

また、Fragment Colocation を導入する場合、コンポーネントと Query や Mutation のファイルが近い方が望ましいです。

# Schema ファイルを作成する

# 参考サイト

- https://zenn.dev/izumin/articles/ffc84c1b4310be
