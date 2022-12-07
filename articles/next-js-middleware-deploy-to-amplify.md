---
title: "Next.jsのMiddlewareをAmplify Hostingで動かす"
emoji: "🧞‍♂️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - "amplify"
  - "nextjs"
  - "cognito"
  - "aws"
published: true
---

先日、Amplify Hosting が Next.js12 と 13 に対応して Next.js の Middleware が Amplify で使えるようになりました。

https://aws.amazon.com/jp/blogs/mobile/amplify-next-js-13/

Next.js には Middleware というリクエストが完了する前にエッジ環境で処理を実行する仕組みがあります。

https://nextjs.org/docs/advanced-features/middleware

Next.js の v12.2 から Middleware が stable になっています。

今回は Amplify を使って Next.js の Middleware を動かしてみたいと思います。

最初に結論ですが、Middleware を使用する上で特別な設定をしなくても Amplify にデプロイするだけで動きます。

最初もっとハマるかと思ったので拍子抜けでした。

ですが、Middleware の中で環境変数を使用する場合、Amplify 特有のハマり所があったので Tips として残しておきます。

# 前提

前回、Next.js の Middleware で Amplify Auth(Cognito)の認証状態によってログイン前後の画面を振り分ける記事を書きました。

https://zenn.dev/zuma_lab/articles/74a999aa1a8c59

認証が必須の MPA の WEB アプリケーションを実装する際に以下の URL ルーティング要件があったとします。

- ルート URL にアクセスがあった場合、認証前であればログイン画面、認証後であればダッシュボード画面にリダイレクトする
- 認証前にダッシュボード画面の URL を直接叩かれた場合にログイン画面にリダイレクトする
- 認証後にログイン画面の URL を直接叩かれた場合にダッシュボード画面にリダイレクトする
- 利用規約やプライバシーポリシー画面は認証状態に関係無く表示する

前回の記事ではこの要件を満たすために Middleware で Cognito の認証状態を判定して URL ルーティングを実装しました。

Middleware はプロジェクトルートに `middleware.ts` ファイルを作成するだけで Vercel や Amplify にデプロイ後エッジ環境で動作します。

以下 Middleware の実装となります。(Cognito の認証判定の実装は前回記事を参照ください)

https://github.com/kazuma-fujita/nextjs-cognito-middleware-sample/blob/main/middleware.ts

今回はローカルで実装した認証 Middleware を Amplify で動かしてみます。

# 実行環境

- Node
  - 16.13.0
- npm
  - 8.1.0
- Next.js
  - 13.0.5
- Amplify CLI
  - 10.5.1

# Amplify Hosting カテゴリを追加してデプロイする

前提として [前回の記事の Amplify を設定する](https://zenn.dev/zuma_lab/articles/74a999aa1a8c59#amplify-%E3%82%92%E8%A8%AD%E5%AE%9A%E3%81%99%E3%82%8B) で Amplify の初期化は実行済みとします。

以下コマンドを実行して Amplify Hosting カテゴリを追加します。

```
amplify add hosting
```

以下設問は `Hosting with Amplify Console` を選択します。

```
? Select the plugin module to execute …  (Use arrow keys or type to filter)
❯ Hosting with Amplify Console (Managed hosting with custom domains, Continuous deployment)
  Amazon CloudFront and S3
```

恐らく本番運用ではほとんどのケースで Amplify Hosting は Github と連携して使うと思うので、次の設問は `Continuous deployment` を選択します。

```
? Choose a type
❯ Continuous deployment (Git-based deployments)
  Manual deployment
  Learn more
```

AWS コンソール画面が立ち上がるので、`Frontend environments` タブを開きます。

Github リポジトリと連携する為、Github を選択し `ブランチを接続` ボタンを押下します。

![](https://storage.googleapis.com/zenn-user-upload/d00977f5194c-20221206.png)

Github の Authorize AWS Amplify 画面になるので、`Authorize aws-amplify-console` ボタンを押下します。

![](https://storage.googleapis.com/zenn-user-upload/90f54fb69f32-20221206.png)

Amplify コンソールに戻り「GitHub 認証が成功しました。」と表示されていることを確認します。

次に Amplify と連携するリポジトリとブランチを選択して `次へ` ボタンを押下します。

![](https://storage.googleapis.com/zenn-user-upload/290e9053b4a1-20221206.png)

ビルドの設定の App name はデフォルトのままで OK で、 Environment は `dev` を選択します。

まだ Amplify 用の Service role が無い場合は、 `新しいロールを作成` ボタン押下して Service role を作成します。

`次へ` ボタンを押下します。

![](https://storage.googleapis.com/zenn-user-upload/00504462f433-20221206.png)

次の画面で設定を確認したら `保存してデプロイ` ボタンを押下します。

# Amplify に環境変数を設定する

Middleware 内で環境変数を使用している場合、Amplify に環境変数を設定します。

[前回の記事の環境変数を設定する](https://zenn.dev/zuma_lab/articles/74a999aa1a8c59#%E7%92%B0%E5%A2%83%E5%A4%89%E6%95%B0%E3%82%92%E8%A8%AD%E5%AE%9A%E3%81%99%E3%82%8B) では Cognito の情報としてユーザープール ID やクライアント ID を環境変数に設定して Middleware から呼び出しました。

アプリの設定から `環境変数` 画面を開いて環境変数を設定します。

今回は `COGNITO_` と prefix が付いている変数と`REGION` という変数を追加しました。

![](https://storage.googleapis.com/zenn-user-upload/c47278f1d21e-20221206.png)

:::message alert
`AWS` が予約語になっているので環境変数には `AWS_XXXXX` 等の prefix は付ける事ができません。
https://docs.aws.amazon.com/amplify/latest/userguide/environment-variables.html
:::

環境変数を反映する為に再度デプロイを実行します。

Amplify で Middleware や SSR などサーバーサイドで環境変数を使用する場合、ビルド設定に変数を `.env` ファイルに書き出す追加処理が必要です。

https://docs.aws.amazon.com/ja_jp/amplify/latest/userguide/environment-variables.html

公式を参考にビルド設定の build commands に環境変数を `.env` に書き出す処理を追加しました。

ちなみにビルドの設定から `amplify.yml` をダウンロードして、プロジェクトのルートディレクトリに配置するとデプロイ時にビルド設定を上書きできるようになります。

```yml:amplify.yml
version: 1
backend:
  phases:
    build:
      commands:
        - "# Execute Amplify CLI with the helper script"
        - amplifyPush --simple
frontend:
  phases:
    preBuild:
      commands:
        - npm ci
    build:
      commands:
        - npm run build
        - echo "REGION=$REGION" >> .env
        - echo "COGNITO_USER_POOLS_ID=$COGNITO_USER_POOLS_ID" >> .env
        - echo "COGNITO_USER_POOLS_WEB_CLIENT_ID=$COGNITO_USER_POOLS_WEB_CLIENT_ID" >> .env
        - echo "COGNITO_URL=$COGNITO_URL" >> .env
  artifacts:
    baseDirectory: .next
    files:
      - "**/*"
  cache:
    paths:
      - node_modules/**/*
```

これで問題無いかのように思われましたがしかし・・

`amplify.yml` を Github に push して Amplify にデプロイしても Middleware から環境変数を取得することが出来ませんでした。

デバッグしようにも Middleware のソースコードに console.log を挟んでみたりしましたがデバッグログがどこにも出力されずしばらくハマってしまいました。

Middleware や SSR などサーバーサイド処理も Lambda のように console.log で CloudWatch にログが出力されるかと思ったのですが。。

(どなたか Amplify のサーバーサイド処理のログの出し方分かればご教授ください 🙇)

調べた結果 Amplify ｘ Next.js の場合、ルートディレクトリにある `next.config.js` の module.exports に env 設定をすると Amplify のサーバーサイド処理で環境変数が使用できるようです。

`next.config.js` には以下のように環境変数を設定しました。

https://github.com/kazuma-fujita/nextjs-cognito-middleware-sample/blob/main/next.config.js

# 動作確認

`https://main.XXXXXXXXXX.amplifyapp.com/signin` にアクセスすると AmplifyUI の Authenticator を使用したログイン画面が表示されるので ID/PASS を入力してログインします。

![](https://storage.googleapis.com/zenn-user-upload/713f1038e7c5-20221207.png)

Cognito に認証済みの状態で再度ログイン画面にアクセスします。

Middleware で Cognito の認証状態の判定結果、認証済みなのでダッシュボード画面にリダイレクトされます。

![](https://storage.googleapis.com/zenn-user-upload/9bdf45cf3367-20221207.png)

開発者コンソールのネットワークを開くと `/signin` のアクセスが HTTP ステータスコード 307 で `/dashboard` にリダイレクトされていることが分かります。

![](https://storage.googleapis.com/zenn-user-upload/580c3ff01ce6-20221207.png)

次に Sign Out をして未認証状態で ダッシュボード画面(`https://main.XXXXXXXXXX.amplifyapp.com/dashboard`) にアクセスします。

ログイン画面が表示され `/dashboard` のアクセスが HTTP ステータスコード 307 で `/signin` にリダイレクトされていることが分かります。

![](https://storage.googleapis.com/zenn-user-upload/ff63ce2fe827-20221207.png)

レスポンスヘッダーを確認すると Amplify エッジ環境の CDN(CloudFront)を経由していることが分かります。

```
$ curl -I https://main.dmgki8ps6fwkm.amplifyapp.com/dashboard
HTTP/2 307
server: CloudFront
date: Wed, 07 Dec 2022 01:18:12 GMT
location: /signin
x-cache: Miss from cloudfront
via: 1.1 497e68f1c2171c15557d721da06055d0.cloudfront.net (CloudFront)
x-amz-cf-pop: NRT57-C2
x-amz-cf-id: hSJE4CfRv5S4Y6SR7SgYRzLtrIZyF6CwVrPSTfphMvmWI1R04GLCqQ==
```

ちなみに今回実装した認証 Middleware はルート URL にアクセスした場合、Middleware で認証状態を判定して未ログインならログイン画面、認証済みならダッシュボード画面に振り分けます。

また、Next.js の Middleware は URI のパスを正規表現で判定し、静的ファイルなどは Middleware をパスする事も出来ます。

利用規約など静的画面は Middleware を通さず、認証判定に関係無く表示する等も可能です。

# まとめ

Amplify Hosting で Next.js の Middleware を使用する上で特別な設定をしなくても Amplify にデプロイするだけで動きます。

但し、Middleware や SSR 等のサーバーサイドで環境変数を扱いたい場合は Amplify に環境変数を設定する他、 `next.config.js` に環境変数を設定する必要がありました。

今回認証 Middleware を実装しましたが、Middleware は Basic 認証や IP 制限を掛けたり、Bot のクローラーを除外するなど様々な処理が出来るようです。

アプリケーション側で実装している処理を Middleware で実装出来ないか考えてみても良いかもしれませんね。

# 参考サイト

https://zenn.dev/thim/articles/04775c68d796445f3c90
