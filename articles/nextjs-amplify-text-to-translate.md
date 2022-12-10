---
title: "Amplify Predictions(Amazon Translate)でマルチ言語翻訳サイトを作る"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - "amplify"
  - "nextjs"
  - "translation"
  - "aws"
published: true
---

:::message
この記事は [AWS Amplify と AWS× フロントエンド Advent Calendar 2022](https://qiita.com/advent-calendar/2022/amplify) 10 日目の記事です。
:::

Amplify Predictions カテゴリの Amazon Translate と Next.js を使ってマルチ言語翻訳サイトを作ってみます。

最終的な成果物はこのようになります。

![](https://storage.googleapis.com/zenn-user-upload/5abcf79ed7bb-20221210.gif)

Amazon Translate は 言語翻訳サービスで同様サービスとして DeepL API、Google Cloud Translation などがあります。

以下 Amplify Predictions の公式ドキュメントですが、そもそもの情報が少なかったり、日本語記事が少なかったので今回テーマとして取り上げてみました。

https://docs.amplify.aws/lib/predictions/translate/q/platform/js/

Amazon Translate では実に 75 言語が対応しています。

https://docs.aws.amazon.com/translate/latest/dg/what-is-languages.html

今回は全言語に対応したマルチ言語翻訳サイトを作ってみます。

# 実行環境

- Node
  - 16.13.0
- npm
  - 8.1.0
- Next.js
  - 13.0.5
- Amplify CLI
  - 10.5.1

# Next.js プロジェクトを作成する

以下コマンドを実行して Next.js プロジェクトを作成します。

```
npx create-next-app@latest --ts --use-npm --eslint nextjs-amplify-text-to-translate
```

プロジェクトルートディレクトリへ移動します。

```
cd nextjs-amplify-text-to-translate
```

以下 Amplify 関連パッケージをインストールします。

```
npm install aws-amplify @aws-amplify/predictions
```

# Amplify を設定する

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

以下設問は `Translate text into a different language` を選択します。

```
? What would you like to convert? (Use arrow keys)
❯ Translate text into a different language
  Generate speech audio from text
  Transcribe text from audio
```

翻訳元/翻訳先の言語は後で動的に変更出来るように実装します。

翻訳元の言語はとりあえず日本語に設定します。

```
? What is the source language?
  Hungarian
  Indonesian
  Italian
❯ Japanese
  Korean
  Malay
  Norwegian
```

翻訳先はとりあえず英語に設定します。

```
? What is the target language?
  Czech
  Danish
  Dutch
❯ English
  Finnish
  French
  German
```

今回未認証のゲストユーザーもアクセス出来るようにします。

```
? Who should have access?
  Auth users only
❯ Auth and Guest users
```

回答が完了すると `aws-exports.js` に以下項目が追加されます。

```yml:aws-exports.js
    "predictions": {
        "convert": {
            "translateText": {
                "region": "ap-northeast-1",
                "proxy": false,
                "defaults": {
                    "sourceLanguage": "ja",
                    "targetLanguage": "en"
                }
            }
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

Next.js の `__app.tsx` で `Amplify.addPluggable` を呼び出した所、以下のエラーが発生しました。

```
Error: Pluggable with name AmazonAIPredictionsProvider has already been added.
```

![](https://storage.googleapis.com/zenn-user-upload/50a4d3af59dd-20221209.png =400x)

エラーについてはこちらの issues で議論されていますが、まだ対応されていないようです。

https://github.com/aws-amplify/amplify-js/issues/10112

現状 work around として `Amplify.addPluggable` で発生するエラー を try - catch で握りつぶすと動作します。

```ts:__app.tsx
Amplify.configure(awsconfig);

try {
  Amplify.addPluggable(new AmazonAIPredictionsProvider());
} catch (error) {}

export default function App({ Component, pageProps }: AppProps) {
  return <Component {...pageProps} />;
}
```

# 翻訳カスタムフックを実装する

以下 が今回の肝である Amplify Predictions を使用した翻訳カスタムフックの実装となります。

翻訳を実行する `translateLanguage` 関数と翻訳結果の値 `translatedText` を返却します。

`Predictions.convert` を呼び出すだけで Amazon Translate を使用した翻訳が実行できます。

https://github.com/kazuma-fujita/nextjs-amplify-text-to-translate/blob/main/src/hooks/use-translate-language.ts

後述する翻訳言語選択プルダウンで選択した翻訳元、翻訳先の Language code をそれぞれ sourceLanguage、targetLanguage に設定しています。

# 翻訳画面を実装する

デザインは `create-next-app` で出力される CSS を利用します。

以下翻訳言語選択プルダウンの実装となります。

Select タグの options に Amplify Translate に対応する全言語の言語ラベルと Language code を設定します。

https://github.com/kazuma-fujita/nextjs-amplify-text-to-translate/blob/main/src/data/language-options.ts

https://github.com/kazuma-fujita/nextjs-amplify-text-to-translate/blob/main/src/components/selecct-box.tsx

https://github.com/kazuma-fujita/nextjs-amplify-text-to-translate/blob/main/src/components/exchange-language-select-box.tsx

以下翻訳元入力テキストエリアの実装です。

useTranslateLanguage hook を呼び出して入力値をリアルタイムに翻訳に掛けます。

https://github.com/kazuma-fujita/nextjs-amplify-text-to-translate/blob/main/src/components/translate-language-textarea.tsx

以下プルダウンとテキストエリアの親コンポーネントです。

言語選択プルダウンのデフォルト値は翻訳元を日本語、翻訳先を英語の Language code に設定します。

https://github.com/kazuma-fujita/nextjs-amplify-text-to-translate/blob/main/src/components/translate-language-form.tsx

最後に index.tsx をから入力フォームを呼び出して実装完了です。

https://github.com/kazuma-fujita/nextjs-amplify-text-to-translate/blob/main/pages/index.tsx

# Amazon Translate の利用料金

Amazon Translate の利用料金 は 無料枠を利用する場合、1 か月 200 万文字が 12 か月間無料です。

以降は 100 万文字あたり 15USD で 1 文字あたり 0.000015USD でした。

たとえば、1 か月に 70 万文字を処理するように送信した場合、10.5USD が請求されます。

https://aws.amazon.com/jp/translate/pricing/

75 言語が対応しています。

https://docs.aws.amazon.com/translate/latest/dg/what-is-languages.html

他の翻訳サービスの利用料金も調べてみました。

DeepL API は Free プランで 1 か月 50 万文字まで無料、Pro プランで基本料金 630 円/月に加え、重量課金で 1 文字あたり 0.0025 円でした。

https://support.deepl.com/hc/ja/articles/360020685720-DeepL-API-%E6%96%87%E5%AD%97%E6%95%B0%E3%81%AE%E3%82%AB%E3%82%A6%E3%83%B3%E3%83%88%E3%81%A8%E8%AB%8B%E6%B1%82

対応言語は 27 言語でした。

https://support.deepl.com/hc/ja/articles/360019925219-DeepL-Pro%E3%81%AE%E5%AF%BE%E5%BF%9C%E8%A8%80%E8%AA%9E

Google Cloud Translation は 1 か月 50 万文字まで無料、50 万文字以降は、100 万文字あたり 20 USD でした。

たとえば、1 か月に 70 万文字を処理するように送信した場合、4USD が請求されます。

https://cloud.google.com/translate/pricing?hl=ja

対応言語は 134 言語でした。

https://cloud.google.com/translate/docs/languages?hl=ja

# まとめ

Amplify Predictions カテゴリの Amazon Translate と Next.js を使ってマルチ言語翻訳サイトを作ってみました。

翻訳ロジックの実装は Amplify Predictions カテゴリを追加して、`Predictions.convert` を呼び出すだけで実装できます。

Next.js で Amplify.addPluggable の呼び出しでエラーが発生した場合は、現状取り急ぎ work around として `Amplify.addPluggable` で発生するエラー を try - catch で握りつぶすと動作します。

お手軽に Amplify の AI カテゴリを試すことが出来るので、他の AI サービスもブンブン素振りしていきたいと思います。
