---
title: "Amazon LexとAmazon Connectで電話でお店の予約が取れるボットを作ってみる"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - "Amazon"
  - "AWS"
  - "機械学習"
published: false
---

こんにちわ。 [ZUMA](https://twitter.com/zuma_lab) です。

最近は毎日 Edge の Bing AI を触って遊んでいます。

Google も ChatGPT に対抗したプロダクト Bard を発表して自然言語処理 AI の盛り上がりがすごいです。

以前の記事では OpenAI の GPT-3 モデルを使って、ChatGPT みたいな LINE チャットボットを作ってみました。

https://zenn.dev/zuma_lab/articles/gpt3-line-chatbot

今回は Amazon の自然言語処理 AI である Lex と 電話のコンタクトセンターサービスを簡単に構築できる Amazon Connect を使って電話でお店の予約が取れるボットを作ってみます。

# 成果物

以下成果物です。

https://www.youtube.com/watch?v=-S_RCPWaQ6w

以下構成図は AWS 公式から抜粋です。

今回は DB 登録処理はしないので Lambda には繋げず、Amazon Connect と Lex のみでボットを作ってみたいと思います。

![](https://storage.googleapis.com/zenn-user-upload/51756e103b00-20230217.png)

# 実行環境

- macOS Ventura 13.1（22C65）
- Amplify 10.6.2
- pipenv version 2022.12.19
- virtualenv 20.17.1

# Amazon Lex でチャットボットを作成する

Amazon Lex コンソールの `ボットを作成` ボタンを押下します。

![](https://storage.googleapis.com/zenn-user-upload/3d20ce17afda-20230217.png)

作成方法は `空のボットを作成します。` を選択します。

ボット名と説明を入力し、IAM アクセス許可は `基本的な Amazon Lex 権限を持つロールを作成します。` を選択して新規でロールを作成します。

児童オンラインプライバシー保護法 (COPPA)は `はい` を選択し、セッションタイムアウトはデフォルトの `5分` とします。

`次へ` ボタンを押下します。

![](https://storage.googleapis.com/zenn-user-upload/835eac93feb3-20230217.png)

次に、言語は `日本語` を選択します。

音声による対話は複数種類用意されているボイスの選択が出来ます。

音声サンプルから実際のボイスを聞く事ができます。

標準テキストとニューラルテキストの読み上げ音声がありますが、ニューラルテキストの方がより自然な発声をします。

[過去の記事](https://zenn.dev/zuma_lab/articles/nextjs-amplify-text-to-speech#amazon-polly-%E3%81%AE%E4%BA%8B%E5%89%8D%E7%9F%A5%E8%AD%98) でボイスの違いについて書いたので参照ください。

今回はニューラルテキストの `Kazuha` を選択しました。

インテント分類信頼スコアのしきい値はデフォルト値の `0.40` とします。

この値は後から変更可能です。

`完了` ボタンを押下してボットを作成します。

![](https://storage.googleapis.com/zenn-user-upload/c58b7c691ae2-20230217.png)

# Amazon Connect を構築する

Amazon Connect のコンソールから `Create Instance` ボタン押下します。

ID 管理は `Amazon Connect にユーザーを保存` を選択します。

`アクセス URL` には任意のサブドメインを入力します。

次に全ての項目を入力して管理者を追加してください。

テレフォニーオプションはデフォルト `着信を許可する` と `発信を許可する` にチェックが入っているのでそのまま進みます。

データストレージもデフォルト値のまま次に進みます。

最後に確認画面が表示されるので入力に問題無いことを確認して `インスタンスの作成` ボタンを押下します。

インスタンス作成が完了するまで 2〜3 分待ち、作成完了したら `今すぐ始める` ボタンを押下します。

Amazon Connect コンソールが表示されるので、言語設定を日本語に変えておきます。

`今すぐ始める` ボタンを押下します。

次の画面でブラウザに Amazon Connect のマイク使用許諾のダイアログが表示されるので許可します。

電話番号の取得画面で番号を取得します。

日本国内の場合直通ダイヤルと料金無料通話は以下となります。

- 直通ダイヤル (DID) 番号
  - DID 番号は市内局番。050 または 03 から始まる番号。
- 料金無料通話
  - 0120 または 0800 から始まる番号。

2023 年 1 月現在日本の電話番号は追加出来ませんでした。

[AWS の公式](https://docs.aws.amazon.com/ja_jp/connect/latest/adminguide/connect-tokyo-region.html) に以下の記載があります。

> アジアパシフィック (東京) リージョンで作成した Amazon Connect インスタンスの電話番号を取得するには、AWS サポートケースに問い合わせて、事業所が日本にあることを示すドキュメントを提出する必要があります。
> 事業用の電話番号のみ登録できます。個人用の電話番号は登録できません。

今回は個人の検証の為、US を選択しタイプは `DID(直通ダイヤル)` を選択します。

料金無料通話は日本国内からかけた場合、結局国際通話料がかかる上に AWS 料金も直通ダイヤルに比べて割高です。

Amazon Connect の料金形態は複雑な為、[こちら](https://blog.usize-tech.com/amazon-connect-calculator/) が纏まってて参考になりました。

直通ダイヤルも料金無料通話も番号を持っているだけで料金が発生するので、検証が終わったら番号を破棄するか、インスタンスを削除するなりしましょう。

電話番号の候補が表示されるので、好きな番号を選択します。

電話番号の取得画面が表示されるので `Continue` ボタンを押下します。

ダッシュボードが表示される事を確認します。

Amazon Connect コンソールを表示し、作成したインスタンスを選択後、左カラムの `問い合わせフロー` を押下します。

Amazon Lex のリージョンが `アジア・パシフィック:東京` となっている事を確認します。

先ほど作成したボットを選択します。

エイリアスには `TestBotAlias` を選択します。

`Amazon Lex ボットを追加` ボタンを押下します。

Amazon Connect ダッシュボード画面に戻り、 `4. プロンプトの作成` の `プロンプトの表示` リンクを押下します。

Amazon Connect ダッシュボード画面に戻り、 `5. 問い合わせフローの作成` の `問い合わせフローの表示` リンクを押下します。

`コンタクトフローの作成` ボタンを押下します。

コンタクトフロー作成画面になるので、左上にあるコンタクトフロー名を任意で入力します。

次に `音声の設定` ブロックを選択しコンタクトフローに配置、エントリと結合します。

# おわりに

次回は Amazon Lex の代わりに、OpenAI の Whisper と GPT-3 を使ってみたいと思います。

# 参考サイト

- [[Amazon Lex] Amazon Lex が日本語対応となったので、Amazon Connect から使用して居酒屋の電話予約をボット化してみました](https://dev.classmethod.jp/articles/amazon-lex-with-amazon-connect/)
