---
title: "学校や TOEIC で絶対出てこない英語フレーズの発音練習が出来るアプリを作った"
emoji: "🗣️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - "nextjs"
  - "react"
  - "amplify"
  - "aws"
  - "english"
published: true
---

:::message
この記事は [クソアプリ Advent Calendar 2022](https://qiita.com/advent-calendar/2022/kuso-app) の 21 日目の記事です。
:::

こんにちわ。 [ZUMA](https://twitter.com/zuma_lab) です。

クソアプ初参加です。お手柔らかにお願いします。

学校や TOEIC のテストでは絶対出てこないであろうスラングのリスニングと発音練習が出来る Slang Boot Camp という WEB アプリを作ってみました。

このアプリで覚えたフレーズは英語のテストで一切役に立たない事を保証いたします。

あと、所構わずスラングばっかり使ってると失礼な奴と思われるので、用量、用法を守って正しくお使いください。

# こんなんです

以下成果物です。

https://main.d1ntt9o0lggnhf.amplifyapp.com/

[![](https://storage.googleapis.com/zenn-user-upload/a6f607816db7-20221223.png)](https://main.d1ntt9o0lggnhf.amplifyapp.com/)

デモ動画も作って YouTube に上げてみたのですが、あまりのスラング具合に年齢制限が課されてしまいました。

https://youtu.be/8ew40RsloKw

# 遊び方

まず英語問題文が読み上げれます。次にユーザーはリスニングした問題文をスピーキングします。

発音が正しければ祝ってくれますし、間違っていればスラングで罵られます。

# やってること

以下技術スタックです。

- フロントエンド
  - Next.js
- ホスティング
  - AWS Amplify
- 問題文の音声読み上げ
  - Amazon Polly
- ユーザー音声のテキスト変換
  - Amazon Transcribe
- 音声録音、音声 PCM データ stream 処理
  - Microphone Stream
- 音声再生
  - React Audio Player

処理は以下の流れです。

1. Amazon Polly で問題文を音声変換
1. React Audio Player で音声再生
1. Microphone Stream でユーザーの音声を録音
1. Amazon Transcribe で音声からテキスト変換
1. 変換されたテキストと問題文が合っているか判定
1. 合っていれば次の問題、以下繰り返し

音声録音の UX だけはこだわってて、音声の無音時間が 2 秒あると自動的に録音停止します。

アプリのソースコードはこちらです。

https://github.com/kazuma-fujita/slang-boot-camp-frontend

Amazon Polly の実装に関しては過去に記事を書いたので興味があればご覧ください。

https://zenn.dev/zuma_lab/articles/nextjs-amplify-text-to-speech

Amazon Transcribe の実装も過去に記事を書きました。

https://zenn.dev/zuma_lab/articles/nextjs-amplify-speech-to-text

# なんで作った

筆者は字幕無しでハリウッド映画を観るという目標があります。

ただ、現状ハリウッド俳優が何を言ってるのかサッパリわかりません。

たぶんこんな理由です。

- 英語力が中二
- ハリウッド俳優の会話速すぎ
- リンキング(単語間が繋がってる)してる
- リダクション(本来発音されるべき音が発音されてない)してる
- **スラングめっちゃ使ってる!**

英語学習において他にやること山の如しですが、なんかスラング覚えたいなーと思ってこのアプリを作りました。

時間無くて 17 問しか作れなかったのですが、ほんとは初級編、中級編、上級編とかコース別に分けたかったです。

来年のクソアプではもっと長文スラングに挑戦していきます。

それでは良いスラングライフを！
