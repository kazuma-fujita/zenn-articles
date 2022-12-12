---
title: "学校や TOEIC で絶対出てこない英語フレーズの発音練習が出来るクソアプリ"
emoji: "🗣️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - "nextjs"
  - "react"
  - "amplify"
  - "aws"
  - "english"
published: true
---

:::message
この記事は [クソアプリ Advent Calendar 2022](https://qiita.com/advent-calendar/2022/kuso-app) 13 日目の記事です。
:::

こんにちわ。 [ZUMA](https://twitter.com/zuma_lab) です。

筆者は英語学習初学者です。字幕無しで洋画を観るのが目標ですが、いつもハリウッド俳優が何を言ってるのかサッパリわかりません。

リスニング出来ない理由として以下分析しました。

- ハリウッド俳優達の会話が速すぎる
- リンキング(単語間が繋がってる)してる
- リダクション(本来発音されるべき音が発音されてない)してる
- **スラングめっちゃ使ってる**

という訳でスラングのリスニングと発音練習が出来る Slang Boot Camp というアプリを作ってみました。

以下成果物です。

まず英語問題文が読み上げれます。次にリスニングした問題文をスピーキングします。発音が正しければ日本語訳の解答が表示され、次の問題文が読み上げれます。

発音が正しければすればスラングで祝ってくれますし、間違っていればスラングで罵られます。

全 30 文あります。

正直実装とかより、この問題文考えるほうが大変でした。

# やってること

技術的には AWS Amplify の Predications カテゴリ(Amazon Polly と Amazon Transcribe)、 Next.js と React Audio Player を使ってます。

1. Amazon Polly で問題文読み上げ
1. React Audio Player でユーザーの音声を録音
1. Amazon Transcribe で音声からテキスト変換
1. 変換されたテキストと問題文が合っているか判定
1. 合っていれば次の問題

# 実行環境

- Node
  - 16.13.0
- npm
  - 8.1.0
- Next.js
  - 13.0.5
- Amplify CLI
  - 10.5.2

# Next.js プロジェクトを作成する

以下コマンドを実行して Next.js プロジェクトを作成します。

```
npx create-next-app@latest --ts --use-npm --eslint slang-boot-camp-frontend
```

プロジェクトルートディレクトリへ移動します。

```
cd slang-boot-camp-frontend
```

Amplify 関連パッケージをインストールします。

```
npm install aws-amplify @aws-amplify/predictions
```

React Audio Player をインストールします。

```
npm install react-audio-player
```

# Amplify を設定する

Amplify Predictions カテゴリを使用する為には Amplify Auth カテゴリを追加する必要があります。

まずは Auth カテゴリを追加します。

## Amplify Auth カテゴリを追加する

前提として `amplify configure` で Amplify で使用する AWS リソースにアクセス可能な IAM ユーザーは作成済みとします。

また、 `amplify init` で Amplify の初期化済みとします。

プロジェクトルートで以下のコマンドを実行して Amplify Auth カテゴリを追加します。

```
amplify add auth
```

Cognito の設定は最低限メールアドレスとパスワードのみの認証とします。

```
Using service: Cognito, provided by: awscloudformation

 The current configured provider is Amazon Cognito.

 Do you want to use the default authentication and security configuration?
❯ Default configuration
  Default configuration with Social Provider (Federation)
  Manual configuration
  I want to learn more.
 How do you want users to be able to sign in?
  Username
❯ Email
  Phone Number
  Email or Phone Number
  I want to learn more.
 Do you want to configure advanced settings? (Use arrow keys)
❯ No, I am done.
  Yes, I want to make some additional changes.
✅ Successfully added auth resource nextjscognitomiddlewe0c8d7ed locally

✅ Some next steps:
"amplify push" will build all your local backend resources and provision it in the cloud
"amplify publish" will build all your local backend and frontend resources (if you have hosting category added) and provision it in the cloud
```

# Amplify Predictions カテゴリを追加する

## Text to speech (Amazon Polly) を追加する

英語問題文を読み上げる為、Amazon Polly を使えるようにします。

以下コマンドを追加して Amplify Predictions カテゴリを追加します。

```
amplify add predictions
```

以下設問は `Convert` を選択します。

```
? Please select from one of the categories below
  Identify
❯ Convert
  Interpret
  Infer
  Learn More
```

以下設問は `Generate speech audio from text` を選択します。

```
? What would you like to convert?
  Translate text into a different language
❯ Generate speech audio from text
  Transcribe text from audio
```

以下設問はデフォルトのままとします。

```
? Provide a friendly name for your resource (speechGeneratora239dcbe)
```

ソースとなるテキストの言語を設定します。 US English を選択します。

```
? What is the source language? (Use arrow keys)
❯ US English
  Turkish
  Swedish
  Russian
  Romanian
  Portuguese
  Brazilian Portuguese
(Move up and down to reveal more choices)
```

音声読み上げをするキャラ(?)を選択します。

今回は Kevin にしました。

```
? Select a speaker (Use arrow keys)
❯ Kevin - Male
  Salli - Female
  Matthew - Male
  Kimberly - Female
  Kendra - Female
  Justin - Male
  Joey - Male
```

今回は未認証のゲストユーザーでもアクセス出来ように設定します。

```
? Who should have access?
  Auth users only
❯ Auth and Guest users
```

カテゴリを追加すると、以下 `bacend-config.json` に項目が追加されます。

```json:bacend-config.json
  "predictions": {
    "speechGeneratora239dcbe": {
      "providerPlugin": "awscloudformation",
      "service": "Polly",
      "convertType": "speechGenerator"
    }
  }
```

## Transcribe audio to text (Amazon Transcribe) を追加する

次に、ユーザーの音声からテキストに変換する為、Amazon Transcribe を使えるようにします。

以下コマンドを再度実行します。

```
amplify add predictions
```

以下設問は `Convert` を選択します。

```
? Please select from one of the categories below
  Identify
❯ Convert
  Interpret
  Infer
  Learn More
```

以下設問は `Transcribe text from audio` を選択します。

```
? What would you like to convert?
  Translate text into a different language
  Generate speech audio from text
❯ Transcribe text from audio
```

以下設問はデフォルトのままとします。

```
? Provide a friendly name for your resource (transcription0e88a3c3)
```

ソースとなる音声の言語を選択します。US English を選択します。

```
? What is the source language?
  British English
❯ US English
  French
  Canadian French
  US Spanish
```

未認証のゲストユーザーでもアクセス出来ように設定します。

```
? Who should have access?
  Auth users only
❯ Auth and Guest users
```

カテゴリを追加すると、以下 `bacend-config.json` に項目が追加されます。

```json:bacend-config.json
  "predictions": {
    "speechGeneratora239dcbe": {
      "providerPlugin": "awscloudformation",
      "service": "Polly",
      "convertType": "speechGenerator"
    }
  }
```

以下のコマンドで作成した Amplify Auth と Predications カテゴリをクラウドにプロビジョニングします。

```
amplify push -y
```

# 画面を実装する
