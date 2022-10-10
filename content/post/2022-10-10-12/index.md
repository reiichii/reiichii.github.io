---
title: "TyperとPoetryでcliを作って公開するまでの流れ"
date: 2022-10-10T12:30:14+09:00
draft: false
categories:
    - tech
tags:
    - Python
---

cliを作りたかったというよりは、FastAPIの兄弟ツールTyperを触ってみたかったのと、Pythonでcliを作る流れを思い出したかったのが動機です。ですのでtodolistは最小限の機能だけを持っています。

最後にcliを作ったのはPython初めて1年目くらいの頃で、その頃はまだPoetryも触っておらず、setup.py書くの大変だったという記憶しかなかったのですが、今はこんなに手軽に作れてしまうんだなと感動でした。

## Poetryでパッケージを作成する

* Poetryでプロジェクトの雛形を作成します

    ```
    poetry new python_cli_todo

    .
    ├── README.md
    ├── pyproject.toml
    ├── r_todolist
    │   └── __init__.py
    └── tests
        └── __init__.py
    ```

* 作成したライブラリに必要なパッケージをインストールします。

    ```
    poetry add typer[all]
    ```
    今回はTyperだけです。
* pyproject.tomlにcli用の設定を追記

    ```
    [tool.poetry.scripts]
    todo = "r_todolist.main:app"
    ```
    `todo`コマンドで動くようになっています

* サンプルコードを載せておく

    動作確認をした後実装する予定なので、今は公式ドキュメントの以下のコードを配置しておきます。

    https://typer.tiangolo.com/tutorial/package/#create-your-app

* 仮想環境内でcliを実行してみる

    ```
    poetry install
    ```

    ```
    $ poetry run todo load
    Loading portal gun

    $ poetry run todo shoot
    Shooting portal gun
    ```

## アプリケーションの実装

今回実装するtodolistはCRUD4つのサブコマンドを持つものです。DBはSQLite3を使います。各サブコマンドの細かい出力例は[README](https://github.com/reiichii/python-todolist-cli#%E4%BB%95%E6%A7%98)に記載してあります。

* `add`: タスクの追加
* `ls`: タスクの一覧参照(デフォルトで未完了のもの・オプションで完了済みのものを表示できる)
* `done`: 完了したタスクを完了済みにする
* `rm`: タスクの削除(複数指定可)

### Typerについて

https://typer.tiangolo.com/

FastAPIの作者が作ったcliを作成するためのライブラリになります。作者曰く兄弟ライブラリとのことで、FastAPIの要素である「型定義してエディタのサポートを受け開発効率を上げる」をcli開発時にも実現する設計思想のようです。ただしPydanticではなくClickパッケージをベースに実現させているようでした。

他にもcli出力時にstyleを良い感じにしてくれるrichというライブラリや、パッケージをインストールしたときに自動で実行環境のシェルに合う形で自動補完設定を追記してくれる機能がついていたりなど、色々充実していました。

以下ではTyper周りの部分だけ抜粋する形で取り上げています。コードの全体は[main.py](https://github.com/reiichii/python-todolist-cli/blob/main/r_todolist/main.py)の1ファイルに全部収まっています。

### addサブコマンド

```python
import typer
from rich import print

app = typer.Typer()


@app.command()  # 1
def add(
    task: str = typer.Option(  # 2
        ...,
        prompt=True,  # 3
        help="what you have to do.",
        show_default="send a email",
    )
):
    """
    add task to list
    """  # 4
    db.insert_task(task)
    print("[green]added.[/green]")  # 5
```

1. `@app.command()` でサブコマンドを関数として定義できる
2. Typerの基本的な挙動として、関数の引数にデフォルト値を含めなければcli実行時の必須のパラメータになり、デフォルト値を指定すればcli実行時のoptionを作ることができる
  - 例えばadd関数の例では、上記の定義をすれば `--task` というoptionが使えるようになっており、`--task`で指定した値がtask変数に渡される
  - 今回はoptionで指定しているが、デフォルト値を省略しているため必須項目としてhelpの表示が出たり、バリデーションが走るようになっている
3. `typer.Option()`や `typer.Arguments()`を使えば、引数により詳細な設定を追加することができる
  - 例えば`prompt=True`にしていることで、この関数ではtaskの入力を対話形式で入力できる
  - `help`パラメータで、`--help`した時にこのオプションの説明を追加することができる
4. docstringで定義した文章は、`--help`時にそのままサブコマンドの説明として表示される
5. richライブラリの機能で、文字を緑色で表示させる。

### lsサブコマンド

```python
import typer
from rich import print

app = typer.Typer()

@app.command()
def ls(done: bool = typer.Option(False, help="Show only DONE tasks.")):  # 1
    """
    show incomplete tasks.
    """
    task_lists = db.get_lists(done)
    for l in task_lists:
        is_done = "\[x]" if l.is_done else "[]"  # 2
        print(f'- {is_done} {l.id}. "{l.task}"')
```

1. lsサブコマンドのデフォルトの引数を `False` に設定している。この定義をした時点で `--done` というoptionが使えるようになっており、これを指定するとTrueが渡される
2. markdownのチェックボックス形式でタスク一覧を表示させるようにしたかったが、`[x]`のように書いてしまうとrichの機能と競合して表示されなかった

### doneサブコマンド

```python
@app.command()
def done(
    id: int = typer.Argument(  # 1
        ..., help="Select task id", metavar="TASK_ID", show_default="1"  # 2
    )
):
    """
    check the task
    """
    task = db.done_task(id)
    print(f'{task.id}. "{task.task}" is done:tada:')  # 3
```

1. task_idが必須項目になるので `typer.Argument(...)`で定義した
2. metavarパラメータでhelpの表示時を指定できる。また関数のデフォルト値ではなく、helpに表示するときのデフォルト値を`show_default`で変えられる
3. richの機能で絵文字が表示させることができる

### rmコマンド

```python
@app.command()
def rm(
    ids: List[int] = typer.Argument(  # 1
        ...,
        help="Select task_ids separated by spaces.",
        show_default="1 2",
        metavar="TASK_ID",
    )
):
    """
    delete the tasks
    """
    db.delete_ids(ids)
    print(f"removed: {','.join([str(i) for i in ids])}.")
```

1. スペース区切りで入力したcliの引数値を、配列として渡すことができる

## PyPIへ公開する

* PyPIへログインし、API tokenを発行する

    ちなみにAPI tokenを発行する際にはscopeを選択するのですが、プロジェクトがない状態だと全プロジェクトしか選択できません。ただプロジェクトをpublishした後にプロジェクトの管理画面からスコープをプロジェクトに限定したAPI tokenが発行できるようでした。

* PoetryにPyPIのAPI tokenを設定する

    ```
    $ poetry config pypi-token.pypi {API TOKEN}
    ```

* publishする

    ```
    poetry publish --build
    ```

    publishしたらもう公開されており、pip installでインストールできるようになっています。また更新はpyproject.tomlで設定しているversionを更新して再度同じコマンドを実行する必要があります。

    https://pypi.org/project/r-todolist/ 🎉

## 参考

* [Building a Package \- Typer](https://typer.tiangolo.com/tutorial/package/)