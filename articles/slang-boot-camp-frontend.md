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

クソアプ初参加です。お手柔らかにお願いします。

学校や TOEIC のテストでは絶対出てこないであろうスラングのリスニングと発音練習が出来る Slang Boot Camp という WEB アプリを作ってみました。

このアプリで覚えたフレーズは英語のテストで一切役に立たない事を保証いたします。

# 動機

筆者の英語力は中二レベルです。

いつか字幕無しで洋画を観るという目標がありますが、現状ハリウッド俳優が何を言ってるのかサッパリわかりません。

筆者がリスニング出来ない理由として主に以下が挙げられます。

- 英語力が中二で止まっている
- ハリウッド俳優達の会話が速くて追いつけない
- リンキング(単語間が繋がってる)してる
- リダクション(本来発音されるべき音が発音されてない)してる
- **スラングめっちゃ使ってる**

英語学習において他にやること山の如しですが、とりあえずスラング覚えたいなーと思ってこのアプリを作りました。

# こんなんです

以下成果物です。

まず英語問題文が読み上げれます。次にユーザーはリスニングした問題文をスピーキングします。

発音が正しければ日本語訳の解答が表示され次の問題が出題されます。

発音が正しければすればスラングで祝ってくれますし、間違っていればスラングで罵られます。

筆者が独断と偏見でレベル分けをした問題が全部で 30 問あります。

正直実装とかより、問題考えるほうが大変でした。

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

音声を再生する為、React Audio Player をインストールします。

```
npm install react-audio-player
```

音声を録音する為、React Media Recorder をインストールします。

こちらは後述しますが、訳あってバージョンを固定しています。

```
npm install react-media-recorder@1.6.5
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
    },
    "transcription0e88a3c3": {
      "providerPlugin": "awscloudformation",
      "service": "Transcribe",
      "convertType": "transcription"
    }
  }
```

以下のコマンドで作成した Amplify Auth と Predications カテゴリをクラウドにプロビジョニングします。

```
amplify push -y
```

# 画面を実装する

## ReferenceError: Blob is not defined が発生する場合

Next.js で React Media Recorder パッケージを使用すると `Blob is not defined` が発生する場合があります。

以下の issues でエラーについて議論されていました。

https://github.com/0x006F/react-media-recorder/issues/103

Next.js はデフォルトでサーバサイドでレンダリングされるので MediaRecorder などクライアントサイドで動作するモジュールは動的 import をする必要があるようです。

issues を参考に以下のように修正しました。

```ts:index.tsx
const Recorder = dynamic(
  () =>
    import("../src/components/audio-recorder").then(
      (module) => module.AudioRecorder
    ),
  { ssr: false }
);
```

ただ、この `Blob is not defined` エラーを解消すると次は以下のエラーが発生しました。

```
Error: There is already an encoder stored which handles exactly the same mime types.
```

以下の issues でエラーについて議論されていました。

https://github.com/0x006F/react-media-recorder/issues/98#issuecomment-1133918236

以下和訳引用

> Next.js はデフォルト `next.config.js` の `reactStrictMode` が true になっています。
> Strict モードではすべてのコンポーネントが 2 回呼び出されるため、Extendable-Media-Recorder でこのエラーがスローされます。

Strict モードを OFF にするとエラーは発生しませんが、本番運用のプロジェクトの開発では極力 Strict モードで開発したいです。

ワークアラウンドな対処法になってしまいますが、2022/12/12 時点で最新である React Media Recorder バージョン 1.6.6 を `npm i react-media-recorder@1.6.5` で バージョンを 1.6.5 に固定すると取り急ぎ全てのエラーが解消されます。

今回はとりあえずバージョン固定しました。
