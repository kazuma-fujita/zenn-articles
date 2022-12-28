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
この記事は [AWS Amplify と AWS× フロントエンド Advent Calendar 2022](https://qiita.com/advent-calendar/2022/amplify) の 19 日目の記事です。
:::

こんにちわ。 [ZUMA](https://twitter.com/zuma_lab) です。

Next.js と Amplify Predictions カテゴリの Amazon Polly を使って多言語に対応した音声読み上げサイトを作ってみます。

以下成果物になります。

https://youtu.be/MpUSaHkkl-A

# Amazon Polly の事前知識

Amazon Polly はテキストを音声に自動的に変換するサービスです。

2022/12/19 現在、Amazon Polly では [計 36 の言語と方言](https://docs.aws.amazon.com/polly/latest/dg/voicelist.html) に対応しています。

Amazon Polly は言語に加え、Gender 別に様々な音声が用意されています。

English(US)に関しては子供の音声まで用意されていました。

![](https://storage.googleapis.com/zenn-user-upload/02f8555e167f-20221219.png)

それぞれの音声の中でも、通常の Text To Speech(以下 TTS) システムを使った Standard Voice の他に [Neural Text To Speech(以下 NTTS) システムを使った Neural Voice](https://docs.aws.amazon.com/ja_jp/polly/latest/dg/NTTS-main.html) が用意されています。

> Amazon Polly にはニューラル TTS (NTTS) システムがあり、標準音声よりも高品質の音声を生成できます。NTTS システムは、最も自然で人間らしいものを生み出します。

NTTS では、可能な限り自然で人間に似たテキスト読み上げ音声を生成しており、親しみやすく、スムーズに聞こえるそうです。

日本語では TTS のみ対応した Mizuki と NTTS に対応した Takumi が用意されています。

実際に TTS と NTTS を比較してみました。

最初に Mizuki の音声、次に Takumi の音声が流れます。

https://youtu.be/pNFhRtIyTT0

確かに、Mizuki の方は良くある機械音声読み上げのような棒読み感があるのに対して、Takumi はナチュナルな発声やイントネーションが再現出来ている気がします。

Amplify Predications は NTTS / TTS 両対応の音声をサポートしています。

残念ながら NTTS のみ対応している音声はサポートしていません。

それでも計 29 の言語と方言に対応した音声を選択する事が出来ます。

今回作るサイトは Amplify Predications カテゴリの Amazon Polly に対応した全音声の切り替えが出来るようにしてみたいと思います。

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
  Polly text from audio
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

音声を選択します。こちらも後で変更できるように実装します。

一旦 Kevin を選択しました。

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

:::message alert
Kevin は前述した NTTS のみ対応した音声なので、Amplify Predictions カテゴリの Amazon Polly で使用するとエラーになります。
Amplify CLI の選択肢にはたまに落とし穴があるので、使用する前に Amazon Polly の公式ドキュメントを読む事をオススメします。
:::

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

[公式ドキュメント](https://docs.amplify.aws/lib/predictions/getting-started/q/platform/js/#configure-the-frontend) を参考に Amplify Configure 設定をします。

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

:::message alert
Next.js の `__app.tsx` で `Amplify.addPluggable` を呼び出すと以下のエラーが発生する場合があります。

```
Error: Pluggable with name AmazonAIPredictionsProvider has already been added.
```

もしエラーが発生したら [筆者の過去記事](https://zenn.dev/zuma_lab/articles/nextjs-amplify-text-to-translate#amplify.addpluggable-%E3%81%A7%E3%82%A8%E3%83%A9%E3%83%BC%E3%81%8C%E7%99%BA%E7%94%9F%E3%81%97%E3%81%9F%E5%A0%B4%E5%90%88) でエラー回避方法について記載していますので参照ください。
:::

# 音声読み上げカスタムフックを実装する

以下 が今回の肝である Amplify Predictions を使用した読み上げカスタムフックの実装となります。

useConvertTextToSpeech hook では翻訳を実行する `convertTextToSpeech` 関数と音声変換結果の Blob URL Scheme である `speechBlobUrl` を返却します。

`Predictions.convert` を呼び出すだけで Amazon Polly を使用した翻訳が実行できます。

https://github.com/kazuma-fujita/nextjs-amplify-text-to-speech/blob/main/src/hooks/use-convert-text-to-speech.ts

# 音声読み上げ画面を実装する

デザインは `create-next-app` で出力される CSS を利用します。

以下読み上げをする言語・音声選択プルダウンの実装となります。

Select タグの options に Amplify Polly に対応する全言語の言語ラベルと音声の ID である voiceId を設定します。

言語選択プルダウンのデフォルト値は日本語の NTTS 音声 Takumi に設定します。

https://github.com/kazuma-fujita/nextjs-amplify-text-to-speech/blob/main/src/data/language-options.ts

https://github.com/kazuma-fujita/nextjs-amplify-text-to-speech/blob/main/src/components/select-box.tsx

以下音声読み上げコンポーネントの実装です。

useConvertTextToSpeech hook を呼び出して入力されたテキストを `convertTextToSpeech` 関数で音声変換、その結果の Blob URL Scheme である `speechBlobUrl` を生成します。

`speechBlobUrl` の値があれば、 `ReactAudioPlayer` で音声を再生します。

`ReactAudioPlayer` に bool 値である `autoPlay` を設定すると `speechBlobUrl` が変わる度に音声を自動で再生してくれます。

https://github.com/kazuma-fujita/nextjs-amplify-text-to-speech/blob/main/src/components/text-to-speech.tsx

以下プルダウンと音声読み上げコンポーネントの親コンポーネントです。

https://github.com/kazuma-fujita/nextjs-amplify-text-to-speech/blob/main/src/components/text-to-speech-form.tsx

最後に index.tsx をから入力フォームを呼び出して実装完了です。

https://github.com/kazuma-fujita/nextjs-amplify-text-to-speech/blob/main/pages/index.tsx

# Amazon Polly の料金

Amazon Polly は１か月あたりの読み上げされた文字数に基づいた従量課金です。

TTS は 100 万字に対して 4.00 USD 、NTTS は 100 万字に対して 16.00 USD でした。

NTTS は TTS に比べて 4 倍の料金でした。

https://aws.amazon.com/jp/polly/pricing/

ちなみに、Google の Cloud Text-to-Speech は標準音声 4.00USD/100 万文字、Neural 音声 16.00USD/100 万文字でした。

https://cloud.google.com/text-to-speech/pricing?hl=ja

Cloud Text-to-Speech は Polly と同じような料金形態でした。

# まとめ

- Amazon Polly はテキストを音声に自動的に変換するサービスで計 36 の言語と方言に対応
- Amazon Polly は言語に加え、Gender 別に様々な音声が用意されている。English(US)に関しては子供の音声まで対応
- Amazon Polly にはニューラル TTS (NTTS) システムがあり、標準音声よりも高品質の音声を生成可能
- Amplify Predications は NTTS / TTS 両対応の音声をサポート。NTTS のみ対応している音声は対応していない。それでも計 29 の言語と方言に対応した音声を選択可能
- 読み上げ処理は `Predictions.convert` 関数の source に 入力テキストを指定して呼び出すだけで Amazon Polly が実行可能
- 料金は１か月あたりの読み上げされた文字数に基づいた従量課金。TTS 4.00USD/100 万文字、NTTS 16.00USD/100 万文字
