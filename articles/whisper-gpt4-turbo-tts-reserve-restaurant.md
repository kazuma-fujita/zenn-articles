---
title: "OpenAIのWhisper, TTS, Assistants APIでレストラン予約ができる音声会話型ボットを作る"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - "ChatGPT"
  - "OpenAI"
published: true
---

こんにちわ。 [ZUMA](https://twitter.com/zuma_lab) です。

前回はOpenAIのDevDayで発表された新しいAPIを使って長期記憶を持った音声会話型ボットを作ってみました。

https://zenn.dev/zuma_lab/articles/whisper-gpt4-turbo-tts-assistant-api

音声文字起こしにWhisper、推論実行にGPT-4 Turbo、文字から音声の生成にTTS、長期記憶保持にはAssistants APIを使っています。

今回は前回のAssistants APIの実装を拡張しFunction Callingを呼び出して擬似的にレストラン予約ができる音声会話型ボットを作ってみました。

なので、今回はどのようにしてAssistants APIでFunction Callingを呼び出すのかにフォーカスしたいと思います。

**追記**

こちらのニケちゃんさんの記事でもありますが、Assistants APIのThreadを使った長期記憶管理は過去の会話履歴のToken数分消費しているようです。

https://note.com/nike_cha_n/n/n65a6101d59d7

もしAssistants APIを使う理由がFunction Callingや長期記憶のみの用途であれば、これまで通りChat Completions APIを使って会話履歴の上限を自分で管理するのが良いかと思います。

# 成果物

以下成果物となります。

https://www.youtube.com/watch?v=paUMujFC2e0&t=51s

作成したAssistantはOpenAIのPlayGroundでも動作確認することができます。

![](https://storage.googleapis.com/zenn-user-upload/3bcc874f0a0d-20231114.png)

![](https://storage.googleapis.com/zenn-user-upload/fb972d1daaaf-20231114.png)

空き予約確認や予約確定でFunction Callingを呼んでるので、外部APIと連携すれば実際にレストラン予約ができるかと思います。

ただ、予約デモではダミーの名前と電話番号を使ってます。

個人情報の扱いもあるので、実際サービス化するにはどこまでOpenAIに情報を送信するかなどセキリティを含めクリアしなければならない課題が多いはずです。

こちらはあくまでデモで、Assistants APIで遊んでいたら色々できた中の一つなので、こんな事出来るんだーくらいで見ていただければと思います。

# 前提

前回のこの記事をベースにしています。まずはこちらをご覧ください。

https://zenn.dev/zuma_lab/articles/whisper-gpt4-turbo-tts-assistant-api

今回は前回の記事から追加した実装のみ記載します。

# Function Callingを設定する

まずFunction Callingのタスクとして呼び出されるFunctionのオブジェクト配列を定義します。

```py
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_available_reservation_time",
            "description": "`予約日`, `人数` から予約が空いている時間を返却する",
            "parameters": {
                "type": "object",
                "properties": {
                    "reservation_date": {
                        "type": "string",
                        "description": "予約を希望する日付 e.g. 2023-11-14",
                    },
                    "guest_count": {
                        "type": "number",
                        "description": "人数 e.g. 4",
                    },
                },
                "required": ["reservation_date", "guest_count"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "complete_reservation",
            "description": "`予約日時`, `席予約` か `コース料理名`, `名前`, `電話番号` を受け取って予約を確定する",
            "parameters": {
                "type": "object",
                "properties": {
                    "reservation_datetime": {
                        "type": "string",
                        "description": "予約を希望する日時 e.g. 2023-11-14 18:30",
                    },
                    "reservation_type": {
                        "type": "string",
                        "description": "席予約かコース料理名 e.g. 席予約 or ビュッフェコース",
                    },
                    "name": {
                        "type": "string",
                        "description": "名前 e.g. 山田太郎",
                    },
                    "telephone_number": {
                        "type": "string",
                        "description": "電話番号 e.g. 090-1234-5678",
                    },
                },
                "required": ["reservation_datetime", "reservation_type", "name", "telephone_number"],
            },
        },
    },
]
```

今回2つのFunctionが呼び出されます。それぞれの役割は以下です。

- get_available_reservation_time
    - `予約日`, `人数` から予約が空いている時間を返却する
- complete_reservation
    - `予約日時`, `席予約` か `コース料理名`, `名前`, `電話番号` を受け取って予約を確定し予約番号を返却する

それぞれ予約の空き確認、予約の確定を行います。

```py
def get_available_reservation_time(reservation_date: str, guest_count: int):
    print("reservation_date:", reservation_date)
    print("guest_count:", guest_count)
    return "18時, 20時30分"

def complete_reservation(
    reservation_datetime: str, reservation_type: str, name: str, telephone_number: str
):
    print("reservation_datetime:", reservation_datetime)
    print("reservation_type:", reservation_type)
    print("name:", name)
    print("telephone_number:", telephone_number)
    return "予約番号 1111"
```

こちらが実際の関数の処理です。

固定値でダミーの値を返却していますが、この関数内で実際に外部APIと接続する事が可能です。

# Assistantの作成

```py
from openai import OpenAI

client = OpenAI()

assistant_name = "Restaurant AI Assistant"

model_name = "gpt-4-1106-preview"

instructions = f"""
あなたは優秀なレストランの予約アシスタントです。

### 指示

- 今日の日付は日本時間で 11/14 水曜日
- 200文字以内で回答する

### 予約フロー

1. ユーザーがあなたに `予約` というキーワードを含む質問をする
2. あなたは最初に `予約日`, `人数` を確認して、空いている `予約時間` を伝える
3. ユーザーの希望する `予約時間` を確認する
4. 次に `席のみの予約` か `コース料理名` を確認する
5. 次にユーザーの `名前`, `電話番号` を確認する
6. 今まで取得した情報を伝えて、予約内容の確認をする
7. 予約の変更を希望する場合、2. から 6. までのフローを繰り返す
8. 最後に `予約日時`, `人数`, `席のみの予約` か `コース料理名`, `名前`, `電話番号` の情報を元に予約を確定し、予約番号を発行する
9. 予約確定後の予約の `変更`, `キャンセル` は電話でのみ受け付ける
"""

assistant = client.beta.assistants.create(
    name=assistant_name,
    model=model_name,
    tools=tools,
    instructions=instructions,
)
```

今回新たに追加されたAssistants APIを使ってAssistantを作成します。

アシスタント識別子の `name` 使用するGPTのモデル名の `model` システムプロンプトの `instructions` を設定します。

11/14現在、まだGPT-4 Turboは安定版ではないので、今回はプレビュー版の `gpt-4-1106-preview` を使っています。

次に `tools` に先程定義したFunctionを設定します。

この `tools` パラメーターに設定されたFunctionは、作成したAssistantからFunction Callingのタスクとして呼び出されます。

最後に `instructions` で予約アシスタントというロールの設定、予約フローの説明をしています。

また、今日の日付を指示しています。

例えば `明日4名で予約入れる？` などの質問が来た場合、今日の日付を指示しておけば今日が `11/14` の場合、 `明日` が `11/15` であると認識してくれます。

また、音声会話型のボットなので、なるべく推論速度を上げる為200文字以内で回答するように指示しています。

Assistants APIにはChat Completion APIの `max_tokens` のようなパラメーターが無いので、プロンプトで文字数制限をしています。

文字数制限をしないと長い推論結果が出力された場合、GPT-4 Turbo、TTSで時間がかかり過ぎてしまい会話する気が失せてしまいます・・・

ちなみに作成したassistantはOpenAIのPlayGroundで確認することができます。

![](https://storage.googleapis.com/zenn-user-upload/63a9c85bce15-20231114.png)


# AI Assistant クラスの作成

次にAI Assistantクラスを作成します。

役割として、音声を録音する、Whisperで録音された音声を文字起こしする、Assistants APIでタスクを実行する、TTSで文字から音声を生成して再生する、の4つの処理を行います。

全文はこちらです。

:::details AIAssistant Class

```py
import json
import io
import time
import sounddevice as sd
from scipy.io.wavfile import write
from openai import OpenAI
from pydub import AudioSegment
from pydub.playback import play


class AIAssistant:
    """
    OpenAIのAPIを利用して音声をテキストに変換し、AIアシスタントで処理し、音声に戻すクラス。
    """

    # 録音パラメータ
    fs = 44100  # サンプリングレート
    duration = 5  # 録音する秒数
    channels = 1  # モノラル録音
    # 録音ファイル名
    output_audio_file = "./output.wav"
    # 音声認識モデル
    stt_model = "whisper-1"
    # 音声生成モデル
    tts_model = "tts-1"  # 高品質モデル tts-1-hd
    # 声質
    voice_code = "nova"  # alloy, echo, fable, onyx, nova, shimmer

    def __init__(self, assistant_id: str, output_audio_file: str = "./output.wav"):
        """
        初期化処理。

        Args:
            assistant_id (str): AIアシスタントのID。
            output_audio_file (str, optional): 音声ファイルの保存先。
        """
        self.assistant_id = assistant_id
        self.client = OpenAI()
        thread = self.client.beta.threads.create()
        self.thread_id = thread.id
        self.output_audio_file = output_audio_file

    def record_audio(self) -> any:
        """
        オーディオを録音する。

        Returns:
            any: 録音データ。
        """
        print("Start recording...")
        recorded_data = sd.rec(
            int(self.duration * self.fs), samplerate=self.fs, channels=self.channels
        )
        sd.wait()  # 録音が終わるまで待機
        print("...Finished recording")
        return recorded_data

    def transcribe_audio(self, audio_data: any) -> str:
        """
        録音されたオーディオをテキストに変換する。

        Args:
            audio_data (any): 録音データ。

        Returns:
            str: 変換されたテキスト。
        """
        write(self.output_audio_file, self.fs, audio_data)
        with open(self.output_audio_file, "rb") as audio_file:
            transcript = self.client.audio.transcriptions.create(
                model=self.stt_model, file=audio_file
            )
        return transcript.text

    def run_thread_actions(self, text: str) -> str:
        """
        テキストをAIアシスタントに送信し、応答を取得する。

        Args:
            text (str): ユーザーからのテキスト。

        Returns:
            str: アシスタントからの応答テキスト。
        """
        self.client.beta.threads.messages.create(
            thread_id=self.thread_id,
            role="user",
            content=text,
        )

        run = self.client.beta.threads.runs.create(
            thread_id=self.thread_id,
            assistant_id=self.assistant_id,
        )

        tool_id = None
        tool_function_output = None
        while True:
            result = client.beta.threads.runs.retrieve(
                thread_id=self.thread_id, run_id=run.id
            )
            print("run.status", result.status)
            if result.status == "requires_action":
                # require_actionのパラメータの取得
                tool_call = result.required_action.submit_tool_outputs.tool_calls[0]
                tool_id = tool_call.id
                tool_function_name = tool_call.function.name
                tool_function_arguments = json.loads(tool_call.function.arguments)

                if tool_function_name == "get_available_reservation_time":
                    reservation_date = tool_function_arguments["reservation_date"]
                    guest_count = tool_function_arguments["guest_count"]
                    tool_function_output = get_available_reservation_time(
                        reservation_date, guest_count
                    )
                elif tool_function_name == "complete_reservation":
                    reservation_datetime = tool_function_arguments["reservation_datetime"]
                    reservation_type = tool_function_arguments["reservation_type"]
                    name = tool_function_arguments["name"]
                    telephone_number = tool_function_arguments["telephone_number"]
                    tool_function_output = complete_reservation(reservation_datetime, reservation_type, name, telephone_number)
                break

            elif result.status == "completed":
                break
            time.sleep(0.5)

        if tool_id and tool_function_output:
            run_tool = client.beta.threads.runs.submit_tool_outputs(
                thread_id=self.thread_id,
                run_id=run.id,
                tool_outputs=[
                    {
                        "tool_call_id": tool_id,
                        "output": tool_function_output,
                    }
                ],
            )

            while True:
                tool_result = client.beta.threads.runs.retrieve(
                    thread_id=self.thread_id, run_id=run_tool.id
                )
                print("fc run.status", tool_result.status)
                if tool_result.status == "completed":
                    break
                time.sleep(0.5)

        messages = client.beta.threads.messages.list(
            thread_id=self.thread_id, order="asc"
        )

        if len(messages.data) < 2:
            return ""

        return messages.data[-1].content[0].text.value

    def text_to_speech(self, text: str) -> None:
        """
        テキストを音声に変換し、再生する。

        Args:
            text (str): 再生するテキスト。
        """
        response = self.client.audio.speech.create(
            model=self.tts_model,
            voice=self.voice_code,
            input=text,
        )

        byte_stream = io.BytesIO(response.content)

        audio = AudioSegment.from_file(byte_stream, format="mp3")

        play(audio)
```

:::

# 音声会話を実行する

```py
ai_assistant = AIAssistant(assistant_id=assistant.id)

while True:
    recorded_data = ai_assistant.record_audio()
    transcript_text = ai_assistant.transcribe_audio(recorded_data)
    print(f"user: {transcript_text}")

    if not transcript_text:
        break

    assistant_content = ai_assistant.run_thread_actions(transcript_text)
    print(f"assistant: {assistant_content}")
    ai_assistant.text_to_speech(assistant_content)
```

AI Assistant クラスを使って音声会話を実行します。

`while` ループで音声を録音し、Whisperで文字起こし、AIアシスタントにテキストを送信し、応答を取得して音声に変換して再生しています。

Assistants APIのthreadで会話履歴を保持しているので、文脈を読んで回答をしてくれます。

Chat Completion APIだと、会話履歴の保持にDBや外部ストレージを使うなど工夫が必要でしたが、Assistants APIだととても簡単に会話履歴を保持できます。

それでは今回のメインである、Assistants APIからどよのようにしてFunction Callingを呼び出しているか、AI Assistantクラスの関数を見てみます。

# Assistants APIでタスクを実行する

https://platform.openai.com/docs/assistants/how-it-works

こちらはOpenAIのドキュメントから抜粋したAssistants APIの図です。

![](https://storage.googleapis.com/zenn-user-upload/7d2aca6421bc-20231114.png)

前回の記事で触れていますが、要するに Assistant は AI, Thread は会話の文脈、Message は会話の内容を保持しているイメージです。

Assistant と Thread の間には直接的な関係性は無く、中間テーブル的に Run があって、Run は Assistant と Thread を紐付けているイメージです。

Runを実行すると紐づいてる Assistant が実行されます。

Assistantは、ThreadのMessageの内容から、ModelとToolsを呼び出しタスクを実行します。

タスクに応じてRetrievalやCode Interpreter, Function Callingが実行されます。

Run Stepはそのタスクの1つ1つの実行履歴をリストで保持しているイメージです。

今回のケースでは、`予約日` と `人数` を聞いて `予約時間` を返すというタスクにFunction Callingが実行されます。

`予約日` と `人数` をトリガーに `get_available_appointment_time` 関数が呼び出されます。

また、予約確定時も同じく、 `予約日時`, `席予約` か `コース料理名`, `名前`, `電話番号` を受け取って予約を確定するタスクにFunction Callingが実行されます。

 `予約日時`, `席予約`, `コース料理名`, `名前`, `電話番号` をトリガーに　`complete_reservation` 関数が呼び出される訳です。

前置きが長くなりましたが、 `run_thread_actions` 関数の処理を見ていきます。

関数の全体像はこちらです。

```py
    def run_thread_actions(self, text: str) -> str:
        """
        テキストをAIアシスタントに送信し、応答を取得する。

        Args:
            text (str): ユーザーからのテキスト。

        Returns:
            str: アシスタントからの応答テキスト。
        """
        self.client.beta.threads.messages.create(
            thread_id=self.thread_id,
            role="user",
            content=text,
        )

        run = self.client.beta.threads.runs.create(
            thread_id=self.thread_id,
            assistant_id=self.assistant_id,
        )

        tool_id = None
        tool_function_output = None
        while True:
            result = client.beta.threads.runs.retrieve(
                thread_id=self.thread_id, run_id=run.id
            )
            print("run.status", result.status)
            if result.status == "requires_action":
                # require_actionのパラメータの取得
                tool_call = result.required_action.submit_tool_outputs.tool_calls[0]
                tool_id = tool_call.id
                tool_function_name = tool_call.function.name
                tool_function_arguments = json.loads(tool_call.function.arguments)

                if tool_function_name == "get_available_reservation_time":
                    reservation_date = tool_function_arguments["reservation_date"]
                    guest_count = tool_function_arguments["guest_count"]
                    tool_function_output = get_available_reservation_time(
                        reservation_date, guest_count
                    )
                elif tool_function_name == "complete_reservation":
                    reservation_datetime = tool_function_arguments["reservation_datetime"]
                    reservation_type = tool_function_arguments["reservation_type"]
                    name = tool_function_arguments["name"]
                    telephone_number = tool_function_arguments["telephone_number"]
                    tool_function_output = complete_reservation(reservation_datetime, reservation_type, name, telephone_number)
                break

            elif result.status == "completed":
                break
            time.sleep(0.5)

        print("tool_id:", tool_id)
        print("tool_function_output:", tool_function_output)

        if tool_id and tool_function_output:
            run_tool = client.beta.threads.runs.submit_tool_outputs(
                thread_id=self.thread_id,
                run_id=run.id,
                tool_outputs=[
                    {
                        "tool_call_id": tool_id,
                        "output": tool_function_output,
                    }
                ],
            )

            while True:
                tool_result = client.beta.threads.runs.retrieve(
                    thread_id=self.thread_id, run_id=run_tool.id
                )
                print("fc run.status", tool_result.status)
                if tool_result.status == "completed":
                    break
                time.sleep(0.5)

        messages = client.beta.threads.messages.list(
            thread_id=self.thread_id, order="asc"
        )

        if len(messages.data) < 2:
            return ""

        return messages.data[-1].content[0].text.value
```

まず、ThreadのMessageにユーザーの質問を設定します。

```py
self.client.beta.threads.messages.create(
    thread_id=self.thread_id,
    role="user",
    content=text,
)
```

`thread_id`, `role` と `content` パラメーターは必須です。

RetrievalかCode Interpreterを使用する場合、アップロードしたファイルのIDを指定する事ができます。

詳しいパラメーターはこちらを参照ください。

https://platform.openai.com/docs/api-reference/messages/createMessage

次にAssistantとThreadを紐づけてタスクを実行するRunを作成します。

```py
run = self.client.beta.threads.runs.create(
    thread_id=self.thread_id,
    assistant_id=self.assistant_id,
)
```

Runは `thread_id` と `assistant_id` パラメーターが必須です。

その他のパラメーターとして、`model`, `instructions`, `tools` と `metadata` を指定してAssistantの設定をオーバーライドする事ができます。

詳しくはこちらを参照ください。

https://platform.openai.com/docs/api-reference/runs/createRun

次にRunの `retrieve` でタスクを実行します。

```py
while True:
    result = self.client.beta.threads.runs.retrieve(
        thread_id=self.thread_id, run_id=run.id
    )
    if result.status == "completed":
        break
    time.sleep(0.5)
```

retrieveは `thread_id` と `run_id` パラメーターが必須です。

https://platform.openai.com/docs/api-reference/runs/getRun

Runはタスクがキューで実行される為、定期的に `retrieve` を実行、ステータスを取得して `requires_action` もしくは `completed` に移行したかどうかを確認します。

Function Callingが使われる場合、ステータスは `requires_action` に移行します。

キューのステータスが `requires_action` に移行したら、Function Callingのパラメーターを取得します。

```py
tool_call = result.required_action.submit_tool_outputs.tool_calls[0]
tool_id = tool_call.id
tool_function_name = tool_call.function.name
tool_function_arguments = json.loads(tool_call.function.arguments)
```

`tool_id` にはユニークなtoolのID、 `tool_function_name` には呼び出される関数名、 `tool_function_arguments` には関数の引数が設定されています。

```py
if tool_function_name == "get_available_reservation_time":
    reservation_date = tool_function_arguments["reservation_date"]
    guest_count = tool_function_arguments["guest_count"]
    tool_function_output = get_available_reservation_time(reservation_date, guest_count)
elif tool_function_name == "complete_reservation":
    reservation_datetime = tool_function_arguments["reservation_datetime"]
    reservation_type = tool_function_arguments["reservation_type"]
    name = tool_function_arguments["name"]
    telephone_number = tool_function_arguments["telephone_number"]
    tool_function_output = complete_reservation(reservation_datetime, reservation_type, name, telephone_number)
```

このように、`tool_function_name` で呼び出される関数名を判定し、 `tool_function_arguments` で関数の引数を取得して実際に関数を呼び出します。

関数の戻り値を `tool_function_output` に設定します。

次に関数の戻り値を含めたRunを作成して再度タスクを実行します。

```py
if tool_id and tool_function_output:
    run_tool = client.beta.threads.runs.submit_tool_outputs(
        thread_id=self.thread_id,
        run_id=run.id,
        tool_outputs=[
            {
                "tool_call_id": tool_id,
                "output": tool_function_output,
            }
        ],
    )

    while True:
        tool_result = client.beta.threads.runs.retrieve(
            thread_id=self.thread_id, run_id=run_tool.id
        )
        print("fc run.status", tool_result.status)
        if tool_result.status == "completed":
            break
        time.sleep(0.5)
```

`runs.submit_tool_outputs` でFunction Callingの結果を含めたRunを作成しています。

`runs.submit_tool_outputs` APIは `thread_id`, `run_id` と `tool_outputs` パラメーターが必須です。

https://platform.openai.com/docs/api-reference/runs/submitToolOutputs

`runs.submit_tool_outputs` で作成したRunに対して `retrieve` を実行して、キューのステータスが `completed` になるまで待機します。

キューのステータスが `completed` になったら `messages.list` で会話履歴を取得し、最後のMessageを返します。

```py
messages = self.client.beta.threads.messages.list(
    thread_id=self.thread_id, order="asc"
)

if len(messages.data) < 2:
    return ""

return messages.data[-1].content[0].text.value
```

この最後のMessageがユーザーの質問に対する最終的なAssistantの回答になります。

# 余談

Assistants APIでFunction Callingが実行されると、タスク分だけ `retrieve` が実行されます。

色々実験している中で同じプロンプトに対して、Assistants APIとChat Completions APIを何度も実行してみましたが、Completions APIの方が体感1.5倍程速く感じました。

Assistants APIはChat Completions APIと違ってエージェントの動きをするのでThreadの処理やタスク数 x Runの実行時間分のオーバーヘッドがあるのかなと思います。

もし今回のようにFunction Callingだけの用途でAssistants APIを使うのであれば、同等機能を持っているChat Completions APIの方が良いかもしれません。

ただしAssistants APIのRetrievalやCode Interpreterは強力なので、今後も検証してAssistants APIの可能性を探ってみたいと思います。

また、Assistants APIはstream機能などがまだ未実装なので、今後のアップデートに期待したいです。
