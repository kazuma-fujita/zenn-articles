---
title: "OpenAIのWhisper, TTS, Assistants APIで長期記憶を持った音声会話型ボットを作る"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - "ChatGPT"
  - "OpenAI"
published: true
---

こんにちわ。 [ZUMA](https://twitter.com/zuma_lab) です。

11/6にOpenAI DevDayが開催され様々な発表がありましたね。

https://devday.openai.com/

発表された新しいAPIを使って長期記憶を持った音声会話型ボットを作ってみました。

音声文字起こしにWhisper、推論実行にGPT-4 Turbo、文字から音声の生成にTTS、長期記憶保持にはAssistants APIを使っています。

# 成果物

以下成果物となります。

https://twitter.com/zuma_lab/status/1722428441319100546

Assistants APIのthreadで会話履歴を持っているので、文脈を読んで回答をしてくれます。

また、関西弁でしゃべるようにプロンプトでキャラを設定しています。

# 事前準備

```py
%pip install openai
%pip install pydub
%pip install sounddevice scipy
```

必要なパッケージをinstallします。

```py
import os

os.environ["OPENAI_API_KEY"] = "Your API Key"
```

自身のOpenAI API Keyを設定します。

# Assistantの作成

```py
from openai import OpenAI

client = OpenAI()

assistant_name = "My AI Assistant"

model_name = "gpt-4-1106-preview"

instructions = """
あなたは優秀な会話型アシスタントです。
あなたはとてもフレンドリーな性格で常にテンションが高いです。

### 指示

- 関西弁で話してください。
- 200文字以内で回答をしてください。
"""

assistant = client.beta.assistants.create(
    name=assistant_name,
    model=model_name,
    instructions=instructions,
)
```

今回新たに追加されたAssistants APIを使ってAssistantを作成します。

アシスタント識別子の `name` 使用するGPTのモデル名の `model` システムプロンプトの `instructions` を設定します。

11/14現在、まだGPT-4 Turboは安定版ではないので、今回はプレビュー版の `gpt-4-1106-preview` を使っています。

`instructions` で関西弁で話すように指示しています。

また、音声会話型のボットなので、なるべく推論速度を上げる為200文字以内で回答するように指示しています。

Assistants APIにはChat Completion APIの `max_tokens` のようなパラメーターが無いので、プロンプトで文字数制限をしています。

文字数制限をしないと長い推論結果が出力された場合、GPT-4 Turbo、TTSで時間がかかり過ぎてしまい会話する気が失せてしまいます・・・

ちなみに作成したassistantはOpenAIのPlayGroundで確認することができます。

![](https://storage.googleapis.com/zenn-user-upload/562e4a147482-20231113.png)


今回必要最低限のパラメーターのみ設定していますので、細かいパラメーターについては以下を参照ください。

https://platform.openai.com/docs/api-reference/assistants/createAssistant


# AI Assistant クラスの作成

次にAI Assistantクラスを作成します。

役割として、音声を録音する、Whisperで録音された音声を文字起こしする、GPT-4 Turboで推論を実行する、TTSで文字から音声を生成して再生する、の4つの処理を行います。

全文はこちらです。

:::details AIAssistant Class

```py
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
    # 音声認識モデル
    stt_model = "whisper-1"
    # 音声生成モデル
    tts_model = "tts-1"  # 高品質モデル tts-1-hd
    # 声質
    voice_code = "nova"  # 男性 alloy, echo, fable, onyx 女性 nova, shimmer

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

        while True:
            result = self.client.beta.threads.runs.retrieve(
                thread_id=self.thread_id, run_id=run.id
            )
            if result.status == "completed":
                break
            time.sleep(0.5)

        messages = self.client.beta.threads.messages.list(
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

それではどのように各APIを呼び出しているか、AI Assistantクラスの各関数を見ていきます。

# AI Assistant クラスの関数解説

## 初期化処理

```py
class AIAssistant:
    """
    OpenAIのAPIを利用して音声をテキストに変換し、AIアシスタントで処理し、音声に戻すクラス。
    """

    # 録音パラメータ
    fs = 44100  # サンプリングレート
    duration = 5  # 録音する秒数
    channels = 1  # モノラル録音
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
```

### クラス変数でモデルを定義する

AIAssistantのクラス変数には録音パラメーター、音声認識モデル、音声生成モデル、声質を設定しています。

音声文字起こしの Whisperモデルは現在の所 `whisper-1` のみです。

OpenAI DevDayで large-v3 whisperモデルが発表されましたが、まだAPIは未公開なので、今回は `whisper-1` を使用しています。

音声生成モデルは標準品質の `tts-1` を使用していますが、高品質の `tts-1-hd` を使用することもできます。

まだ試しきれていませんが、日本語だと体感では `tts-1` と `tts-1-hd` の品質差異、生成速度に差があまりなかったです。

https://platform.openai.com/docs/guides/text-to-speech

こちらでも以下の記載がありますね。

> 場合によっては、リスニングデバイスや個人によっては、音声に顕著な違いが感じられない場合があります。

声質は `alloy, echo, fable, onyx, nova, shimmer` の6種類から選択できます。

個人的に一番関西弁が上手かった(?) `nova` を設定しています。

### インスタンス変数でassistant_id, thread_idを設定する

インスタンス変数は、各APIで使用するassistant_id, thread_idを設定しています。

この2つのIDは以下で作成したthreadを実行する際に利用します。

```py
thread = self.client.beta.threads.create()
```

ここで生成しているthreadは会話のセッションで、今回は会話履歴を保持する目的で使用します。

ここでは素のthreadを生成していますが、`create` 時に `messages` パラメーターで最初の会話を設定することもできます。

`messages` パラメーターについてはこちらを参照ください。

https://platform.openai.com/docs/api-reference/threads/createThread


## 音声を録音する

```py
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
        sd.wait()
        print("...Finished recording")
        return recorded_data
```

`sounddevice` パッケージを使って音声を録音しています。

`sd.rec` で録音を開始し、 `sd.wait` で録音が終わるまで待機しています。

録音時間はクラス変数の `self.duration` で5秒に設定しています。

無音時間が続くと自動で録音停止できた方がUXが良いのですが、今回はライトに実装しています。

## Whisperで録音された音声を文字起こしする

```py
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
```

`pydub` パッケージを使って録音された音声を `output_audio_file` に保存し、 `openai` パッケージを使ってWhisperで文字起こししています。

ここでは必要最低限のパラメーターしか設定していない為、その他のパラメーターについては以下を参照ください。

https://platform.openai.com/docs/api-reference/audio/createTranscription

余談ですが、実験していて発見したのが、Whisperに無音のwavファイルを送っても空文字が返却されると思いきや、 `.` や `. .` などのピリオドのみの文字列が返ってきます。

また、キーボードのタッチ音や拍手の音など人の音声以外のwavファイルを送信すると、`おやすみなさい`, `ご視聴ありがとうございました`, `チャンネル登録お願いします` や `Thank you for watching.` などYouTuberっぽい会話の終わりの挨拶を返します。

何がなんでも文字起こしをしたいのでしょうか。

勝手な文字を生成されては困るケースもあると思うので `prompt` で無音時の調整や `temperature` で生成結果を固定化できたりするのか、また別の機会に試してみます。

## GPT-4 Turboで推論を実行する


一番重要な部分、Assistants APIで推論を実行する関数です。

https://platform.openai.com/docs/assistants/how-it-works

こちらはOpenAIのドキュメントから抜粋したAssistants APIの図です。

![](https://storage.googleapis.com/zenn-user-upload/7d2aca6421bc-20231114.png)

以下は相関関係を翻訳したものです。

```
Assistantは OpenAI のModelを使用し、Toolsを呼び出す AI です。

一つのThreadにはMessageオブジェクトリストを内包しています。

MessageオブジェクトはUserの入力プロンプト, Assistantの推論結果を保持しています。

Messageオブジェクトにはテキストの他、画像、その他のファイルを含めることができます。

RunはThread上のAssistantを実行するためのオブジェクトです。

Assistantは、ThreadのMessageを使用して、ModelとToolsを呼び出してタスクを実行します。

Run StepはAssistantが実行したステップの詳細なリストです。

Assistantは、タスクの実行中にToolsを呼び出したり、新たにMessageを作成したりできます。

実行ステップを調べると、Assistantが最終結果にどのように到達するかを見ることができます。
```

自分は最初ちょっと良く分からなかったですが、今は勝手にこんな理解をしています。

要するに Assistant は AI, Thread は会話の文脈、Message は会話の内容を保持しているイメージです。

Assistant と Thread の間には直接的な関係性は無く、中間テーブル的に Run があって、Run は Assistant と Thread を紐付けているイメージです。

Runを実行すると紐づいてる Assistant が実行されます。

Assistantは、ThreadのMessageの内容から、ModelとToolsを呼び出しタスクを実行します。

タスクに応じてRetrievalやCode Interpreter, Function Callingが実行されます。

Run Stepはそのタスクの1つ1つの実行履歴をリストで保持しているイメージです。

Assistants APIはAssistant、Thread、Message単体ではタスクを実行する事はできず、Runを実行する必要があります。

なので、Chat Completions APIには無かった新しい概念の Run が特に重要になります。

前置きが長くなりましたが、 `run_thread_actions` 関数の処理を見ていきます。

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

        while True:
            result = self.client.beta.threads.runs.retrieve(
                thread_id=self.thread_id, run_id=run.id
            )
            if result.status == "completed":
                break
            time.sleep(0.5)

        messages = self.client.beta.threads.messages.list(
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

今回はチャットのみの用途なので最低限の設定のみします。

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

その他のパラメーターとして、`model`, `instructions`, `tools` と `metadata` が指定できます。

`model`, `instructions`, `tools` はAssistantの設定をオーバーライド出来るらしいです。

`file_ids` パラメーターが無いので、ファイル指定のオーバーライドは出来ないようです。

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

Runはタスクがキューで実行される為、定期的に `retrieve` 実行、ステータスを取得して `completed` に移行したかどうかを確認します。

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

この messages.list はデフォルトで20件までMessageを返しますが、より多くの文脈を取得したい場合、 `limit` で100件まで取得可能です。

詳しくはこちらを参照ください。

https://platform.openai.com/docs/api-reference/messages/listMessages

## TTSで文字から音声を生成して再生する

```py
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

最後に `openai` パッケージを使ってTTSで音声を生成し、 `pydub` パッケージを使って再生しています。

`speed` パラメーターで音声の再生速度を変更することもできます。デフォルトでは `1.0` です。

筆者の場合、英語音声の再生はネイティブ並みの早口で聞き取れないので、1以下の値を指定してゆっくり再生しています。

その他詳しくは以下を参照ください。

https://platform.openai.com/docs/api-reference/audio/createSpeech

# 余談

OpenAIのPlayGroundでAssistants APIを色々試してみてあまりに便利だったので、もうChat Completions APIは要らないじゃない？

と思ったのですが、 Assistants APIは `max_tokens` や `temperature`、 `seed` などのパラメーターが見当たりませんでした。

更にAssistants APIにはまだstream機能などが実装されていません。

また、同じプロンプトに対して、Assistants APIとChat Completions APIを何度も実行してみましたが、Completions APIの方が体感1.5倍程速く感じました。

なのでエージェント的な使い方をしないのであれば今まで通り Chat Completions API を使うべきかなと思いました。
