---
title: "ChatGPT (GPT-4) API と AWS Amplify で会話履歴と文脈を読んで回答する LINE ボット を作る"
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

こんにちわ。 [ZUMA](https://twitter.com/zuma_lab) です。

日本時間 3/15 に OpenAI から ChatGPT の最新チャットボット AI である GPT-4 が公開されました。

筆者は GPT-4 API 利用申請をしていた為、3/18 に OpenAI から以下メールが送られてきて GPT-4 API の β 版が利用可能になりました。

![](https://storage.googleapis.com/zenn-user-upload/ea435a70328f-20230320.png)

以下 GPT-4 モデル仕様です。

https://platform.openai.com/docs/models/gpt-4

ChatGPT に翻訳、また AI モデル一覧をマークダウンのテーブル形式に直して貰いました。

> GPT-4 は、テキスト入力からテキスト出力を生成する大規模な多モーダルモデルで、将来的には画像入力も受け付ける予定です。
> GPT-4 は、広範な一般知識と高度な推論能力のおかげで、これまでのどのモデルよりも難しい問題を高い精度で解くことができます。
> gpt-3.5-turbo と同様に、GPT-4 はチャットに最適化されていますが、従来の補完タスクにもうまく対応します。

残念ながらまだ GPT-4 では画像の入力は受け付けていません。

但し、以下の表にある通り、最大トークン長が`gpt-4` モデルでは 8,192 トークン、 `gpt-4-32k` ではなんとその 4 倍である 32,768 トークンに拡大しています。

`gpt-3.5-turbo` モデルの最大トークン長が 4,096 トークンなので、それぞれ一度に 2 倍、8 倍のトークンが扱える事になります。

`gpt-4` 系のモデルでは `gpt-3.5-turbo` モデルと比較してより長い会話履歴や文脈を読んだチャットボットを作成する事ができます。

学習データについては `gpt-3.5-turbo` モデルと同様 2021 年 9 月までとなっています。

| 最新モデル     | 説明                                                                                                                                                           | 最大トークン数  | トレーニングデータ |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------- | ------------------ |
| gpt-4          | GPT-3.5 モデルよりも能力が高く、より複雑なタスクを実行でき、チャット用に最適化されている。最新モデルにアップデートされます。                                   | 8,192 トークン  | 2021 年 9 月まで   |
| gpt-4-0314     | 2023 年 3 月 14 日の GPT-4 のスナップショット。gpt-4 とは異なり、アップデートは受け取らず、2023 年 6 月 14 日に終了する 3 か月間だけサポートされます。         | 8,192 トークン  | 2021 年 9 月まで   |
| gpt-4-32k      | 基本の gpt-4 モードと同じ機能を持つが、コンテキスト長が 4 倍になっている。最新モデルにアップデートされます。                                                   | 32,768 トークン | 2021 年 9 月まで   |
| gpt-4-32k-0314 | 2023 年 3 月 14 日の gpt-4-32k のスナップショット。gpt-4-32k とは異なり、アップデートは受け取らず、2023 年 6 月 14 日に終了する 3 か月間だけサポートされます。 | 32,768 トークン | 2021 年 9 月まで   |

今回は `gpt-4` モデル を使って、会話履歴と文脈を読んで回答してくれる ChatGPT の LINE チャットボットを作ってみました。

会話の文脈を読むのは ChatGPT で普通に出来ることですが、ChatGPT API で自前で実装しようとすると少し工夫が必要だったので記事にしました。

# 成果物

このように会話履歴と文脈を読んで回答してくれます。

![](https://storage.googleapis.com/zenn-user-upload/f505e9bbb80a-20230320.png)

また、フレンドリーな性格で絵文字をたくさん使うようにしてます。

こちらは後述する role パラメーターで以下のキャラ設定をしています。

```
敬語を使うのをやめてください。友達のようにタメ口で話してください。また、絵文字をたくさん使って話してください。
```

![](https://storage.googleapis.com/zenn-user-upload/d26ee233248f-20230315.png)

前回の記事では `gpt-3.5-turbo` モデルを使用して LINE チャットボットを作成しました。

https://zenn.dev/zuma_lab/articles/chatgpt-line-chatbot

今回実装する `gpt-4` モデルと前回作成した `gpt-3.5-turbo` モデルで比較してみました。

まず、 `AIを全て「あ」から始まる文章で説明して。` という問を「あ」から「お」まで投げかけてみました。

期待する答えは、回答に含まれる文章の始まりが全て指定した文字となっている事です。

まず、 `gpt-3.5-turbo` モデルの回答です。

![](https://storage.googleapis.com/zenn-user-upload/3ff7fe22e0fe-20230320.png)

確かに回答に含まれている一番最初の文字は指定した文字から始まっていますが、文中の文章の始まりの文字はこちらが期待する結果とは大きく異なっています。

次に `gpt-4` モデルの回答です。

![](https://storage.googleapis.com/zenn-user-upload/4d461efed2d0-20230320.png)

すごいですね、回答に含まれている全ての文章の始まりが指定した文字となっています。

次に、会話履歴を以下の手順でテーブル作成の指示を与えてみました。

1. 最初のテーブル作成:

| 機種 |  カラー  | 金額 |
| :--: | :------: | :--: |
|  A   | ブラック | 1000 |
|  B   |  レッド  | 2000 |
|  C   | イエロー | 3000 |

2. テーブルに追加:

```
D ブルー 4000
E グリーン 5000
```

3. テーブルから削除:

```
機種 A B C D
```

4. テーブルに追加:

```
F ホワイト 6000
G ピンク 7000
```

まず、`gpt-3.5-turbo` モデルの回答です。

![](https://storage.googleapis.com/zenn-user-upload/5facf020cb06-20230320.png)

テーブル作成手順を実行した後に「今まで指示した内容を箇条書きにしてください」と聞いた所、テーブル作成の手順が間違えています。

最初に作成したテーブルのデータは 「A」「B」「C」です。

「D」と「E」のデータも最初に作成したテーブルに含まれてしまいました。

`gpt-3.5-turbo` モデルの最大トークン長が 4,096 トークンの為、プログラム側で入力プロンプトに含める会話履歴の数を制限しています。

その為、冒頭のテーブル作成手順が入力プロンプトに含まれず、誤った回答をしてしまいました。

さらに、「フレンドリーな性格で絵文字をたくさん使う」チャットボットの性格はもはや機能してません。

次に `gpt-4` モデルの回答です。

![](https://storage.googleapis.com/zenn-user-upload/eea69c67f43e-20230320.png)

`gpt-4` モデルでは、きちんと最初に作成したテーブルのデータが「A」「B」「C」であると回答しました。

`gpt-4` モデルの最大トークン長は 8,192 トークンの為、より多くの会話履歴を入力プロンプトに含める事ができます。

その為、冒頭のテーブル作成手順も会話履歴として入力プロンプトに含まれ、より正確な回答が出来るようになりました。

このように `gpt-4` モデルは拡大した最大トークン長により、長い会話履歴や文脈を読んだ回答が可能になります。

また、「フレンドリーな性格で絵文字をたくさん使う」チャットボットの性格をちゃんと守っています。

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

role には後述する ChatGPT API のパラメーターである role を保存します。

content にはユーザーと ChatGPT のメッセージを保存します。

# ChatGPT API の role について

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

LINE や OpenAI のトークン・キー取得、Amplify の構築など前準備が長くなってしまったので、先にコードを出します。

Python 初心者なので、ChatGPT にベースを書いてもらって後はググりながら書きました。

ChatGPT に実装を助けて貰ってるので私が記法など理解してなかったりします。

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
GPT_MODEL = 'gpt-4'

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
            model=GPT_MODEL,
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

`gpt-3.5-turbo` モデルは最大トークン長が 4096 トークン、`gpt-4` モデルでは 8,192 トークン、 `gpt-4-32k` ではなんとその 4 倍である 32,768 トークンに拡大しています。

今回は `gpt-4` モデルを使用している為、8,192 トークンまで扱えます。

前回作成した `gpt-3.5-turbo` モデルを使用した LINE ボットでは QUERY_LIMIT を 5 に設定しましたが、今回は 10 に設定しています。

QUERY_LIMIT を大きくしすぎるとトークン長が長くなりすぎエラーとなるので注意が必要です。

```py:message_repository.py
import uuid
from datetime import datetime

import chatgpt_api
import db_accessor
import message_repository

QUERY_LIMIT = 10


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
    chat_histories = message_repository._fetch_chat_histories_by_line_user_id(line_user_id, prompt_text)

    # Call the GPT3 API to get the completed text
    completed_text = chatgpt_api.completions(chat_histories)

    # Put a record of the user into the Messages table.
    message_repository._insert_message(line_user_id, 'user', prompt_text)

    # Put a record of the assistant into the Messages table.
    message_repository._insert_message(line_user_id, 'assistant', completed_text)

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

> LINE プラットフォームから送信される HTTP POST リクエストをボットではサーバーで受信したときは、ステータスコード 200 を返してください。
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

次から実装前準備の手順となるので、まずは以降手順の完了後コードを実装してください。

# Amplify CLI でリソースを作成する

:::message
前提として `amplify configure` で Amplify で使用する AWS リソースにアクセス可能な IAM ユーザーは作成済みとします。
:::

## Amplify を初期化する

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

## Amplify API カテゴリを追加する

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

## DynamoDB 用の Amplify API カテゴリを追加する

今回のテーマとして Amplify のみでバックエンドを構築します。

DynamoDB を作成して Amplify で管理をする為、GraphQL(内部的には AppSync が作成される) の Amplify API カテゴリを追加します。

:::message alert
今回検証で DynamoDB を使いたいだけで GraphQL は使用しない為、無駄な AppSync リソースを作成する事になります。

本番環境などプロダクトに導入する際は CloudFormation や CDK を利用して DynamoDB を構築する事も検討ください。
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

開かれた `schema.graphql` ファイルに以下スキーマを記述します。

`amplify push` を実行するとユーザーの入力プロンプトと ChatGPT の推論結果を保存する DynamoDB テーブルが作成されます。

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

簡単にスキーマの説明ですが、`@index` で `byLineUserId` という名前の GSI を作成します。

GSI を作成すると、lineUserId で DynamoDB に Query を投げる事が出来ます。

また、sortKeyFields に `createdAt` を指定する事により、Query 結果を作成日時でソートする事が出来ます。

## API カテゴリをクラウドに反映する

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

# LINE 各種トークン・OpenAI API キーを取得する

LINE Developers コンソールからチャネルアクセストークンとチャネルシークレットを取得し、OpenAI コンソールで OpenAI のリソースにアクセス出来る API キーを発行します。

## LINE Developers でチャネルアクセストークンとチャネルシークレットを取得する

[LIne Developers](https://developers.line.biz/ja/) のアカウント登録、ログインします。

プロバイダーの `作成` ボタンを押下します。

![](https://storage.googleapis.com/zenn-user-upload/638a5204960b-20230203.png)

任意のプロバイダー名を入力して `作成` ボタンを押下します。

※ 前回 GPT-3 の LINE チャットボットを作成した記事の画像を流用している為、ここでは GPT-3 としてますが正確には GPT-4 です。適宜読み替えて下さい。

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

## OpenAI の API キーを取得する

[OpenAI](https://platform.openai.com/overview) にアカウント登録、ログインをします。

`View API Keys` 画面を開きます。

![](https://storage.googleapis.com/zenn-user-upload/cdca9b414d72-20230203.png)

`Create New Secret Key` ボタンを押下し、API キーを取得します。

![](https://storage.googleapis.com/zenn-user-upload/ce10e3bb6cdf-20230203.png)

作成した API キーを控えておきます。

![](https://storage.googleapis.com/zenn-user-upload/47f067ba39fe-20230203.png)

# 取得した各種トークン、API キーを安全に扱う

取得した LINE チャネルアクセストークン、 チェネルシークレット、OpenAI API キーを悪用されないよう、AWS Systems Manager のシークレットに登録し Lambda 関数から呼び出すようにします。

具体的には Amplify CLI から AWS Systems Manager Parameter Store にシークレットを保存、環境変数にキーを保存し Lambda 関数内から AWS SDK 経由にシークレットを取得できます。

## LINE 各種トークン、OpenAI API キーをシークレットを設定する

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

以下のキー名の prefix (パス部分)を後から環境変数に登録するので控えておきます。

```

/amplify/{AppID}/dev/AMPLIFY_chatGPTLineChatBotFunction_{KeyName}

```

## シークレットのキー名 prefix と DynamoDB のテーブル名 postfix を Lambda の環境変数に設定する

先程控えたシークレットのキー名の prefix を Lambda の環境変数に設定します。

Lambda から環境変数に登録されたシークレットの prefix のキーを使って、AWS SDK 経由で AWS Systems Manager からシークレットの値を取得します。

また、`amplify push` 時に作成された DynamoDB のテーブル名には postfix の値が付与されています。

![](https://storage.googleapis.com/zenn-user-upload/c3ba8927dd01-20230315.png)

筆者の環境では既に複数の Amplify バックエンド環境があるのでテーブルが複数ありますが、通常なら `Messages-XXXXXXXXXXXXXXXXXX-dev` の 1 テーブルが作成されています。

テーブル名の postfix `-XXXXXXXXXXXXXXXXXX-dev` 部分を環境変数に設定して Lambda から呼び出して使用します。

この postfix は後述する開発環境、本番環境の切り替え時に参照するテーブルも自動的に切り替える為に使用します。

こちらの値は後ほど登録するので控えておいてください。

まずはシークレットのキー名 prefix を環境変数に設定します。

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

次に DynamoDB のテーブル名 postfix を環境変数に登録します。

`Add new environment variable` を選択します。

```
? Select what you want to do with environment variables: (Use arrow keys)
❯ Add new environment variable
Update existing environment variables
Remove existing environment variables
I'm done
? Do you want to edit the local lambda function now? No
```

`DB_TABLE_NAME_POSTFIX` と命名しました。

```
? Enter the environment variable name: DB_TABLE_NAME_POSTFIX
```

先程控えておいたテーブル名の postfix を入力します。

```
? Enter the environment variable value: -XXXXXXXXXXXXXXXXXXXXXX-dev
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

クラウドに反映させると Lambda コンソールの設定から作成した環境変数が確認できます。

![](https://storage.googleapis.com/zenn-user-upload/3fcbbefee3fd-20230315.png)

# 外部ライブラリをインストールする

今回使用する外部ライブラリをインストールします。

Lambda Layers を使用して複数の Lambda で外部ライブラリを共有する方法もありますが、今回は Lambda が一つしかないので pipenv で個別にインストールします。

Lambda Function のルートディレクトリに移動します。

```

cd amplify/backend/function/chatGPTLineChatBotFunction/

```

今回使う外部ライブラリをインストールします。

```

pipenv install boto3
pipenv install line-bot-sdk
pipenv install openai

```

# AWS Systems Manager と DynamoDB のアクセス権限を設定する

AWS Systems Manager の パラメーターストアにアクセスできる権限や DynamoDB の権限が無いと API 実行時に以下エラーが発生します。

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
    "Action": [
      "ssm:GetParameters",
      "ssm:GetParameter",
      "dynamodb:PutItem",
      "dynamodb:DeleteItem",
      "dynamodb:GetItem",
      "dynamodb:Scan",
      "dynamodb:Query",
      "dynamodb:UpdateItem"
    ],
    "Resource": [
      "arn:aws:ssm:ap-northeast-1:XXXXXXXXXXXX:parameter/*",
      "arn:aws:dynamodb:ap-northeast-1:XXXXXXXXXXXX:table/*"
    ]
  }
]
```

# Lambda の Timeout 値を延長する

Amplify で作成する Lambda の Timeout 値はデフォルトで 25 秒です。

ユーザーの入力プロンプトの内容や ChatGPT のアクセス状況次第ですが、ChatGPT API のレスポンスが 1 分を超える事があります。

時には 2〜3 分経ってレスポンスが返って来ることがあります。

Timeout 値が 25 秒だとタイムアウトエラーになって処理しきれない場合があるので以下 cloudformation-template.json を編集します。

```
vi amplify/backend/function/chatGPTLineChatBotFunction/chatGPTLineChatBotFunction-cloudformation-template.json
```

`Timeout` を好きな数値に変更します。

Lambda に設定出来る最大の Timeout 値は 900 秒（15 分）です。

筆者の場合、300 秒(5 分) に設定しています。

```json
        "Runtime": "python3.8",
        "Layers": [],
        "Timeout": 25 -> 300
```

これで実装の前準備が完了です。

後は [実装コード](#実装コード) を参照して Lambda を実装していきましょう。

# Amplify の開発環境と本番環境を切り替える

開発が進むにつれて Amplify のバックエンド開発環境と本番環境を切り替えたい場合が出てくると思います。

`amplify env` コマンドで簡単に開発環境と本番環境を切り替える事が出来ます。

:::message
前提として、これまでに作成した LINE チャットボットは開発環境とし、新たに本番環境を追加する事とします。

既に取得している LINE チャネルアクセストークン、 チェネルシークレット、OpenAI API キーとは別に、事前に本番環境用の各種トークン、API キーを作成してください。
:::

まずは `amplify env list` コマンドで現在の環境の状態を表示します。

```
$ amplify env list

| Environments |
| ------------ |
| *dev         |
```

`amplify add env` コマンドで本番環境を追加してみます。

以下のようにコマンドの引数に環境名を指定できます。

```
amplify add env prod
```

amplify コマンドを実行する CLI ユーザーの profile を選択します。

```
$ amplify add env prod
Note: It is recommended to run this command from the root of your app directory
Using default provider  awscloudformation
? Select the authentication method you want to use: (Use arrow keys)
❯ AWS profile
  AWS access keys
```

既に Lambda Function の環境変数を登録しているので、新しい環境に値を引き継ぐか追加するか選択します。

今回は開発環境と本番環境で環境変数を分けるので `Update environment variables now` を選択します。

```
? You have configured environment variables for functions. How do you want to proceed? …  (Use arrow keys or type to filter)
  Carry over existing environment variables to this new environment
❯ Update environment variables now
```

環境変数を追加する Lambda Function を選択します。

```
? Select the Lambda function you want to update values …  (Use arrow keys or type to filter)
❯ chatGPTLineChatBotFunction
  I'm done
```

追加する環境変数を選択します。

```
? Which function's environment variables do you want to edit? …  (Use arrow keys or type to filter)
  DB_TABLE_NAME_POSTFIX
❯ BASE_SECRET_PATH
  I'm done
```

既存の環境では `BASE_SECRET_PATH` に以下値を設定しています。

`{AppID}` は皆さんそれぞれの Amplify 固有の ID です。

```
/amplify/{AppID}/dev/AMPLIFY_chatGPTLineChatBotFunction_
```

パスに含まれている `dev` を 本番環境である `prod` に書き換えて上書き登録します。

```
? Enter the environment variable value: › /amplify/{AppID}/prod/AMPLIFY_chatGPTLineChatBotFunction_
```

環境変数の上書きが完了したので `I'm done` を選択して終了します。

```
? Which function's environment variables do you want to edit? …  (Use arrow keys or type to filter)
  BASE_SECRET_PATH
❯ I'm done
? Select the Lambda function you want to update values …  (Use arrow keys or type to filter)
  chatGPTLineChatBotFunction
❯ I'm done
```

次に、既に `AWS Systems Manager Parameter Store` に LINE チャネルアクセストークン、 チェネルシークレット、OpenAI API キーを登録しているので新たにシークレットを登録します。

新たに本番環境用の各種トークン、API キーのシークレットを追加します。

値を既存のシークレットから引き継ぐか更新するか聞かれるので `Update secret values now` を選択します。

```
? You have configured secrets for functions. How do you want to proceed?
  Carry over existing secret values to the new environment
❯ Update secret values now (you can always update secret values later using `amplify update function`)
```

先程同様、対象の Lambda Function を選択します。

```
? Select a function to update secrets for: (Use arrow keys)
❯ chatGPTLineChatBotFunction
  I'm done
```

`Update a secret` を選択します。

```
? What do you want to do?
  Add a secret
❯ Update a secret
  Remove secrets
  I'm done
```

追加するシークレットのキーを選択します。

```
? Select the secret to update: (Use arrow keys)
❯ LINE_CHANNEL_ACCESS_TOKEN
  OPEN_AI_API_KEY
  LINE_CHANNEL_SECRET
```

シークレットの値を入力します。

```
? Enter the value for LINE_CHANNEL_ACCESS_TOKEN: [hidden]
```

他のシークレットの追加するので、 `Update a secret` を選択します。

```
? What do you want to do? Update a secret
```

先程同様作業を繰り返し他のシークレットの値を入力していきます。

```
? Select the secret to update: OPEN_AI_API_KEY
? Enter the value for OPEN_AI_API_KEY: [hidden]
? What do you want to do? Update a secret
? Select the secret to update: LINE_CHANNEL_SECRET
? Enter the value for LINE_CHANNEL_SECRET: [hidden]
```

最後に `I'm done` で終了します。

```
? What do you want to do? (Use arrow keys)
  Add a secret
  Update a secret
  Remove secrets
❯ I'm done
? Select a function to update secrets for:
  chatGPTLineChatBotFunction
❯ I'm done

```

この時点で AWS Systems Manager Parameter Store を確認すると以下のように新たにシークレットが追加されています。

![](https://storage.googleapis.com/zenn-user-upload/bb9dd530a7c3-20230315.png)

また、 `amplify env list` コマンドで環境を確認すると `prod` 環境が追加されています。

```
$ amplify env list

| Environments |
| ------------ |
| dev          |
| *prod        |
```

`amplify status` で環境の状態が確認できます。

```
$ amplify status

    Current Environment: prod

┌──────────┬──────────────────────────────┬───────────┬───────────────────┐
│ Category │ Resource name                │ Operation │ Provider plugin   │
├──────────┼──────────────────────────────┼───────────┼───────────────────┤
│ Api      │ chatGPTLineChatBotGraphQLApi │ Create    │ awscloudformation │
├──────────┼──────────────────────────────┼───────────┼───────────────────┤
│ Api      │ chatGPTLineChatBotRestApi    │ Create    │ awscloudformation │
├──────────┼──────────────────────────────┼───────────┼───────────────────┤
│ Function │ chatGPTLineChatBotFunction   │ Create    │ awscloudformation │
└──────────┴──────────────────────────────┴───────────┴───────────────────┘

GraphQL transformer version: 2
```

忘れずに `amplify push -y` で AWS のクラウドに反映させましょう。

最後に amplify push が完了して作成されたリソースの中の DynamoDB のテーブル名 postfix を環境変数に登録します。

DynamoDB コンソールを確認してテーブル名の postfix である `-XXXXXXXXXXXXXXXXXXXXX-prod` を控えておきます。

![](https://storage.googleapis.com/zenn-user-upload/c3ba8927dd01-20230315.png)

`amplify update function` を実行して以下のように入力します。

```
$ amplify update function
? Select the Lambda function you want to update chatGPTLineChatBotFunction
? Which setting do you want to update? Environment variables configuration
? Select what you want to do with environment variables: Add new environment variable
? Enter the environment variable name: DB_TABLE_NAME_POSTFIX
? Enter the environment variable value: -XXXXXXXXXXXXXXXXXXXXX-prod
? Select what you want to do with environment variables: I'm done
? Do you want to edit the local lambda function now? No
```

:::message
環境変数 DB_TABLE_NAME_POSTFIX が存在している場合 `>> Key "DB_TABLE_NAME_POSTFIX" is already used` エラーが発生します。

また、既存の環境変数に `-XXXXXXXXXXXXXXXXXXXXX-prod` の値を更新しようとすると `-` がある為 `>> You can use the following characters: a-z A-Z 0-9 _` バリデーションが発生します。

その場合、一度環境変数 DB_TABLE_NAME_POSTFIX を削除してから改めて `Add new environment variable` で変数を追加するとバリデーションを通過し登録できます。

環境変数の新規追加の場合は `-` が登録出来ますが、更新時は何故か `-` が登録できないので注意が必要です。
:::

再度 `amplify push -y` コマンドを実行して AWS クラウドに反映させます。

クラウドに反映させると Lambda コンソールの設定から作成した環境変数が取得できます。

各環境変数の値に本番環境である `prod` という識別子がついていますね。

![](https://storage.googleapis.com/zenn-user-upload/c53a5a1f5b6f-20230315.png)

環境の移動は `amplify env checkout {envName}` コマンドを使用します。

開発時の作業は `amplify env checkout dev` で開発環境、本番反映の時は `amplify env checkout prod` で本番環境に切り替えて `amplify push` を実行しましょう。

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

また、`amplify mock` でローカルから DynamoDB を実行するするには以下 CLI の IAM ユーザーに DynamoDB へのアクセス権限を付与する必要があります。

今回はポリシーを作成して DynamoDB のアクセス権限をユーザーに紐付けました。

![](https://storage.googleapis.com/zenn-user-upload/7647829d4580-20230317.png)

![](https://storage.googleapis.com/zenn-user-upload/b6ffe0bffe1e-20230317.png)

![](https://storage.googleapis.com/zenn-user-upload/8b9cb394b6e5-20230317.png)

参考になれば幸いです。
