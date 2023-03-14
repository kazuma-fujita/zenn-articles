---
title: "ChatGPT API と LINE チャットボットで ChatGPT クローンを作る"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - "ChatGPT"
  - "LINE"
  - "AWS"
  - "Lambda"
  - "Amplify"
published: true
---

こんにちわ。 [ZUMA](https://twitter.com/zuma_lab) です。

今回は ChatGPT API を使って、ChatGPT クローンの LINE チャットボットを作ってみました。

GPT-3.5 モデルを実行する為に OpenAI の Completions API を使います。

https://platform.openai.com/docs/api-reference/completions

LINE チャットボットの入力を AWS Lambda に投げる、Lambda で Completions API を実行、レスポンスをチャットボットに表示という流れで実装します。

# 成果物

以下成果物です。

https://www.youtube.com/watch?v=-S_RCPWaQ6w

質問の仕方や内容によっては回答に数分かかったりします。

どう改善すれば良いかアタリをつけて聞いてみました。

なるほど、Completions API の `max_tokens`や `n` パラメーターを調整すれば改善の余地はありそうですね。

![](https://storage.googleapis.com/zenn-user-upload/224ca62f7dc4-20230208.png)

Python で Completions API を実行するコードも教えてくれます。

![](https://storage.googleapis.com/zenn-user-upload/91a061d4cbb9-20230208.png)

あとはコードのリファクタリング、コメントの追記までやってくれました。

![](https://storage.googleapis.com/zenn-user-upload/6915b55b9c6f-20230208.png)

# 前提

今回は以下前回記事の GPT-3 実装部分を GPT-3.5 に変更した差分のみを抜粋した記事となります。

https://zenn.dev/zuma_lab/articles/gpt3-line-chatbot

サーバ側である Amplify 環境の構築、LINE の各種トークン、OpenAI の API キー取得方法は前回の記事を参照ください。

GPT-3.5 の Lambda 実装以外は前回記事の手順でボット作成出来ます。

# 実行環境

- macOS Ventura 13.1（22C65）
- Amplify 10.8.1
- pipenv version 2022.12.19
- virtualenv 20.17.1

# 実装コード

Python 初心者なので、ChatGPT にベースを書いてもらって後はググりながら書きました。

ChatGPT に実装を助けて貰ってるので私が記法など理解してなかったりします。

誤っている箇所あればマサカリお願いします。

## OpenAI Completions API 実装

OpenAI の Completions API を実行するコードです。

細かい Completions API のパラメーターについては OpenAI の [こちら](https://platform.openai.com/docs/api-reference/completions/create) を参照ください。

https://github.com/kazuma-fujita/line-chat-gpt-bot-demo/blob/develop/amplify/backend/function/lineChatGPTBotDemoFunction/src/gpt3_api.py

公式にも書いてますが、 `max_tokens` や `n` の値を大きくしすぎるとすぐに API の利用無料枠使ってしまう可能性があるので注意が必要です。

OpenAI の API は無料枠 $18、有効期限 3 ヶ月です。

また、ここは ChatGPT に頼らず書いたので、実装した後に気付いたのですが [OpenAI のライブラリ](https://platform.openai.com/docs/quickstart/build-your-application) がありました。。

以下インストールして...

```
pipenv install openai
```

以下のように簡単に Completions API を実行する事ができます。

```py
import openai
import sys

prompt=f"{sys.argv[1]}->"
response = openai.Completion.create(
    model="text-davinci-003",
    prompt=prompt)

print(prompt+response['choices'][0]['text'])
```

最初から ChatGPT に実装方法を聞けば良かったです。

## LINE Messaging API 実装

次に LINE Bot に GPT-3 API のレスポンスを返却する実装です。

ここは [LINE 公式のライブラリ](https://github.com/line/line-bot-sdk-python) を使用しました。

https://github.com/kazuma-fujita/line-chat-gpt-bot-demo/blob/develop/amplify/backend/function/lineChatGPTBotDemoFunction/src/line_api.py

ポイントとしては、GPT-3 API のレスポンスにはプロンプトの投げ方(質問の仕方)によってレスポンステキストの先頭に改行が入るので `strip` で改行を削除してます。

```py
completed_text = gpt3_api.completions(prompt)
response_message = completed_text.strip()
```

また、LINE ChatBOT で質問を入力した際に LINE サーバーから送られてくるリクエストは `event` で渡されますが、リクエストの値は以下のようになっています。

https://github.com/kazuma-fujita/line-chat-gpt-bot-demo/blob/develop/amplify/backend/function/lineChatGPTBotDemoFunction/src/event.json

上記は見やすくする為、改行や細かい値を修正して json 形式に整形しているのですが、実際の LINE サーバーのリクエストに含まれる body の値は文字列になっていたりします。

LINE Messaging API 実装時のポイントとして、body の値は `json.loads(event['body'])` として json 形式にパースしてから受信メッセージやリプライトークンを取得する必要があります。

```py
        # Parse the event body as a JSON object
        event_body = json.loads(event['body'])

        # Check if the event is a message type and is of text type
        if event_body['events'][0]['type'] == 'message' and event_body['events'][0]['message']['type'] == 'text':
            # Get the reply token from the event
            replyToken = event_body['events'][0]['replyToken']
            # Get the prompt text from the event
            prompt = event_body['events'][0]['message']['text']
```

## LINE サーバー以外からのリクエストを弾く

このままだと Lambda の API が丸裸なので、外部から実行される可能性があります。

外部からリクエストが来ても LINE のリプライトークンが一致していない限り ChatBot に返信は出来ないのですが、そもそも LINE サーバー以外からのリクエストはエラーとしましょう。

LINE Developers の設定で Webhook のサーバー IP アドレスを指定する事が出来きます。

ですが Amplify のみで Lambda の IP アドレスの固定化は難しそうだったので今回は LINE サーバーからリクエストヘッダーで送られくる `x-line-signature` を検証する形にします。

LINE Developers の公式で [署名の検証の方法](https://developers.line.biz/ja/reference/messaging-api/#signature-validation) が記載されています。

以下実装コードです。

https://github.com/kazuma-fujita/line-chat-gpt-bot-demo/blob/develop/amplify/backend/function/lineChatGPTBotDemoFunction/src/guard.py

## LINE トークン ・ OpenAI API キーをシークレットから取得する

LINE のチャネルトークンやシークレットトークン、OpenAI の API キーなど秘匿情報は AWS Systems Manager Parameter Store に登録して隠蔽します。

AWS Systems Manager のシークレットの登録方法は後述しますので、シークレットの値を取得するコードを先に出します。

boto3 で簡単に AWS System Manager から値を取得することが可能です。

https://github.com/kazuma-fujita/line-chat-gpt-bot-demo/blob/develop/amplify/backend/function/lineChatGPTBotDemoFunction/src/aws_systems_manager.py

実装したモジュールを利用して、以下のように定数として取得出来るようにします。

https://github.com/kazuma-fujita/line-chat-gpt-bot-demo/blob/develop/amplify/backend/function/lineChatGPTBotDemoFunction/src/const.py

## Lambda 関数を実装する

最後に Lambda 関数を実装します。

先程実装した LINE サーバー以外のリクエストを弾くモジュールと LINE Messaging API を実行するモジュールを呼び出します。

https://github.com/kazuma-fujita/line-chat-gpt-bot-demo/blob/develop/amplify/backend/function/lineChatGPTBotDemoFunction/src/index.py

ポイントとして、エラー時にもステータスコード 200 を返すようにしています。

[LINE Developers のレスポンス項目](https://developers.line.biz/ja/reference/messaging-api/#response) に以下記載があります。

> LINE プラットフォームから送信される HTTP POST リクエストをボットサーバーで受信したときは、ステータスコード 200 を返してください。
> LINE プラットフォームから疎通確認のために、Webhook イベントが含まれない HTTP POST リクエストが送信されることがあります。

実際に 200 を返却しないと、例えば後述する LINE チャネルトークンを取得する際の手順で、LINE プラットフォームから Webhook の疎通確認をする箇所があります。

疎通確認の際にリクエスト中に Webhook イベントが無い為、実装上エラーとなるのですが、その際に 200 を返却しないと疎通確認が成功しません。

以上、実装コードでした。

次から実装前準備の手順となるので、まずは以降手順の完了後コードを実装してください。

# Amplify を初期化する

:::message
前提として `amplify configure` で Amplify で使用する AWS リソースにアクセス可能な IAM ユーザーは作成済みとします。
:::

プロジェクトルートディレクトリを作成して移動します。

```
mkdir chat-gpt-line-chat-bot && cd chat-gpt-line-chat-bot
```

プロジェクトルートで以下のコマンドを実行して Amplify を初期化します。

```
amplify init
```

プロジェクト名は `chatgptlinechatbot` とします。

```
Note: It is recommended to run this command from the root of your app directory
? Enter a name for the project (chatgptlinechatbot)
```

以下はフロントエンドの Amplify の設定です。

今回 Amplify はバックエンドのみで使用します。

フロントエンドはデフォルト設定で良いの Y を入力します。

```
The following configuration will be applied:

Project information
| Name: chatgptlinechatbot
| Environment: dev
| Default editor: Visual Studio Code
| App type: javascript
| Javascript framework: none
| Source Directory Path: src
| Distribution Directory Path: dist
| Build Command: npm run-script build
| Start Command: npm run-script start

? Initialize the project with the above configuration? (Y/n)
```

以下で amplify コマンドを実行する CLI ユーザーの profile を選択します。

```
? Select the authentication method you want to use:
❯ AWS profile
  AWS access keys
```

最後に、エラー発生時に　 Amplify CLI の改善の為 AWS にエラー内容を送信するか否かを選択します。

```
? Help improve Amplify CLI by sharing non sensitive configurations on failures (y/N) ›
```

# Amplify API カテゴリを追加する

Amplify で LINE チャットボットからのリクエストを受ける REST API を作成します。

具体的には Amplify CLI で API カテゴリを追加して API Gateway と Lambda を作成します。

プロジェクトルートで以下のコマンドを実行して Amplify API カテゴリを追加します。

```

amplify add api

```

REST を選択します。

```

$ amplify add api
? Select from one of the below mentioned services:
GraphQL
❯ REST

```

API 名を決めます。今回は `chatGPTLineChatBotRestApi` と命名しました。

```

? Provide a friendly name for your resource to be used as a label for this category in the project: › chatGPTLineChatBotRestApi

```

API のパスを指定します。 今回は `/v1/line/bot/message/reply` と指定しました。

```

? Provide a path (e.g., /book/{isbn}): › /v1/line/bot/message/reply

```

Lambda Function 名を決めます。今回は `chatGPTLineChatBotFunction` と命名しました。

```

Only one option for [Choose a Lambda source]. Selecting [Create a new Lambda function].
? Provide an AWS Lambda function name: chatGPTLineChatBotFunction

```

次に言語を選択します。

Python を選択します。

```

? Choose the runtime that you want to use:
.NET 6
Go
Java
NodeJS
❯ Python

```

詳細設定追加の有無を選択します。

Lambda Layers や Lambda の環境変数など作成できます。

こちらは後から追加するので N を入力します。

```

Available advanced settings:

- Resource access permissions
- Scheduled recurring invocation
- Lambda layers configuration
- Environment variables configuration
- Secret values configuration

? Do you want to configure advanced settings? (y/N)

```

以下 Y とするとエディタが開きます。

Lambda のコードは後から修正するので N を入力します。

```

? Do you want to edit the local lambda function now? N

```

API のアクセス制限、また追加の API Path 指定を問われますが、今回は両方 N を入力します。

今回 API のアクセス制限については Lambda で実装します。

詳しくは [LINE サーバー以外からのリクエストを弾く](#LINE-サーバー以外からのリクエストを弾く) を参照ください。

```

✔ Restrict API access? (Y/n) · no
✔ Do you want to add another path? (y/N) · no

```

Lambda Function が作成されます。

```

? Press enter to continue
Successfully added resource lineChatGPTBotDemoFunction locally.

Next steps:
Check out sample function code generated in <project-dir>/amplify/backend/function/lineChatGPTBotDemoFunction/src
"amplify function build" builds all of your functions currently in the project
"amplify mock function <functionName>" runs your function locally
To access AWS resources outside of this Amplify app, edit the /Users/kazuma/Documents/projects/chat-gpt/line-chat-gpt-bot-demo/amplify/backend/function/lineChatGPTBotDemoFunction/custom-policies.json
"amplify push" builds all of your local backend resources and provisions them in the cloud
"amplify publish" builds all of your local backend and front-end resources (if you added hosting category) and provisions them in the cloud
✅ Succesfully added the Lambda function locally

```

# DynamoDB 用の Amplify API カテゴリを追加する

今回のテーマとして Amplify のみでバックエンドを構築します。

DynamoDB を作成して Amplify で管理をする為、GraphQL(内部的には AppSync が作成される) の Amplify API カテゴリを追加します。

:::message alert
今回検証で DynamoDB を使いたいだけで GraphQL は使用しない為、無駄な AppSync リソースを作成する事になります。

本番環境などプロダクトに導入する際は、ちゃんと CloudFormation や CDK を利用して DynamoDB のみを構築してください！
:::

プロジェクトルートで以下のコマンドを実行して Amplify API カテゴリを追加します。

```

amplify add api

```

GraphQL を選択します。

```

$ amplify add api
? Select from one of the below mentioned services: (Use arrow keys)
❯ GraphQL
  REST

```

API 名を変更するので、Name を選択して Enter を押下します。

```

? Here is the GraphQL API that we will create. Select a setting to edit or continue
❯ Name: chatgptlinechatbot
  Authorization modes: API key (default, expiration time: 7 days from now)
  Conflict detection (required for DataStore): Disabled
  Continue

```

`chatGPTLineChatBotGraphQLApi` という API 名 を入力します。

```
? Provide API name: chatGPTLineChatBotGraphQLApi
```

次に Continue を選択します。

```
? Here is the GraphQL API that we will create. Select a setting to edit or continue (Use arrow keys)
  Name: chatGPTLineChatBotGraphQLApi
  Authorization modes: API key (default, expiration time: 7 days from now)
  Conflict detection (required for DataStore): Disabled
❯ Continue
```

Blank Schema を選択します。

```

? Choose a schema template: (Use arrow keys)
  Single object with fields (e.g., “Todo” with ID, name, description)
  One-to-many relationship (e.g., “Blogs” with “Posts” and “Comments”)
❯ Blank Schema

```

以下 Y を選択すると、エディタが開きます。

```
? Do you want to edit the schema now? (Y/n) ›
```

`schema.graphql` ファイルに以下スキーマを記述します。

これはユーザーの入力プロンプトと ChatGPT の推論結果を保存するテーブルです。

ユーザーと ChatGPT との会話を履歴として残すテーブルですね。

```graphql
type Messages @model {
  id: ID!
  lineUserId: String! @index(name: "byLineUserId", sortKeyFields: ["createdAt"])
  content: String!
  role: String!
  createdAt: AWSDateTime!
  updatedAt: AWSDateTime!
}
```

Lambda 実行時に lineUserId をリクエストから取得してテーブルで対して検索を実行します。

会話履歴の項目があればユーザーの入力プロンプトと一緒に ChatGPT に投げて、結果を取得します。

ユーザーの入力プロンプト + 会話履歴をまとめて ChatGPT に推論させる事により、会話履歴や文脈を読んだ回答が可能な訳ですね。

また簡単にスキーマの説明ですが、`@index` で `byLineUserId` という名前の GSI を作成します。

GSI を作成すると、lineUserId で DynamoDB に Query を投げる事が出来ます。

また、sortKeyFields に `createdAt` を指定する事により、Query 結果を作成日時でソートする事が出来ます。

# API カテゴリをクラウドに反映する

`amplify status` コマンドで現在のカテゴリの状態を確認します。

```
$ amplify status

    Current Environment: dev

┌──────────┬──────────────────────────────┬───────────┬───────────────────┐
│ Category │ Resource name                │ Operation │ Provider plugin   │
├──────────┼──────────────────────────────┼───────────┼───────────────────┤
│ Function │ chatGPTLineChatBotFunction   │ Create    │ awscloudformation │
├──────────┼──────────────────────────────┼───────────┼───────────────────┤
│ Api      │ chatGPTLineChatBotRestApi    │ Create    │ awscloudformation │
├──────────┼──────────────────────────────┼───────────┼───────────────────┤
│ Api      │ chatGPTLineChatBotGraphQLApi │ Create    │ awscloudformation │
└──────────┴──────────────────────────────┴───────────┴───────────────────┘

GraphQL transformer version: 2
```

AWS のクラウドに API カテゴリをプロビジョニングする為以下コマンドを実行します。

```

amplify push -y

```

コマンドを実行した所、開発環境に `pipenv` と `virtualenv` が無いので以下のエラーが発生しました。

```

You must have pipenv installed and available on your PATH as "pipenv". It can be installed by running "pip3 install --user pipenv".
You must have virtualenv installed and available on your PATH as "venv". It can be installed by running "pip3 install venv".
🛑 Missing required dependencies to package lineChatGPTBotDemoFunction

```

私の環境は macOS なので Homebrew でインストールしました。

ここは適宜環境に合わせてインストールしてください。

```

brew install pipenv
brew install virtualenv

```

改めて、 `amplify push -y` コマンドを実行します。

成功すると以下 GraphQL と REST API のエンドポイントが表示されます。

```

$ amplify status

    Current Environment: dev

┌──────────┬──────────────────────────────────┬───────────┬───────────────────┐
│ Category │ Resource name                    │ Operation │ Provider plugin   │
├──────────┼──────────────────────────────────┼───────────┼───────────────────┤
│ Function │ chatGPTLineChatBotFunction       │ No Change │ awscloudformation │
├──────────┼──────────────────────────────────┼───────────┼───────────────────┤
│ Api      │ chatGPTLineChatBotRestApi        │ No Change │ awscloudformation │
├──────────┼──────────────────────────────────┼───────────┼───────────────────┤
│ Api      │ chatGPTLineChatBotGraphQLApi     │ No Change │ awscloudformation │
└──────────┴──────────────────────────────────┴───────────┴───────────────────┘

GraphQL endpoint: https://XXXXXXXXXXXXXXXXXXX.appsync-api.ap-northeast-1.amazonaws.com/graphql
GraphQL API KEY: XXXXXXXXXXXXXXXXXXXXXXXX

GraphQL transformer version: 2
REST API endpoint: https://XXXXXXXXXXX.execute-api.ap-northeast-1.amazonaws.com/dev


```

REST API を実行しデフォルトのメッセージが表示される事を確認します。

```

$ curl https://XXXXXXXX.execute-api.ap-northeast-1.amazonaws.com/dev/v1/line/bot/message/reply
"Hello from your new Amplify Python lambda!"

```

# LINE Developers でチャネルアクセストークンとチャネルシークレットを取得する

[LIne Developers](https://developers.line.biz/ja/) のアカウント登録、ログインします。

プロバイダーの `作成` ボタンを押下します。

![](https://storage.googleapis.com/zenn-user-upload/638a5204960b-20230203.png)

任意のプロバイダー名を入力して `作成` ボタンを押下します。

![](https://storage.googleapis.com/zenn-user-upload/5ec75195c13f-20230203.png)

次に Messaging API を選択します。

![](https://storage.googleapis.com/zenn-user-upload/7db2c74e9043-20230203.png)

新規チャネル作成で任意のチャネル名、チャネル説明、その他入力必須項目を埋めます。

入力したら画面下部の `作成` ボタンをクリックします。

![](https://storage.googleapis.com/zenn-user-upload/b0e2955476d2-20230203.png)

チャネル基本設定タブの画面にあるチャネルシークレットを控えておきます。

チャネルシークレットは後ほど Lambda の LINE サーバのみアクセス許可をする実装の時に使います。

![](https://storage.googleapis.com/zenn-user-upload/5ef002c00448-20230203.png)

次にチャネルアクセストークンを発行します。

Messaging API 設定タブの画面にあるチャネルアクセストークン `発行` ボタンを押下します。

![](https://storage.googleapis.com/zenn-user-upload/c1d0bdaea069-20230203.png)

以下のようなアクセストークンが発行されるので控えておきます。

```

6/hBGi07nF5TPxiYasi5/XXXXXXXXXXXXXXXXXXXXX/fTatrGm8BCQSP9KAn4Reyd6mK7yGC8MkOlk32c5l0KSbn44aLPYoF11v01wRu4E+E8+ofqCIdlk3H3L/XXXXXXXXXXXXXXXXXX/1O/w1cDnyilFU=

```

次に Webhook を設定します。

Messaging API 設定タブの画面にある Webhook 設定の Webhook URL に先程 Amplify で生成した API エンドポイントを設定します。

![](https://storage.googleapis.com/zenn-user-upload/36133d9985f3-20230203.png)

Webhook の設定をしたら必ず `検証` ボタンを押下して疎通確認を行ってください。

![](https://storage.googleapis.com/zenn-user-upload/ab4a0b691e79-20230203.png)

疎通確認が問題なければ `Webhookの利用` を ON にします。

`Webhookの再送` は Webhook の送信に失敗した場合に、LINE サーバーから Webhook URL に Webhook を再送します。

リクエストが 2 回送信されるので、許容出来るのであれば ON にしても良いかと思いますが、今回は OFF にします。

![](https://storage.googleapis.com/zenn-user-upload/d4651b3f6c2c-20230203.png)

最後に自動応答メッセージを無効にします。

Messaging API 設定タブにある、LINE 公式アカウント機能 > 応答メッセージ の編集リンクをクリックします。

開いたページ内にある 詳細設定 > 応答メッセージ の設定をオフにします。

友達追加時のあいさつメッセージも、この画面から編集ができます。

![](https://storage.googleapis.com/zenn-user-upload/0afcfdf7e748-20230203.png)

# OpenAI の API キーを取得する

[OpenAI](https://platform.openai.com/overview) にアカウント登録、ログインをします。

`View API Keys` 画面を開きます。

![](https://storage.googleapis.com/zenn-user-upload/cdca9b414d72-20230203.png)

`Create New Secret Key` ボタンを押下し、API キーを取得します。

![](https://storage.googleapis.com/zenn-user-upload/ce10e3bb6cdf-20230203.png)

作成した API キーを控えておきます。

![](https://storage.googleapis.com/zenn-user-upload/47f067ba39fe-20230203.png)

# LINE 各種トークン、OpenAI API キーをシークレットを設定する

取得した LINE チャネルアクセストークン、 チェネルシークレット、OpenAI API キーを悪用されないよう、AWS Systems Manager のシークレットに登録し Lambda 関数から呼び出すようにします。

具体的には Amplify CLI から AWS Systems Manager Parameter Store にシークレットを保存、環境変数にキーを保存し Lambda 関数内から AWS SDK 経由にシークレットを取得できます。

以下コマンドを実行します。

```

amplify update function

```

シークレットを設定する Lambda 関数を指定します。

```

? Select the Lambda function you want to update chatGPTLineChatBotFunction

```

`Secret values configuration` を選択します。

```

? Which setting do you want to update?
Resource access permissions
Scheduled recurring invocation
Lambda layers configuration
Environment variables configuration
❯ Secret values configuration

```

LINE チャネルアクセストークンのキー名を入力します。

```

? Enter a secret name (this is the key used to look up the secret value): LINE_CHANNEL_ACCESS_TOKEN

```

キーの値を入力します。

```

? Enter the value for LINE_CHANNEL_ACCESS_TOKEN: [hidden]

```

続いて OpenAI の API キーを登録するので、 `Add a secret` を選択します。

```

? What do you want to do?
❯ Add a secret
Update a secret
Remove secrets
I'm done

```

API キー名を入力します。

```

? Enter a secret name (this is the key used to look up the secret value): OPEN_AI_API_KEY

```

キーの値を入力します。

```

? Enter the value for OPEN_AI_API_KEY: [hidden]

```

同じ要領で、LINE のチャネルシークレットも登録してください。

キー名は `LINE_CHANNEL_SECRET` で、値は先程 LINE Developers コンソールで控えておいた値です。

3 種類のキーの登録が終わったら作業を終了するので、 `I'm done` を選択、次の設問は Y を入力します。

```

? What do you want to do? (Use arrow keys)
Add a secret
Update a secret
Remove secrets
❯ I'm done
? This will immediately update secret values in the cloud. Do you want to continue? Yes
Use the AWS SSM GetParameter API to retrieve secrets in your Lambda function.
More information can be found here: https://docs.aws.amazon.com/systems-manager/latest/APIReference/API_GetParameter.html
? Do you want to edit the local lambda function now? No

```

この時点で、 AWS Systems Manager Parameter Store に指定したキーと値からシークレットが登録されます。

AWS コンソールを開いて、AWS Systems Manager > アプリケーション管理 > パラメータストア から確認できます。

![](https://storage.googleapis.com/zenn-user-upload/faeb27b3892d-20230203.png)

以下のパスを後から環境変数に登録するので控えておきます。

```

/amplify/{AppID}/dev/AMPLIFY_chatGPTLineChatBotFunction_{KeyName}

```

# シークレットのキーを環境変数に設定する

先程控えたシークレットのキー名を環境変数に設定します。

Lambda から環境変数に登録されたシークレットのキーを使って、AWS SDK 経由で AWS Systems Manager からシークレットの値を取得します。

以下コマンドを実行します。

```

amplify update function

```

シークレットを設定する Lambda 関数を指定します。

```

? Select the Lambda function you want to update chatGPTLineChatBotFunction

```

`Environment variables configuration` を選択します。

```

? Which setting do you want to update?
Resource access permissions
Scheduled recurring invocation
Lambda layers configuration
❯ Environment variables configuration
Secret values configuration

```

環境変数に登録するキー名を入力します。今回は `BASE_SECRET_PATH` と命名しました。

```

? Enter the environment variable name: BASE_SECRET_PATH

```

キーの値を入力します。

先程控えておいたシークレットのキー名の `{KeyName}` を省いた値を入力します。

`{AppID}` は自分の環境の値を入力します。

```

/amplify/{AppID}/dev/AMPLIFY_lineChatGPTBotDemoFunction_{KeyName}

```

```

? Enter the environment variable value: /amplify/XXXXXXXXXX/dev/AMPLIFY_lineChatGPTBotDemoFunction_

```

`I'm done` を選択し、次に N を入力し完了です。

```

? Select what you want to do with environment variables: (Use arrow keys)
Add new environment variable
Update existing environment variables
Remove existing environment variables
❯ I'm done
? Do you want to edit the local lambda function now? No

```

最後に `amplify push -y` を実行してクラウドに反映させてください。

# 外部ライブラリをインストールする

今回使用する外部ライブラリをインストールします。

Lambda Layers を使用して複数の Lambda で外部ライブラリを共有する方法もありますが、今回は Lambda が一つしかないので pipenv で個別にインストールします。

Lambda Function のルートディレクトリに移動します。

```

cd amplify/backend/function/lineChatGPTBotDemoFunction/

```

今回使う外部ライブラリをインストールします。

```

pipenv install boto3
pipenv install line-bot-sdk
pipenv install openai

```

# AWS Systems Manager のアクセス権限を設定する

AWS Systems Manager の パラメーターストアにアクセスできる権限が無いと API 実行時に以下エラーが発生します。

```
botocore.exceptions.ClientError: An error occurred (AccessDeniedException) when calling the GetParameters operation: User: arn:aws:iam::XXXXXXXXXXXX:user/ai-lab-amplify-cli-user is not authorized to perform: ssm:GetParameters on resource: arn:aws:ssm:ap-northeast-1:XXXXXXXXXXXX:parameter/OPEN_AI_API_KEY because no identity-based policy allows the ssm:GetParameters action
```

Lambda Function のルートディレクトリに `custom-policies.json` があるので、必要な権限を追加します。

以下パスのファイルを開きます。

`vi amplify/backend/function/lineChatGPTBotDemoFunction/custom-policies.json`

以下記述します。Resource に記述するアクセス可能なパラーメーターは複数作成可能なので、ワイルドカードで定義します。

```json
[
  {
    "Action": ["ssm:GetParameters", "ssm:GetParameter"],
    "Resource": ["arn:aws:ssm:ap-northeast-1:XXXXXXXXXXXX:parameter/*"]
  }
]
```

# DynamoDB のアクセス権限を設定する

これで実装の前準備が完了です。

後は [実装コード](#実装コード) を参照して Lambda を実装していきましょう。

# DynamoDB の疎通確認を行う

:::message alert
このパートは筆者が DynamoDB の疎通確認用 Lambda Function 作成作業履歴として残すものです。

通常こちらの作業は必要無いので読み飛ばしてください。
:::

`amplify add function` を実行します。

```
$ amplify add function
? Select which capability you want to add: (Use arrow keys)
❯ Lambda function (serverless function)
  Lambda layer (shared code & resource used across functions)
```

Lambda Function 名を決めます。今回は `checkDynamoDBConnectionFunction` と命名しました。

```
? Provide an AWS Lambda function name: checkDynamoDBConnectionFunction
```

言語は Python を選択します。

```
? Choose the runtime that you want to use:
  .NET 6
  Go
  Java
  NodeJS
❯ Python
```

次の設問は N を入力します。

```
Only one template found - using Hello World by default.

Available advanced settings:
- Resource access permissions
- Scheduled recurring invocation
- Lambda layers configuration
- Environment variables configuration
- Secret values configuration

? Do you want to configure advanced settings? (y/N)
```

最後の設問で Y を入力して Lambda Function の index.py を開きます。

```
? Do you want to edit the local lambda function now? Yes
```

コードを書く事前準備として Lambda のルートディレクトリに移動して boto3 をインストールします。

```
cd amplify/backend/function/checkDynamoDBConnectionFunction
pipenv install boto3
```

# おまけ

最後に、実装中にローカルで Lambda を実行して動作確認をする方法です。

amplify mock コマンドを使用してローカルで Lambda のテストができます。

実行方法は簡単で `amplify mock function` コマンドに Lambda 関数名を指定します。

```
amplify mock function lineChatGPTBotDemoFunction --event "src/event.json"
```

`event.json` は以下のような LINE サーバーから送られてくるリクエストの値とします。

https://github.com/kazuma-fujita/line-chat-gpt-bot-demo/blob/develop/amplify/backend/function/lineChatGPTBotDemoFunction/src/event.json

:::message
見やすくする為、実際のリクエストの値から改行や細かい値を修正して json 形式に整形しています。
:::

# 参考サイト

- [LINE の Webhook からのアクセスを”x-line-signature”を用いて検証する](https://poota.net/archives/362)
