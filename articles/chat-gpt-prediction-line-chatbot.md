---
title: "ChatGPT で Bing Chat スタイルの質問提案をする LINE ボットを作る"
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

ChatGPT で Bing Chat のようにユーザーの質問予測をする LINE ボット を作ってみました。

ChatGPT の回答を元に ChatGPT がユーザーの質問予測をするので、脳死で ChatGPT に無限質問する事ができます。

# 成果物

以下成果物となります。

https://youtu.be/rS7qfRgu4f0

LINE の画面下部にユーザーが次にする質問予測をボタンで表示しています。

ボタンをタップするとその質問を ChatGPT に聞くことができます。

(デモは GPT-4 API を利用しているのでレスポンスが遅いです)

また、このように会話履歴と文脈を読んで回答をしてくれます。

ちなみに ChatGPT はフレンドリーで絵文字をやたら使うキャラ設定をしてるので少し画面がうるさいのはご了承ください。

![](https://storage.googleapis.com/zenn-user-upload/40bc779847c9-20230414.jpg =400x)
![](https://storage.googleapis.com/zenn-user-upload/63bc290236d9-20230414.jpg =400x)

画面下部に表示されているのが質問予測ボタンです。

ChatGPT の回答を元に ChatGPT がユーザーの質問を予測した結果を表示しています。

# 前提

前提として本記事は以下の LINE チャットボット作成記事をベースにしています。

https://zenn.dev/zuma_lab/articles/gpt-4-line-chatbot

バックエンド環境構築は[Amplify CLI でリソースを作成する](https://zenn.dev/zuma_lab/articles/gpt-4-line-chatbot#amplify-cli-%E3%81%A7%E3%83%AA%E3%82%BD%E3%83%BC%E3%82%B9%E3%82%92%E4%BD%9C%E6%88%90%E3%81%99%E3%82%8B)以降を参照ください。

Amplify の環境構築、REST API から DynamoDB の作成方法、 LINE 設定、OpenAI の API キー取得方法を解説しています。

また、DynamoDB を用いて会話や文脈を読んだ実装をしています。

まずは上記手順で LINE ボットのベース作成後、コードを実装してください。

# どうやってるの？

最初にアプローチを２つ考えました。

- ChatGPT の回答と質問予測をセットで出力するよう指示
- ChatGPT の回答を元に LangChain で質問予測を取得

LangChain の説明は省きますが、 LLM を使ってサービスを開発したいときのよくある機能をまとめて提供してくれているライブラリです。

2 パターンそれぞれ Pros/Cons を考えました。

| 方法                         | Pros                       | Cons                                                   |
| ---------------------------- | -------------------------- | ------------------------------------------------------ |
| 回答と質問予測をセットで出力 | 一度のリクエストで済む     | 質問予測を出力しない場合がある。消費トークン数が多い。 |
| LangChain で別々に出力       | 質問予測が確実に取得できる | 2 回リクエストが送信されるので回答レスポンスが遅い     |

それでは各パターンの解説です。

## 回答と質問予測をセットで出力するパターン

やり方としては単純で、システムプロンプトに以下のプロンプトを組み込んでいます。

```
フォーマルな言葉遣いはやめてください。友達のようにフレンドリーな口調で、話すときにはたくさんの絵文字を使ってください。
また、以下のフォーマットに従って回答してください。

# フォーマット

ユーザーの質問への回答

### Predictions ###
ユーザーの次の3つの質問を予測して、それぞれ20文字以内でリストアップ
```

質問予測は 3 つで 20 文字以内で回答するようにします。

なぜ 20 文字以内かというと、LINE に表示できる質問予測ボタンの最大文字数が 20 文字だからです。

試しに `日本食で安く食事ができるジャンルは何ですか？` と質問してみます。

ChatGPT からは以下のフォーマットで回答が返ってきます。

```
寿司やラーメン、天ぷら、焼き鳥などは、日本の居酒屋やレストランで比較的手軽に食べることができます。

### Predictions ###
1. どのくらいの値段ですか？
2. その中で高級なお店はどこですか？
3. おすすめのお店はありますか？
```

回答結果の `### Predictions ###` の識別子があったら以降の文字列を質問予測と見なしてボタンにセットします。

以下 プロダクトに組み込む Python コード全文となります。

:::details chatgpt_api.py

```py
import const
import openai
import re
from typing import List, Tuple, Dict

# Model name
GPT_MODEL = 'gpt-4'  # or gpt-3.5-turbo

# Temperature
TEMPERATURE = 0.7

PREDICTION_KEYWORD = '### Predictions ###'

SYSTEM_PROMPT = f'''フォーマルな言葉遣いはやめてください。友達のようにフレンドリーな口調で、話すときにはたくさんの絵文字を使ってください。
また、以下のフォーマットに従って回答してください。

# フォーマット

ユーザーの質問への回答

{PREDICTION_KEYWORD}
ユーザーの次の3つの質問を予測して、それぞれ20文字以内でリストアップ
'''

DEFAULT_PREDICTION_TEXT = f'''{PREDICTION_KEYWORD}
1. ChatGPTって何ですか？
2. ChatGPTでは何ができますか？
3. どんな質問に答えてくれますか？
'''

SYSTEM_PROMPTS = [{'role': 'system', 'content': SYSTEM_PROMPT}]


def _parse_completed_text(completed_text: str) -> Tuple[str, List[str]]:
    # キーワードで分割。キーワードは含まない。第2引数は分割数。
    completed_texts = completed_text.split(PREDICTION_KEYWORD, 1)
    if len(completed_texts) < 2:
        raise Exception('The keyword is not found in the text.')
    # ユーザーに返却するテキストを取得。文字列の前後の空白と改行を削除
    assistant_answer = completed_texts[0].strip()
    # キーワード以降のテキストを取得
    prediction_text = completed_texts[1]
    # 改行で分割し、文字列の前後の空白と改行を削除
    predictions = list(map(lambda line: _remove_ordinal_number(line.strip()), prediction_text.strip().split('\n')))
    return assistant_answer, predictions


# 文字列の先頭にある序数を削除する関数
def _remove_ordinal_number(text: str) -> str:
    # 正規表現で先頭の数字とピリオドを削除
    return re.sub(r'^\d+\.\s*', '', text)


def _print_total_length(completed_text, messages):
    join_message = completed_text + ' ' + ' '.join(map(lambda message: message['content'], messages))
    print(completed_text.replace('\n', ''))
    print(f"total length:{len(join_message)}")


def completions(history_prompts: List[Dict[str, str]]) -> Tuple[str, str, List[str]]:
    messages = SYSTEM_PROMPTS + history_prompts

    try:
        openai.api_key = const.OPEN_AI_API_KEY
        response = openai.ChatCompletion.create(
            model=GPT_MODEL,
            messages=messages,
            temperature=TEMPERATURE,
        )
        completed_text = response['choices'][0]['message']['content']

        if PREDICTION_KEYWORD not in completed_text:
            completed_text = f'{completed_text}\n{DEFAULT_PREDICTION_TEXT}'

        _print_total_length(completed_text, messages)

        assistant_answer, predictions = _parse_completed_text(completed_text)

        return completed_text, assistant_answer, predictions
    except Exception as e:
        raise e
```

:::

### Pros

メリットは一度のリクエストで回答と質問予測が得られるのでユーザーへの回答速度が上がります。

レスポンスが遅い GPT-4 API でも質問予測を取得したい場合はこちらのやり方が良さそうです。

### Cons

このやり方の欠点としては、今回のように会話履歴と文脈を読んだ回答をさせているケースでは、過去の会話履歴にも全て質問予測を含める必要があります。

ChatGPT は過去の会話履歴で無い文脈については答えてくれません。

過去会話に `### Predictions ###` の識別子が無ければ `以下のフォーマットに従って回答してください。` という指示は無視されます。

よって、Predictions の識別子と質問予測を全て質問毎の会話履歴に入れ込まないといけないのでトークン数が増えていきます。

会話履歴を含めない QA ボットであれば気にしなくていいですが、文脈を読むボットの場合はコストが高くなっていきます。

ちなみに、システムプロンプトの指示を英語にした場合も動作が不安定です。

なぜなら回答が日本語なので、 `### Predictions ###` が `### 予測 ###` 等翻訳されてしまい、動作が不安定になります。

また、今後登場する AI モデルによっても挙動が変わる可能性があります。

## LangChain で回答と質問予測を別々に出力するパターン

LangChain の Memory in Sequential Chains を利用します。

https://python.langchain.com/en/latest/modules/chains/generic/sequential_chains.html#memory-in-sequential-chains

まず過去の会話履歴のテンプレートを用意します。

次に回答用 LLM と質問予測用 LLM を作成します。

質問予測用 LLM は回答用 LLM の出力を入力として質問予測します。

LangChain の Chains という仕組みを使って、回答用 LLM -> 質問予測用 LLM と実行順番を決めます。

今回は SequentialChain の memory 機能を使って会話履歴を読み込んで回答用 LLM のプロンプトに埋め込んでいます。

以下 Google Colab 上で動くコードです。

```
!pip install langchain
!pip install openai
```

まず、過去の会話履歴テンプレートを設定します。

こちらは Colab 用で会話履歴を直書きしてますが、実際は DynamoDB から取得します。

AI と Human の過去の会話を SimpleMemory に設定する事により文脈を読んだ回答をします。

```py
from langchain.memory import SimpleMemory

OPENAI_API_KEY='Your API Key'

GPT_MODEL='gpt-3.5-turbo'
### 会話履歴
history = """Human: 食べ物の中で何が好き？
AI: 私は食べ物を食べることはできませんが、日本の文化に興味を持っており、寿司やラーメン、天ぷら、焼き鳥などの日本料理が好きです。また、インドカレーやタイ料理、メキシコ料理など、多様な国の料理も好きです。
Human: その中で何が好き？
AI: 私はおすすめの料理を選ぶことができませんが、日本の寿司は非常に人気があり、新鮮な魚介類を使用していることが多いため、おすすめです。また、ラーメンは日本のソウルフードの一つで、様々な種類がありますが、豚骨ラーメンや醤油ラーメンが特に人気があります。天ぷらも、サクサクとした食感が美味しいです。焼き鳥は、串に刺した鶏肉を炭火で焼いたもので、ビールとの相性が良いです。
"""

memories = {"history": history}
memory = SimpleMemory(memories=memories)
```

次に ChatGPT の回答用 LLM の作成です。

question_template でチャットボットの性格、会話履歴のプロンプトを作成します。

プロンプトの精度を高める為に英語で質問して最後に `answer in Japanese language` と日本語で出力させます。

ChatOpenAI の temperature はデフォルトの 0.7 を設定していますが、ここはお好みで変更してください。

```py
from langchain import PromptTemplate, LLMChain
from langchain.chat_models import ChatOpenAI

### 質問
# question_template訳
# 以下は、人間とAIの友好的な会話です。
# AIはおしゃべりで、その文脈からたくさんの具体的な詳細を提供します。
# それから、フォーマルな言葉を使うのをやめて、友達のように親しみやすく話してください。また、たくさんの絵文字を使って話してください。
# AIが質問に答えられない場合は、正直にわからないと言います。
# 日本語でお答えください。

question_template = """The following is a friendly conversation between a human and an AI.
The AI is talkative and provides lots of specific details from its context.
After that, stop using formal language. Talk to me in a friendly way, like a friend. Also, use lots of emojis when you talk.
If the AI does not know the answer to a question, it truthfully says it does not know.
Please answer in Japanese language.

Current conversation:
{history}
Human: {question}
AI:"""

question_prompt_template = PromptTemplate(input_variables=["question", "history"], template=question_template)
question_llm = ChatOpenAI(model_name=GPT_MODEL, temperature=0.7, openai_api_key=OPENAI_API_KEY)
question_chain = LLMChain(llm=question_llm, prompt=question_prompt_template, output_key="answer")
```

次に質問予測用 LLM の作成です。

prediction_template で回答結果をコンテキストとしてセット、質問予測の指示プロンプトを作成します。

```py
### 質問予測

# prediction_template訳
# 次の3つの質問を予測してください。ユーザーが与えられた文脈に応じてそれぞれ20文字以内で質問します。日本語で答えてください。
prediction_template = """Predict the next 3 questions the user will ask in response to the given context, each within 20 characters. Please answer in Japanese language.

### Context ###
{answer}
"""
prediction_prompt_template = PromptTemplate(input_variables=["answer"], template=prediction_template)
prediction_llm = ChatOpenAI(model_name=GPT_MODEL, temperature=0.7, openai_api_key=OPENAI_API_KEY)
prediction_chain = LLMChain(llm=prediction_llm, prompt=prediction_prompt_template, output_key="prediction")
```

最後に SequentialChain の memory に会話履歴を設定、回答用 LLM -> 質問予測用 LLM の順に設定し実行します。

```py
from langchain.chains import SequentialChain

### 推論実行
question_answer_prediction_chain = SequentialChain(
    memory=memory,
    chains=[question_chain, prediction_chain],
    input_variables=["question"],
    output_variables=["answer", "prediction"],
    verbose=True)
result = question_answer_prediction_chain({"question": "値段は？"})
result
```

以下推論結果となります。

```
> Entering new SequentialChain chain...

> Finished chain.
{'question': '値段は？',
 'history': 'Human: 食べ物の中で何が好き？\nAI: 私は食べ物を食べることはできませんが、日本の文化に興味を持っており、寿司やラーメン、天ぷら、焼き鳥などの日本料理が好きです。また、インドカレーやタイ料理、メキシコ料理など、多様な国の料理も好きです。\nHuman: その中で何が好き？\nAI: 私はおすすめの料理を選ぶことができませんが、日本の寿司は非常に人気があり、新鮮な魚介類を使用していることが多いため、おすすめです。また、ラーメンは日本のソウルフードの一つで、様々な種類がありますが、豚骨ラーメンや醤油ラーメンが特に人気があります。天ぷらも、サクサクとした食感が美味しいです。焼き鳥は、串に刺した鶏肉を炭火で焼いたもので、ビールとの相性が良いです。\n',
 'answer': 'それぞれの料理によって値段は異なりますが、寿司やラーメン、天ぷら、焼き鳥などは、日本の居酒屋やレストランで比較的手軽に食べることができます。また、高級なお店では、より高価な料理もあります。ただし、日本の食文化では、値段が高いからといって必ずしも美味しいとは限りません。地元の人が通うような、味の良いお店を探すのがおすすめです。🍣🍜🍢🍗💰',
 'prediction': '1. どのくらいの値段ですか？\n2. どこで探せばいいですか？\n3. 何がおすすめですか？'}
```

回答結果が返却されます。

```
'answer': 'それぞれの料理によって値段は異なりますが、寿司やラーメン、天ぷら、焼き鳥などは、日本の居酒屋やレストランで比較的手軽に食べることができます。また、高級なお店では、より高価な料理もあります。ただし、日本の食文化では、値段が高いからといって必ずしも美味しいとは限りません。地元の人が通うような、味の良いお店を探すのがおすすめです。🍣🍜🍢🍗💰'
```

質問予測も返却されるのでこちらを LINE の質問予測ボタンにセットします。

```
'prediction': '1. どのくらいの値段ですか？\n2. どこで探せばいいですか？\n3. 何がおすすめですか？'
```

以下 プロダクトに組み込む Python コード全文となります。

:::details langchain_api.py

```py
import re
from typing import Dict, List, Tuple

import const
from langchain import LLMChain, PromptTemplate
from langchain.chains import SequentialChain
from langchain.chat_models import ChatOpenAI
from langchain.memory import SimpleMemory

GPT_MODEL = 'gpt-3.5-turbo'
TEMPERATURE = 0.7

# question_template訳
# 以下は、人間とAIの友好的な会話です。
# AIはおしゃべりで、その文脈からたくさんの具体的な詳細を提供します。
# それから、フォーマルな言葉を使うのをやめて、友達のように親しみやすく話してください。また、たくさんの絵文字を使って話してください。
# AIが質問に答えられない場合は、正直にわからないと言います。
# 日本語でお答えください。

QUESTION_TEMPLATE = """The following is a friendly conversation between a human and an AI.
The AI is talkative and provides lots of specific details from its context.
After that, stop using formal language. Talk to me in a friendly way, like a friend. Also, use lots of emojis when you talk.
If the AI does not know the answer to a question, it truthfully says it does not know.
Please answer in Japanese language.

Current conversation:
{history}
Human: {question}
AI:"""

# prediction_template訳
# 次の3つの質問を予測してください。ユーザーが与えられた文脈に応じてそれぞれ20文字以内で質問します。日本語で答えてください。
PREDICTION_TEMPLATE = """Predict the next 3 questions the user will ask in response to the given context, each within 20 characters. Please answer in Japanese language.

### Context ###
{answer}
"""


def create_simple_memory(history):
    memories = {"history": history}
    memory = SimpleMemory(memories=memories)
    return memory


def create_question_chain():
    question_prompt_template = PromptTemplate(input_variables=["question", "history"], template=QUESTION_TEMPLATE)
    question_llm = ChatOpenAI(model_name=GPT_MODEL, temperature=TEMPERATURE, openai_api_key=const.OPEN_AI_API_KEY)
    question_chain = LLMChain(llm=question_llm, prompt=question_prompt_template, output_key="answer")
    return question_chain


def create_prediction_chain():
    prediction_prompt_template = PromptTemplate(input_variables=["answer"], template=PREDICTION_TEMPLATE)
    prediction_llm = ChatOpenAI(model_name=GPT_MODEL, temperature=TEMPERATURE, openai_api_key=const.OPEN_AI_API_KEY)
    prediction_chain = LLMChain(llm=prediction_llm, prompt=prediction_prompt_template, output_key="prediction")
    return prediction_chain


def create_question_answer_prediction_chain(memory, question_chain, prediction_chain):
    question_answer_prediction_chain = SequentialChain(
        memory=memory,
        chains=[question_chain, prediction_chain],
        input_variables=["question"],
        output_variables=["answer", "prediction"],
        verbose=True)
    return question_answer_prediction_chain


def parse_history_prompts(messages: List[Dict[str, str]]) -> Tuple[str, str]:
    formatted_messages = []
    question = ""

    for i, message in enumerate(messages):
        if message["role"] == "user":
            role = "Human"
        elif message["role"] == "assistant":
            role = "AI"
        formatted_messages.append(f"{role}: {message['content']}")

        # 入力配列の最後の行のcontentをquestion変数に格納
        if i == len(messages) - 1 and message["role"] == "user":
            question = message["content"]

    # 文字列に結合
    history = "\n".join(formatted_messages)
    return history, question


def _extract_prediction_and_answer(data: Dict[str, str]) -> Tuple[List[str], str]:
    prediction = data["prediction"]
    assistant_answer = data["answer"]

    # 改行で分割し、文字列の前後の空白と改行を削除
    predictions = list(map(lambda line: _remove_ordinal_number(line.strip()), prediction.strip().split('\n')))
    return assistant_answer, predictions


# 文字列の先頭にある序数を削除する関数
def _remove_ordinal_number(text: str) -> str:
    # 正規表現で先頭の数字とピリオドを削除
    return re.sub(r'^\d+\.\s*', '', text)


def completions(history_prompts: List[Dict[str, str]]) -> Tuple[str, str, List[str]]:
    history, question = parse_history_prompts(history_prompts)
    memory = create_simple_memory(history)
    question_chain = create_question_chain()
    prediction_chain = create_prediction_chain()
    question_answer_prediction_chain = create_question_answer_prediction_chain(memory, question_chain, prediction_chain)
    result = question_answer_prediction_chain({"question": question})
    assistant_answer, predictions = _extract_prediction_and_answer(result)
    return assistant_answer, predictions
```

:::

### Pros

回答 LLM 実行後、回答結果から質問予測用 LLM に推論させるので確実に質問予測を取得する事ができます。

### Cons

質問に対する推論と、質問予測の推論 2 回 ChatGPT API にリクエストが投げられるので、レスポンスが遅いです。

実際に GPT-4 API で実行した所 Timeout エラーとなる事がありました。

# LINE のタイムラインに質問予測ボタンを設定する

LINE のタイムラインに質問予測ボタンを設置するには LINE SDK の QuickReplyButton を使います。

QuickReplyButton に質問予測を設定するのですが、最大文字数 20 文字なのでセットする前に文字数チェックをします。

QuickReply ブジェクトと推論結果を TextSendMessage に追加して LineBotApi の reply_message で LINE サーバに返却すれば質問予測ボタンが表示されます。

```py
from typing import List

import const
from linebot import LineBotApi
from linebot.models import (MessageAction, QuickReply, QuickReplyButton,
                            TextSendMessage)


def reply_message_for_line(reply_token: str, assistant_answer: str, predictions: List[str]):
    try:
        # Create an instance of the LineBotApi with the Line channel access token
        line_bot_api = LineBotApi(const.LINE_CHANNEL_ACCESS_TOKEN)
        # predications 配列内の要素の文字列が21文字以上の場合配列から削除
        predictions = list(filter(lambda line: len(line) <= 20 and line != "", predictions))
        # predications が空だったら message に TextSendMessage を設定
        if len(predictions) == 0:
            message = TextSendMessage(text=assistant_answer)
        else:
            # クイックリプライアクションを作成
            quick_reply_actions = map(lambda line: QuickReplyButton(action=MessageAction(label=line, text=line)), predictions)
            # クイックリプライオブジェクトを作成
            quick_reply = QuickReply(items=quick_reply_actions)
            # テキストメッセージにクイックリプライを追加
            message = TextSendMessage(text=assistant_answer, quick_reply=quick_reply)

        # Reply the message using the LineBotApi instance
        line_bot_api.reply_message(reply_token, message)

    except Exception as e:
        raise e
```

その他、DynamoDB 操作等のコードは [こちら](https://zenn.dev/zuma_lab/articles/gpt-4-line-chatbot#%E5%AE%9F%E8%A3%85%E3%82%B3%E3%83%BC%E3%83%89) に掲載しております。

参考になれば幸いです。
