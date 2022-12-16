---
title: "Amplify Predictions(Amazon Polly)で多言語対応した音声読み上げサイトを作る"
emoji: "🗣️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - "nextjs"
  - "react"
  - "amplify"
  - "aws"
  - "polly"
published: true
---

:::message
この記事は [AWS Amplify と AWS× フロントエンド Advent Calendar 2022](https://qiita.com/advent-calendar/2022/amplify) の N 日目の記事です。
:::

こんにちわ。 [ZUMA](https://twitter.com/zuma_lab) です。

Next.js と Amplify Predictions カテゴリの Amazon Polly を使って多言語に対応した音声読み上げサイトを作ってみます。

以下成果物になります。

動作環境はローカルですが、ものすごい短い文章でも読み上げ結果レスポンスは日本語より英語の方が速かったです。

https://www.youtube.com/watch?v=zIn1YAnB1mQ

Amazon Transcribe は音声をテキストに自動的に変換するサービスです。

2022/12/15 現在、実に 37 言語に対応しています。

https://docs.aws.amazon.com/transcribe/latest/dg/supported-languages.html

Amplify Predictions カテゴリの Amazon Transcribe はリアルタイム音声読み上げが出来る Amazon Transcribe Streaming を使用しています。

https://docs.aws.amazon.com/transcribe/latest/dg/streaming.html

調査した所、Amazon Transcribe Streaming に対応している言語は 17 言語でした。

今回作るサイトは読み上げをする対象言語、全 17 言語の切り替えが出来るようにしてみたいと思います。

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
npx create-next-app@latest --ts --use-npm --eslint nextjs-amplify-text-to-speech
```

プロジェクトルートディレクトリへ移動します。

```
cd nextjs-amplify-text-to-speech
```

Amplify 関連パッケージをインストールします。

```
npm install aws-amplify @aws-amplify/predictions
```

音声を再生する為、React Audio Player をインストールします。

```
npm install react-audio-player
```

# Amplify を設定する

Amplify Predictions カテゴリを使用する為には Amplify Auth カテゴリを追加する必要があります。

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

## Amplify Predictions カテゴリを追加する

次に Amazon Polly を使えるようにします。

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

ソースとなるテキストの言語を設定します。

ここは後で動的に変更できるように実装します。一旦、US English を選択します。

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

こちらも後で変更できるように実装できます。

一旦 Kevin にしました。

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

カテゴリを追加すると、以下 `aws-exports.js` に項目が追加されます。

```json:bacend-config.json
    "predictions": {
        "convert": {
            "speechGenerator": {
                "region": "ap-northeast-1",
                "proxy": false,
                "defaults": {
                    "VoiceId": "Kevin",
                    "LanguageCode": "en-US"
                }
            },
        }
    }
```

以下のコマンドで作成した Amplify Auth と Predications カテゴリをクラウドにプロビジョニングします。

```
amplify push -y
```

# フロントエンドで Amplify Configure を設定する

以下公式ドキュメントを参考に Amplify Configure 設定をします。

https://docs.amplify.aws/lib/predictions/getting-started/q/platform/js/#configure-the-frontend

```ts:__app.tsx
import { Amplify } from 'aws-amplify';
import {
  Predictions,
  AmazonAIPredictionsProvider
} from '@aws-amplify/predictions';
import awsconfig from './aws-exports';

Amplify.configure(awsconfig);
Amplify.addPluggable(new AmazonAIPredictionsProvider());
```

## Amplify.addPluggable でエラーが発生した場合

Next.js の `__app.tsx` で `Amplify.addPluggable` を呼び出すと以下のエラーが発生する場合があります。

```
Error: Pluggable with name AmazonAIPredictionsProvider has already been added.
```

以下過去記事でエラー回避方法について記載していますので参照ください。

https://zenn.dev/zuma_lab/articles/nextjs-amplify-text-to-translate#amplify.addpluggable-%E3%81%A7%E3%82%A8%E3%83%A9%E3%83%BC%E3%81%8C%E7%99%BA%E7%94%9F%E3%81%97%E3%81%9F%E5%A0%B4%E5%90%88

# テキスト読み上げカスタムフックを実装する

以下 が今回の肝である Amplify Predictions を使用した読み上げカスタムフックの実装となります。

翻訳を実行する `convertSpeechToText` 関数と翻訳結果の値 `transcribeText` を返却します。

`Predictions.convert` を呼び出すだけで Amazon Transcribe を使用した翻訳が実行できます。

https://github.com/kazuma-fujita/nextjs-amplify-speech-to-text/blob/main/src/hooks/use-convert-speech-to-text.ts

`convertSpeechToText` 関数引数の bytes は用意されている BytesSource 型( Buffer | ArrayBuffer | Blob | string のユニオン型)なのですが筆者が試した所読み上げの実行結果、空文字が返却されてしまいました。

Buffer[] | ArrayBuffer[]型のみ読み上げが実行されたので、今回 bytes 引数には仕方なく any 型を指定しています。

また、source には後述する翻訳言語選択プルダウンで選択した言語の Language code を設定しています。

# 音声録音カスタムフックを実装する

以下 `microphone-stream` パッケージを利用した音声 stream 処理の実装です。

公式サンプルを Typescript 化し、useSpeechToText hook として component から呼び出せるようにしています。

サンプルを単純に Typescript 化しても動作せず何時間もハマったので、一部 any 型で逃げています。。

https://github.com/kazuma-fujita/nextjs-amplify-speech-to-text/blob/main/src/hooks/use-speech-to-text.ts

また、録音停止をトリガーに前述した `convertSpeechToText` 関数を呼び出して読み上げを実行しています。

# 翻訳画面を実装する

デザインは `create-next-app` で出力される CSS を利用します。

以下読み上げをする言語選択プルダウンの実装となります。

Select タグの options に Amplify Transcribe Streaming に対応する全言語の言語ラベルと Language code を設定します。

言語選択プルダウンのデフォルト値は日本語の Language code に設定します。

https://github.com/kazuma-fujita/nextjs-amplify-speech-to-text/blob/main/src/data/language-options.ts

https://github.com/kazuma-fujita/nextjs-amplify-speech-to-text/blob/main/src/components/select-box.tsx

以下音声録音コンポーネントの実装です。

useSpeechToText hook を呼び出して録音の開始、終了、読み上げ結果の表示を行っています。

https://github.com/kazuma-fujita/nextjs-amplify-speech-to-text/blob/main/src/components/speech-to-text.tsx

以下プルダウンと音声録音コンポーネントの親コンポーネントです。

https://github.com/kazuma-fujita/nextjs-amplify-speech-to-text/blob/main/src/components/speech-to-text-form.tsx

最後に index.tsx をから入力フォームを呼び出して実装完了です。

https://github.com/kazuma-fujita/nextjs-amplify-speech-to-text/blob/main/pages/index.tsx

# Amazon Transcribe の料金

Amazon Transcribe は１か月あたりの読み上げされた音声データの秒数に基づいた従量課金です。

Batch 処理 と Amplify で使用している Streaming 処理に費用差はなく、0.024USD/分から使用時間によって 0.015USD、0.0102USD と割安になっていきます。

例えば 1 時間の音声処理では 1.44USD となります。

https://aws.amazon.com/jp/polly/pricing/?p=ft&c=ml&z=3

ちなみに、Google の Cloud Speech-to-Text は 1 時間迄無料で、以降基本は 0.024USD となっていました。

https://cloud.google.com/speech-to-text/pricing?hl=ja

# まとめ

- Amplify Predictions カテゴリの Amazon Transcribe はリアルタイム音声読み上げが出来る Amazon Transcribe Streaming を使用している
- Amazon Transcribe の対応言語は 37 言語。その中で Transcribe Streaming に対応している言語は 17 言語。
- Transcribe Streaming は WAV にする前の生 PCM データを Websocket 通信でやり取りする
- Transcribe Streaming に 音声 PCM データを投げるに当たって `microphone-stream` パッケージを使用して音声 stream を取得する必要がある
- 読み上げ処理は `Predictions.convert` 関数の source に PCM データを指定して呼び出すだけで Amazon Transcribe Streaming が実行できる
- 料金は Batch 処理 と Streaming 処理に費用差はなく、従量課金制。0.024USD/分から使用時間によって 0.015USD、0.0102USD と割安になっていく
