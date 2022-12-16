---
title: "Next.jsで発生するReact Media RecorderのBlob is not defined エラー対処方法"
emoji: "🚨️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - "nextjs"
  - "react"
  - "javascript"
published: true
---

こんにちわ。 [ZUMA](https://twitter.com/zuma_lab) です。

とあるプロジェクトでブラウザ経由で音声録音する為に React Media Recorder パッケージを導入しています。

```
npm i react-media-recorder
```

React Media Recorder は内部的に WEB 標準 API の MediaRecorder を使っていて `useReactMediaRecorder` hook なども用意されており簡単に録音機能を実装できます。

以下は React Media Recorder を導入していた時にエラーでハマったので回避策を Tips として残しておきます。

# React Media Recorder で `Blob is not defined` が発生する場合

Next.js で React Media Recorder パッケージを使用すると `Blob is not defined` が発生する場合があります。

以下の issues でエラーについて議論されていました。

https://github.com/0x006F/react-media-recorder/issues/103

Next.js はデフォルトでサーバサイドでレンダリングされるので MediaRecorder などクライアントサイドで動作するモジュールは動的 import をする必要があるようです。

issues を参考に以下のように修正しました。

```ts:index.tsx
const Recorder = dynamic(
  () =>
	  // audio-recorderは内部でReact Media Recorderを呼び出しているComponent
    import("../src/components/audio-recorder").then(
      (module) => module.AudioRecorder
    ),
  { ssr: false }
);
```

# There is already an encoder stored which handles exactly the same mime types エラーが発生する場合

ただ、この `Blob is not defined` エラーを解消すると次は以下のエラーが発生しました。

```
Error: There is already an encoder stored which handles exactly the same mime types.
```

以下の issues でエラーについて議論されていました。

https://github.com/0x006F/react-media-recorder/issues/98#issuecomment-1133918236

以下和訳引用

> Next.js はデフォルト `next.config.js` の `reactStrictMode` が true になっています。
> Strict モードではすべてのコンポーネントが 2 回呼び出されるため、Extendable-Media-Recorder でこのエラーがスローされます。

Strict モードを OFF にするとエラーは発生しませんが、プロジェクト開発中は極力 Strict モードで開発したいです。

ワークアラウンドな対処法になってしまいますが、2022/12/16 時点で最新である React Media Recorder バージョン 1.6.6 を以下コマンドで バージョンを 1.6.5 にダウングレードすると取り急ぎ全てのエラーが解消されます。

```
npm i react-media-recorder@1.6.5
```

以上手短ですが、Next.js x React Media Recorder で発生するエラーの回避方法 Tips でした。
