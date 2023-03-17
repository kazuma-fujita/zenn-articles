---
title: "Amplifyで作成したLambda(Python)でasyncioインストール後に発生するUserCodeSyntaxErrorトラブルシューティング"
emoji: "🚨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - "Python"
  - "AWS"
  - "Lambda"
  - "Amplify"
published: true
---

`amplify add function` コマンドで作成した Lambda (Python) で amplify push コマンド実行時 UserCodeSyntaxError が発生したので、そのトラブルシューティングです。

# 環境

- macOS Ventura 13.1（22C65）
- Amplify 10.8.1
- pipenv version 2022.12.19
- virtualenv 20.17.1

# 再現方法

1. pipenv で asyncio インストール
1. amplify push コマンド実行
1. 内部的に pipenv install 実行
1. pipenv install 中に UserCodeSyntaxError 発生

# 原因

Python で非同期処理を行う為に、asyncio と aioboto3 を pipenv でインストールしました。

ですが、 asyncio は Python3.7 から標準で組み込まれている為、自分でインストールする必要がありませんでした。

Amplify で作成する Lambda はデフォルトで Python ランタイムが 3.8 なのでそもそも asyncio のインストールは不要でした。

それに気づかず amplify push で Lambda をデプロイして実行した所以下のエラーが発生しました。

```
{
  "errorMessage": "Syntax error in module 'index': invalid syntax (base_events.py, line 296)",
  "errorType": "Runtime.UserCodeSyntaxError",
  "stackTrace": [
    "  File \"/var/task/asyncio/base_events.py\" Line 296\n
    future = tasks.async(future, loop=self)\n"
  ]
}
```

恐らく Python 標準では無く、自分でインストールした asyncio パッケージを参照しようとしてエラーになったと仮説を立てました。

# 解決方法

問題解決の為、非同期処理周りのパッケージである asyncio と aioboto3 を pipenv uninstall コマンドでアンイストールしましたが、変わらずエラーが発生しました。

どうやら、pipenv uninstall コマンドで削除したパッケージは内部的に依存パッケージとして残っている場合があるようです。

ここらへん Python 初心者なので良く分かって無いのですが、以下で解決しました。

`pipenv graph` コマンドを実行して現在インストールされているパッケージと依存パッケージを一覧化します。

aioboto3 と asyncio が残っています。

```
$ pipenv graph
aioboto3==11.0.1
  - aiobotocore [required: ==2.4.2, installed: 2.4.2]
    - aiohttp [required: >=3.3.1, installed: 3.8.3]
      - aiosignal [required: >=1.1.2, installed: 1.3.1]
        - frozenlist [required: >=1.1.0, installed: 1.3.3]
      - async-timeout [required: >=4.0.0a3,<5.0, installed: 4.0.2]
      - attrs [required: >=17.3.0, installed: 22.2.0]
      - charset-normalizer [required: >=2.0,<3.0, installed: 2.1.1]
      - frozenlist [required: >=1.1.1, installed: 1.3.3]
      - multidict [required: >=4.5,<7.0, installed: 6.0.4]
      - yarl [required: >=1.0,<2.0, installed: 1.8.2]
        - idna [required: >=2.0, installed: 3.4]
        - multidict [required: >=4.0, installed: 6.0.4]
    - aioitertools [required: >=0.5.1, installed: 0.11.0]
      - typing-extensions [required: >=4.0, installed: 4.5.0]
    - botocore [required: >=1.27.59,<1.27.60, installed: 1.29.88]
      - jmespath [required: >=0.7.1,<2.0.0, installed: 1.0.1]
      - python-dateutil [required: >=2.1,<3.0.0, installed: 2.8.2]
        - six [required: >=1.5, installed: 1.16.0]
      - urllib3 [required: >=1.25.4,<1.27, installed: 1.26.14]
    - wrapt [required: >=1.10.10, installed: 1.15.0]
asyncio==3.4.3
```

次に `pipenv clean` コマンドで Pipfile.lock に記載されていないパッケージを削除します。

```
$ pipenv clean
Uninstalling wrapt...
Uninstalling aioboto3...
Uninstalling typing-extensions...
Uninstalling aiobotocore...
Uninstalling asyncio...
Uninstalling aioitertools...
```

この状態で Lambda をデプロイした所、正常動作しました。

# 振り返り

もしパッケージのせいでエラーが発生した場合、一度以下のファイルを解凍して不要なパッケージが残っていないか確認するべきでした。

```
amplify/backend/function/yourLambdaFunctionName/dist/latest-build.zip
```

latest-build.zip は amplify deploy コマンド実行時に作成される Python ファイルを全て圧縮して Lambda にデプロイするファイルです。

このファイルの中身を解凍すれば、どんなファイルやパッケージがデプロイされるか確認する事が出来ます。
