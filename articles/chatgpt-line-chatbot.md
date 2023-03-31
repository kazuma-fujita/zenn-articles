---
title: "ChatGPT API と AWS Amplify で会話履歴と文脈を読んで回答する LINE ボット を作る"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - "ChatGPT"
  - "OpenAI"
  - "LINE"
  - "AWS"
  - "Amplify"
published: true
---

:::message
3/18 から GPT-4 API waitlist に登録しているユーザー向けに GPT-4 API の β 版が利用可能になりました。
GPT-4 API を使用した LINE チャットボットの作り方については以下記事を参照ください。

https://zenn.dev/zuma_lab/articles/gpt-4-line-chatbot
:::

こんにちわ。 [ZUMA](https://twitter.com/zuma_lab) です。

3/15 に OpenAI から GPT-4 が公開され、Google からも PaLM が発表されたり毎日が目まぐるしいですね。

OpenAI から現在公開されている最新のチャットモデル API は `gpt-3.5-turbo` です。

さて、今回は `gpt-3.5-turbo` モデル を使って、会話履歴と文脈を読んで回答してくれる ChatGPT の LINE チャットボットを作ってみました。

会話の文脈を読むのは ChatGPT で普通に出来ることですが、ChatGPT API で自前で実装しようとすると少し工夫が必要だったので記事にしました。

ちなみにブログ記事の内容も一部 ChatGPT(GPT-4) に書いて貰っています。

# 成果物

以下成果物です。

https://youtu.be/7PEmEJv6L7U

このように会話履歴と文脈を読んで回答してくれます。

![](https://storage.googleapis.com/zenn-user-upload/f9542a70ce9b-20230317.png)

![](https://storage.googleapis.com/zenn-user-upload/e62fa4eeb8f5-20230317.png)

性格もフレンドリーでタメ口の口調で絵文字をいっぱい使うようにしてます。

![](https://storage.googleapis.com/zenn-user-upload/d26ee233248f-20230315.png)

もちろんマジメに聞けばマジメに答えてくれます。

![](https://storage.googleapis.com/zenn-user-upload/f48e657174cc-20230315.png)

# バックエンド環境について

今回バックエンド全て AWS Amplify を使用します。

具体的には Amplify で REST API と Lambda、 DynamoDB を作成します。

LINE からの Request を受ける Lambda でユーザーからのメッセージを取得、ChatGPT API で推論実行、メッセージを会話履歴として DynamoDB に保存をします。

Amplify CLI で全てのリソースが作成出来るので、環境を捨てるのも簡単です。

更には Amplify CLI でバックエンドの開発環境、本番環境を切り分ける方法も解説します。

# DynamoDB テーブル構成について

会話履歴を保存する DynamoDB の属性名とデータ型は以下となります。

- Messages テーブル

| プライマリーキー | GSI          | 属性名     | 属性データ型 |
| ---------------- | ------------ | ---------- | ------------ |
| id               | -            | ID         | String       |
| -                | byLineUserId | lineUserId | String       |
| -                | -            | content    | String       |
| -                | -            | role       | String       |
| -                | -            | createdAt  | String       |
| -                | -            | updatedAt  | String       |

複数ユーザーが LINE ボットを使用した時に会話履歴が混在しないようにする必要があります。

ユーザー毎にユニークな lineUserId に対し GSI（グローバルセカンダリインデックス）を追加し byLineUserId というインデックス名に設定しています。

この GSI を用いて lineUserId で DynamoDB に対するクエリ検索を行うことで高速に検索が可能かつ、複数ユーザーが使用しても会話履歴が混在しないようにします。

ソートキーには createdAt が含まれており、クエリの結果は作成日時でソートされます。

パーティションキーとソートキーをテーブルで表現すると以下になります。

| パーティションキー | ソートキー | 属性                                            |
| ------------------ | ---------- | ----------------------------------------------- |
| id                 | -          | lineUserId, content, role, createdAt, updatedAt |
| lineUserId (GSI)   | createdAt  | id, content, role, updatedAt                    |

id には プログラムから発行したユニークな uuid を保存します。

role には後述する gpt-3.5-turbo API のパラメーターである role を保存します。

content にはユーザーと ChatGPT のメッセージを保存します。

# gpt-3.5-turbo api の role について

role とは、チャットモデルの入力と出力に影響するパラメーターです。

role を設定することで、チャットモデルがどのような役割を担って会話するかを制御できます。

role は messages パラメーターに含まれる各メッセージオブジェクトに設定する必要があります。

messages パラメーターは、チャットモデルに渡す会話文の履歴を表す配列です。

各メッセージオブジェクトは text と role を持ちます。

role は以下の 3 種類があります。

- system: チャットモデル自身の役割です。チャットモデルの性格や前提条件などを設定します。
- user: チャットモデルと会話する人間の役割です。メッセージを入力してチャットモデルに送信します。
- assistant: AI アシスタントの役割です。user からのメッセージに対して情報を提供します。

今回は system ロールで AI モデルがフレンドリーな性格で絵文字を多めに使うように設定します。

また、user と assistant ロールのメッセージを DynamoDB に保存し、ChatGPT API 実行前に会話履歴テーブルからメッセージを取得、会話履歴として入力プロンプトに含めます。

実際に DynamoDB には以下のように user と assistant ロールが交互に保存されます。

![](https://storage.googleapis.com/zenn-user-upload/7185a83eace4-20230317.png)

会話履歴を ChatGPT API の入力プロンプトに含めることにより、文脈を読んだ推論が可能になります。

また、複数ユーザーが使っても会話履歴が混在しないように DynamoDB を設計しています。

# 実行環境

- macOS Ventura 13.1（22C65）
- Amplify 10.8.1
- pipenv version 2022.12.19
- virtualenv 20.17.1

# 実装コード

本記事では Lambda の実装コードのみを掲載します。

バックエンド環境構築は以下記事の[Amplify CLI でリソースを作成する](https://zenn.dev/zuma_lab/articles/gpt-4-line-chatbot#amplify-cli-%E3%81%A7%E3%83%AA%E3%82%BD%E3%83%BC%E3%82%B9%E3%82%92%E4%BD%9C%E6%88%90%E3%81%99%E3%82%8B)以降を参照ください。

https://zenn.dev/zuma_lab/articles/gpt-4-line-chatbot#amplify-cli-%E3%81%A7%E3%83%AA%E3%82%BD%E3%83%BC%E3%82%B9%E3%82%92%E4%BD%9C%E6%88%90%E3%81%99%E3%82%8B

Amplify の環境構築、REST API から DynamoDB の作成方法、 LINE 設定、OpenAI の API キー取得方法を解説しています。

まずは上記手順で LINE ボットのベース作成後、コードを実装してください。

ちなみに筆者は Python 初心者なので、コードは ChatGPT にベースを書いてもらって後はググりながら書きました。

ChatGPT に助けて貰ってるので私が記法など理解してなかったりします。

誤っている箇所あればマサカリお願いします。

## ディレクトリ構成

Lambda Function の ディレクトリ構成は以下となります。

```
amplify/backend/function/chatGPTLineChatBotFunction
├── Pipfile
├── Pipfile.lock
├── amplify.state
├── chatGPTLineChatBotFunction-cloudformation-template.json
├── custom-policies.json
├── function-parameters.json
└── src
    ├── aws_systems_manager.py
    ├── chatgpt_api.py
    ├── const.py
    ├── db_accessor.py
    ├── event.json
    ├── guard.py
    ├── index.py
    ├── line_api.py
    ├── line_request_body_parser.py
    ├── message_repository.py
    └── setup.py
```

## OpenAI Completions API 実装

OpenAI の Completions API を実行するコードです。

system ロールでモデルの性格を設定しています。

content は英語の方が精度が高いので英語で設定しています。

```py:chatgpt_api.py
import openai
import const

# Model name
GPT3_MODEL = 'gpt-3.5-turbo'

# Maximum number of tokens to generate
MAX_TOKENS = 1024

# Create a new dict list of a system
# SYSTEM_PROMPTS = [{'role': 'system', 'content': '敬語を使うのをやめてください。友達のようにタメ口で話してください。また、絵文字をたくさん使って話してください。'}]
SYSTEM_PROMPTS = [{'role': 'system', 'content': 'Please stop using polite language. Talk to me in a friendly way like a friend. Also, use a lot of emojis when you talk.'}]


def completions(history_prompts):
    messages = SYSTEM_PROMPTS + history_prompts

    print(f"prompts:{messages}")
    try:
        openai.api_key = const.OPEN_AI_API_KEY
        response = openai.ChatCompletion.create(
            model=GPT3_MODEL,
            messages=messages,
            max_tokens=MAX_TOKENS
        )
        return response['choices'][0]['message']['content']
    except Exception as e:
        # Raise the exception
        raise e
```

今回はトークンが長くなりすぎないように設定パラメーターは `max_tokens` のみ設定しています。

その他細かい Completions API のパラメーターについては OpenAI の [こちら](https://platform.openai.com/docs/api-reference/completions/create) を参照ください。

## LINE API 実装

次に LINE Bot に ChatGPT API のレスポンスを返却する実装です。

ここは [LINE 公式のライブラリ](https://github.com/line/line-bot-sdk-python) を使用しました。

```py:line_api.py
import const
from linebot import LineBotApi
from linebot.models import TextSendMessage

def reply_message_for_line(reply_token, completed_text):
    try:
        # Create an instance of the LineBotApi with the Line channel access token
        line_bot_api = LineBotApi(const.LINE_CHANNEL_ACCESS_TOKEN)

        # Reply the message using the LineBotApi instance
        line_bot_api.reply_message(reply_token, TextSendMessage(text=completed_text))

    except Exception as e:
        # Raise the exception
        raise e
```

## DynamoDB 実装

DynamoDB へのアクセスは boto3 を利用します。

会話履歴を登録する PUT 関数、会話履歴を取得する QUERY 関数を作成しています。

```py:db_accessor.py
import boto3
from datetime import datetime
import const

TABLE_NAME = f'Messages{const.DB_TABLE_NAME_POSTFIX}'
QUERY_INDEX_NAME = 'byLineUserId'

dynamodb = boto3.client('dynamodb')


def query_by_line_user_id(line_user_id: str, limit: int) -> list:
    # Create a dictionary of query parameters
    query_params = {
        'TableName': TABLE_NAME,
        'IndexName': QUERY_INDEX_NAME,
        # Use a named parameter for the key condition expression
        'KeyConditionExpression': '#lineUserId = :lineUserId',
        # Define an expression attribute name for the hash key
        'ExpressionAttributeNames': {
            '#lineUserId': 'lineUserId'
        },
        # Define an expression attribute value for the hash key
        'ExpressionAttributeValues': {
            ':lineUserId': {'S': line_user_id}
        },
        # Sort the results in descending order by sort key
        'ScanIndexForward': False,
        # Limit the number of results
        'Limit': limit
    }

    try:
        # Call the query method of the DynamoDB client with the query parameters
        query_result = dynamodb.query(**query_params)
        # Return the list of items from the query result
        return query_result['Items']
    except Exception as e:
        # Raise any exception that occurs during the query operation
        raise e


def put_message(partition_key: str, uid: str, role: str, content: str, now: datetime) -> None:
    # Create a dictionary of options for put_item
    options = {
        'TableName': TABLE_NAME,
        'Item': {
            'id': {'S': partition_key},
            'lineUserId': {'S': uid},
            'role': {'S': role},
            'content': {'S': content},
            'createdAt': {'S': now.isoformat()},
        },
    }
    # Try to put the item into the table using dynamodb client
    try:
        dynamodb.put_item(**options)

    # If an exception occurs, re-raise it
    except Exception as e:
        raise e
```

## 会話データ取得・登録・ChatGPT 推論実装

会話履歴テーブルからデータの取得、登録、ChatGPT API の推論実行〜推論結果取得をする関数です。

QUERY_LIMIT 定数の値は ChatGPT API の入力プロンプトに含める会話履歴数です。

この値を大きくするとより多くの user と assistant の会話履歴を入力プロンプトに含める事ができます。

ただし、gpt-3.5-turbo モデルは最大トークン長が 4096 なので、QUERY_LIMIT を大きくしすぎるとトークン長が長くなりすぎエラーとなるので注意が必要です。

```py:message_repository.py
import uuid
from datetime import datetime

import chatgpt_api
import db_accessor

QUERY_LIMIT = 5


def _fetch_chat_histories_by_line_user_id(line_user_id, prompt_text):
    try:
        if line_user_id is None:
            raise Exception('To query an element is none.')

        # Query messages by Line user ID.
        db_results = db_accessor.query_by_line_user_id(line_user_id, QUERY_LIMIT)

        # Reverse messages
        reserved_results = list(reversed(db_results))

        # Create new dict list of a prompt
        chat_histories = list(map(lambda item: {"role": item["role"]["S"], "content": item["content"]["S"]}, reserved_results))
        # Create the list of a current user prompt
        current_prompts = [{"role": "user", "content": prompt_text}]

        # Join the lists
        return chat_histories + current_prompts

    except Exception as e:
        # Raise the exception
        raise e


def _insert_message(line_user_id, role, prompt_text):
    try:
        if prompt_text is None or role is None or line_user_id is None:
            raise Exception('To insert elements are none.')

        # Create a partition key
        partition_key = str(uuid.uuid4())

        # Put a record of the user into the Messages table.
        db_accessor.put_message(partition_key, line_user_id, role, prompt_text, datetime.now())

    except Exception as e:
        # Raise the exception
        raise e


def create_completed_text(line_user_id, prompt_text):
    # Query messages by Line user ID.
    chat_histories = _fetch_chat_histories_by_line_user_id(line_user_id, prompt_text)

    # Call the GPT3 API to get the completed text
    completed_text = chatgpt_api.completions(chat_histories)

    # Put a record of the user into the Messages table.
    _insert_message(line_user_id, 'user', prompt_text)

    # Put a record of the assistant into the Messages table.
    _insert_message(line_user_id, 'assistant', completed_text)

    return completed_text
```

このトークン数の調整はトークン長カウンター、tokeniser である [tiktoken](https://github.com/openai/tiktoken) を使えば出来そうです。

今回はこのトークン長オーバーのエラーハンドリングは省いていますが、この処理も別で記事にしたいと思います。

## LINE サーバー以外からのリクエストを弾く

このままだと Lambda の API が丸裸なので、外部から実行される可能性があります。

外部からリクエストが来ても LINE のリプライトークンが一致していない限り ChatBot に返信は出来ないのですが、そもそも LINE サーバー以外からのリクエストはエラーとしましょう。

LINE Developers の設定で Webhook のサーバー IP アドレスを指定する事が出来きます。

ですが Amplify のみで Lambda の IP アドレスの固定化は難しそうだったので今回は LINE サーバーからリクエストヘッダーで送られくる `x-line-signature` を検証する形にします。

LINE Developers の公式で [署名の検証の方法](https://developers.line.biz/ja/reference/messaging-api/#signature-validation) が記載されています。

以下実装コードです。

```py:guard.py
import hashlib
import hmac
import base64
import const


def verify_request(event):
    x_line_signature = event["headers"].get("x-line-signature") or event["headers"].get("X-Line-Signature")
    body = event["body"]

    # Generate the signature using HMAC-SHA256
    hash = hmac.new(const.LINE_CHANNEL_SECRET.encode('utf-8'), body.encode('utf-8'), hashlib.sha256).digest()
    signature = base64.b64encode(hash)

    # Compare the signature from the request headers with the generated signature
    if signature != x_line_signature.encode():
        raise Exception("Request verification failed. Request came from a non-LINE server source.")

```

## LINE トークン ・ OpenAI API キーをシークレットから取得する

LINE のチャネルトークンやシークレットトークン、OpenAI の API キーなど秘匿情報は AWS Systems Manager Parameter Store に登録して隠蔽します。

AWS Systems Manager のシークレットの登録方法は後述しますので、シークレットの値を取得するコードを先に出します。

boto3 で簡単に AWS System Manager から値を取得することが可能です。

```py:aws_systems_manager.py
import boto3
from botocore.exceptions import ClientError

# AWS region name
AWS_REGION = "ap-northeast-1"


def get_secret(secret_key):
    try:
        # Create a client for AWS Systems Manager
        ssm = boto3.client('ssm', region_name=AWS_REGION)

        # Get the secret from AWS SSM
        response = ssm.get_parameter(
            Name=secret_key,
            WithDecryption=True
        )

        # Return the value of the secret
        return response['Parameter']['Value']
    except ClientError as e:
        raise e
```

実装したモジュールを利用して、以下のように定数として取得出来るようにします。

```py:const.py
import os
import aws_systems_manager

BASE_SECRET_PATH = os.environ.get('BASE_SECRET_PATH')
DB_TABLE_NAME_POSTFIX = os.environ.get('DB_TABLE_NAME_POSTFIX')
OPEN_AI_API_KEY = aws_systems_manager.get_secret(f'{BASE_SECRET_PATH}OPEN_AI_API_KEY')
LINE_CHANNEL_SECRET = aws_systems_manager.get_secret(f'{BASE_SECRET_PATH}LINE_CHANNEL_SECRET')
LINE_CHANNEL_ACCESS_TOKEN = aws_systems_manager.get_secret(f'{BASE_SECRET_PATH}LINE_CHANNEL_ACCESS_TOKEN')
```

## Lambda の index.py 実装

最後に Lambda から最初に呼び出される index.py を実装します。

ここでは LINE サーバーからのリクエスト処理、ChatGPT の推論、推論結果を LINE サーバーに返却しています。

また、先程実装した LINE サーバー以外のリクエストを弾くモジュールを呼び出しています。

```py:index.py
import json

import guard
import line_api
import line_request_body_parser
import message_repository


def handler(event, context):
    try:
        # Verify if the request is valid
        guard.verify_request(event)

        # Parse the event body as a JSON object
        event_body = json.loads(event['body'])
        prompt_text = line_request_body_parser.get_prompt_text(event_body)
        line_user_id = line_request_body_parser.get_line_user_id(event_body)
        reply_token = line_request_body_parser.get_reply_token(event_body)
        # Check if the event is a message type and is of text type
        if prompt_text is None or line_user_id is None or reply_token is None:
            raise Exception('Elements of the event body are not found.')

        print(prompt_text.replace('\n', ''))

        # Create the completed text by Chat-GPT 3.5 turbo
        completed_text = message_repository.create_completed_text(line_user_id, prompt_text)
        # Reply the message using the LineBotApi instance
        line_api.reply_message_for_line(reply_token, completed_text)

    except Exception as e:
        # Log the error
        print(e)

        # Return 200 even when an error occurs as mentioned in Line API documentation
        # https://developers.line.biz/ja/reference/messaging-api/#response
        return {'statusCode': 200, 'body': json.dumps(f'Exception occurred: {e}')}

    # Return a success message if the reply was sent successfully
    return {'statusCode': 200, 'body': json.dumps('Reply ended normally.')}
```

ポイントとして、エラー時にもステータスコード 200 を返すようにしています。

[LINE Developers のレスポンス項目](https://developers.line.biz/ja/reference/messaging-api/#response) に以下記載があります。

> LINE プラットフォームから送信される HTTP POST リクエストをボットサーバーで受信したときは、ステータスコード 200 を返してください。
> LINE プラットフォームから疎通確認のために、Webhook イベントが含まれない HTTP POST リクエストが送信されることがあります。

実際に 200 を返却しないと、例えば後述する LINE チャネルトークンを取得する際の手順で、LINE プラットフォームから Webhook の疎通確認をする箇所があります。

疎通確認の際にリクエスト中に Webhook イベントが無い為、実装上エラーとなるのですが、その際に 200 を返却しないと疎通確認が成功しません。

また、LINE ChatBOT で質問を入力した際に LINE サーバーから送られてくるリクエストは `event` で渡されますが、リクエストの値は以下のようになっています。

https://github.com/kazuma-fujita/line-chat-gpt-bot-demo/blob/develop/amplify/backend/function/lineChatGPTBotDemoFunction/src/event.json

上記は見やすくする為、改行や細かい値を修正して json 形式に整形しているのですが、実際の LINE サーバーのリクエストに含まれる body の値は文字列になっていたりします。

LINE サーバーからのリレエスト処理実装時のポイントとして、body の値は `json.loads(event['body'])` として json 形式にパースしてから受信メッセージやリプライトークンを取得する必要があります。

パースした body から以下のようにしてユーザーのメッセージ、LINE ユーザー ID、LINE サーバーに返答する時に使用する replyToken が取得できます。

```py:line_request_body_parser.py
def get_prompt_text(event_body):
    if event_body['events'][0]['type'] == 'message' and event_body['events'][0]['message']['type'] == 'text':
        return event_body['events'][0]['message']['text']
    return None


def get_line_user_id(event_body):
    if event_body['events'][0]['source'] and event_body['events'][0]['source']['type'] == 'user':
        return event_body['events'][0]['source']['userId']
    return None


def get_reply_token(event_body):
    if event_body['events'][0]['replyToken']:
        return event_body['events'][0]['replyToken']
    return None
```

以上、実装コードでした。

# バックエンド環境構築

バックエンド環境構築は以下記事の[Amplify CLI でリソースを作成する](https://zenn.dev/zuma_lab/articles/gpt-4-line-chatbot#amplify-cli-%E3%81%A7%E3%83%AA%E3%82%BD%E3%83%BC%E3%82%B9%E3%82%92%E4%BD%9C%E6%88%90%E3%81%99%E3%82%8B)以降を参照ください。

https://zenn.dev/zuma_lab/articles/gpt-4-line-chatbot#amplify-cli-%E3%81%A7%E3%83%AA%E3%82%BD%E3%83%BC%E3%82%B9%E3%82%92%E4%BD%9C%E6%88%90%E3%81%99%E3%82%8B

Amplify の環境構築、REST API から DynamoDB の作成方法、 LINE 設定、OpenAI の API キー取得方法を解説しています。

上記の手順で LINE ボットのベース作成後にコードを実装してください。

以上、参考になれば幸いです。
