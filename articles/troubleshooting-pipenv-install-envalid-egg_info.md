---
title: "Amplifyで作成したLambda(Python)で発生するinvalid command 'egg_info'エラー解決法"
emoji: "🚨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - "Python"
  - "AWS"
  - "Lambda"
  - "Amplify"
published: true
---

`amplify add function` コマンドで作成した Lambda (Python) で amplify push コマンド実行時 invalid command 'egg_info' エラーが発生したので、そのトラブルシューティングです。

Python 初心者な為、2 時間以上試行錯誤したので覚書として残しておきます。

# 環境

- macOS Ventura 13.1（22C65）
- Amplify 10.8.1
- pipenv version 2022.12.19
- virtualenv 20.17.1

# 再現方法

1. pipenv で asyncio インストール
1. amplify push コマンド実行
1. 内部的に pipenv install 実行
1. pipenv install 中に invalid command 'egg_info' 発生

以下エラー発生時のログです。

```
Pipfile.lock (6e44e5) out of date, updating to (242122)...
Locking [packages] dependencies...
⠙ Resolving dependencies...
usage: setup.py [global_opts] cmd1 [cmd1_opts] [cmd2 [cmd2_opts] ...]
   or: setup.py --help [cmd1 cmd2 ...]
   or: setup.py --help-commands
   or: setup.py cmd --help
error: invalid command 'egg_info'

🛑 Command failed with exit code 1: pipenv install

Learn more at: https://docs.amplify.aws/cli/project/troubleshooting/
```

# 原因

ChatGPT に聞いた所、どうやら setuptools が古くて egg_info が見つからないから更新しろとの事でした。

ChatGPT の指示通り `pip install --upgrade setuptools` コマンドを実行しても解決はしませんでした。

次に setuptools を利用しているファイルを調べました。

Amplify add function コマンドデフォルトで作成される `setup.py` ファイルと同階層にある `pyproject.toml` ファイルが setuptools を使用しているようでした。

```toml:pyproject.toml
[build-system]
requires = ["setuptools", "wheel"]
build-backend = "setuptools.build_meta:__legacy__"
```

そして `setup.py` には setuptools の記載がありません。

```py:setup.py
from distutils.core import setup

setup(name='src', version='1.0')
```

エラーログには `Resolving dependencies... usage: setup.py ` の記載があるので、どうやら `setup.py` を呼び出しているようです。

恐らくデフォルトの `setup.py` には setuptools パッケージの記載が無いのに setuptools に含まれている egg_info を使用しようとしてエラーとなっているようです。

# 解決方法

以下の方法で解決しました。

1. Lambda ディレクトリの `setup.py` 削除

```
rm amplify/backend/function/yourFunctionName/src/setup.py
```

2. pipenv install コマンドが実行出来るか確認

```
cd amplify/backend/function/yourFunctionName
pipenv install
```

pipenv install コマンドが正常に実行出来れば amplify push も問題無く実行出来ます。

`setup.py` を削除し `pyproject.toml` を使用するようにしたら pipenv install コマンドが通るようになりました。

pyproject.toml で setuptools を使うようにしたので上手く動作したのかと思います。
