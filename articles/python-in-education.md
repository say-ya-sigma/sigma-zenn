---
title: "Pythonが教育用途において十分だという話"
emoji: "🐷"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["Python"]
published: true
---

# Pythonが教育用途において十分だという話

今話題のPythonを教えている現役の講師です。Pythonを教える際に重視すべきだと考えている機能等について書いておきます。

# dataclass / Pydantic

自分は型ヒントよりもdataclassやPydanticを使った型付けを重視しています。いわゆるクラスベースな言語の書き方が大事だと考えています。

## dataclass

Pythonは動的型付け言語であり、interface相当の機能すらclassの構文で書く変わった言語です。近年Pythonの型ヒントは少しづつ充実してきていますが発展途上であることは否めないですし、何より実行時にその型であることは保証されないので、dataclass等を使った開発スタイルが依然強力だと考えています。

Python+TypeScriptというようなスタックを使う際には両言語の差に混乱するでしょう。TypeScriptでは必要に迫られた時しかclassを使いませんが、これはJavaScriptがプロトタイプベースの言語であり実行時にclassすらobjectに還元されてしまうことからこのような文化が定着していると考えられます。

一方でPythonはクラスベースの言語なので、PHPやJavaを書くときにそうするように、多くの実装をclassで書くべき言語です。実際はそうなっていないことはとても嘆かわしいことです。

例えば値オブジェクトのようなものを書きたい場合、Pythonではdataclassを使います。

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class User:
    name: str
    email: str
```

```python
user = User('sigma', 'sigma@mail.com')

user.name = 'SIGMA' # dataclasses.FrozenInstanceError: cannot assign to field 'name'
print(user) # User(name='sigma', email='sigma@mail.com')

user2 = User('sigma', 'sigma@mail.com')

print(user == user2) # True
```

frozenによりデータがイミュータブル(*)であることを保証しています。__repr__が設定されprint()時の表現が得られたり__eq__が設定されたりします。

> \* Pythonのclassに真の意味でのイミュータブルはないため、この表現は正確ではないですが、上書きを制限することの意義というようなことを説明する際に他に適当な語がないためこの語を使っています。

[https://docs.python.org/3/library/dataclasses.html](https://docs.python.org/3/library/dataclasses.html)

よくあるアンチパターンは生のdictを使うことです。

```python
user_by_dict = {'name': 'sigma', 'email': 'sigma@mail.com'}
```

何がダメというか全部ダメなんですが、このパターンの恐ろしいところは例えば後から任意のキーを追加し得るところです。

```python
user_by_dict['something'] = 'something'
```

これによりプログラム中には「このdictに特定のこのキーがあればこうする」というような分岐が発生したりします。そのキーが実際に追加されている個所や追加される条件はプログラムの奥深くに眠っていたりします。どうしようもないPythonのコードに遭遇した場合、まず間違いなく生のプリミティブなdictがふんだんに使われています。

別の解決策としてTypedDictを使う方法もあります。この方法もかなり良くはありますが、Pythonそのものの型ヒントの書き方がバージョンによって違うなどすることもあり、運用が面倒だという印象はあります。

[https://mypy.readthedocs.io/en/stable/typed_dict.html](https://mypy.readthedocs.io/en/stable/typed_dict.html)


## Pydantic

Pydanticは相当クールなソリューションです。砂漠を歩く者を引き寄せるオアシスのようにPython開発者を引き寄せています。

値オブジェクトを書く際には各値がどのような値であるべきかバリデーションを書いたりします。これは当然実行時にも評価されるため値に一定の保障を与えてくれますが、値を受け渡す度に発生する計算であり、本質的に遅いです。Pydanticは豊富なオプションを使って汎用的なバリデーションを記述できますが、バリデーションの実装はrustで書かれています。

例えばidが1以上であることを保証したい場合は以下のように書きます。

```python
from pydantic import BaseModel, Field

class User(BaseModel):
    id: int = Field(ge=1)
    name: str

user = User(id=0, name='sigma') # pydantic_core._pydantic_core.ValidationError: 1 validation error for User
# id
#   Input should be greater than or equal to 1 [type=greater_than_equal, input_value=0, input_type=int]
#     For further information visit https://errors.pydantic.dev/2.6/v/greater_than_equal
```

id=1等とすると色んな情報を提供してくれるエラーが出てきます。例えばバックエンドを書くのであればリクエストのスコープの殆ど全ての区間でPydanticを使うことで、実行時エラーを頼りにしたデバッグが格段にやりやすくなります。ミュータブルである必要がなければ `frozen=True` にして扱います。

```python
class User(BaseModel, frozen=True):
    id: int = Field(ge=1)
    name: str

user = User(id=1, name='sigma')
user.id = 2 # pydantic_core._pydantic_core.ValidationError: 1 validation error for User
# id
#   Instance is frozen [type=frozen_instance, input_value=2, input_type=int]
```

# ABCとDI

PythonにはABCという冗談みたいな名前のパッケージがあります。Abstract Base Classessの略のようで抽象クラスを書くためのパッケージです。dataclassで表現するのが適切でないようなクラスにはこれを使います。

[https://docs.python.org/ja/3/library/abc.html](https://docs.python.org/ja/3/library/abc.html)

Pythonにはインターフェースがないため、DIP(Dependency Inversion Principle)を実現するためにはABCを使います。例えばRepositoryクラスの抽象クラスは以下のように書きます。

```python
from abc import ABC, abstractmethod
from entity.user.UserId import UserId

class AbstractUserRepository(ABC):
    @abstractmethod
    def find_all(self):
        raise NotImplementedError

    @abstractmethod
    def find_by_id(self, id: UserId):
        raise NotImplementedError
```


dataclassやPydanticとABCを使えばそれほど型に困ることはなくなります。classが全ての中心であるPythonにとってこれらは欠かすことのできない要素です。

## DIコンテナ

特にFlask等のマイクロフレームワーク扱う上で、DIコンテナの存在は無視できないものです。サービスクラスをリクエストの度にインスタンス化するようなコードを見たことがあるでしょうか。速度的にはそれほど致命的ではありませんが、SDGsの時代に資源の無駄遣いは避けたいです。もっともPythonは多くのタスクで資源を余計に使うんですが。

Pythonはシンプルな言語だからシンプルなマイクロフレームワークを使おうというシンプルな考え方は良いとは思いますが、その方針を採用するのであれば必要な時に必要な技術要素を知っているかどうかが命運を分けます。

## injector

DIコンテナのパッケージであるinjectorを使えばインスタンスを使いまわせます。3Rのreuseです。エコですね。

それ以外にも例えば抽象クラスと具象クラスをバインドすることもできます。Webバックエンドでは殆ど必須のテクニックと言えるのではないでしょうか。

バインドやリゾルブをするクラスは例えば以下のような形になります。InjectorにBinderを受け取る関数を渡しますが、その関数をclass内にstaticmethodとして書いています。

```python
from injector import Binder, Injector
from repository.UserRepository import AbstractUserRepository, UserRepository
from service.UserService import UserService


class Dependency:
    def __init__(self) -> None:
        self.injector = Injector(self.config)

    @staticmethod
    def config(binder: Binder):
        binder.bind(AbstractUserRepository, to=UserRepository)
        binder.bind(UserService, to=UserService)

    def resolve(self, cls):
        return self.injector.get(cls)

di = Dependency()
```

例えばサービスクラスには以下のように注入できます。

```python
from entity.user.UserId import UserId
from injector import inject
from repository.UserRepository import UserRepository

class UserService:
    @inject
    def __init__(self, user_repository: UserRepository):
        self.user_repository = user_repository

    def get_user(self, user_id: UserId):
        return self.user_repository.find_by_id(user_id)
```

# パッケージ管理システムとlockファイル

「きちんとlockファイルを生成するパッケージ管理システムを使いましょう。」ということは強く教えています。

Pythonには `requirements.txt` を使って必要なパッケージを管理する方法があり、実際に広く使われています。現在インストールされているパッケージとそのバージョンをファイルに起こし、どこでもインストールできるようにするというものです。この方法は正直言って原始的過ぎます。友だちに書き捨てのスクリプトをシェアしたい、というような用途でなければ推奨はしません。

`requirements.txt` の最大の問題点は直接依存するパッケージのみが記述されていて、依存の依存のそのまた依存というような情報が一切記載されていません。そのような情報が書かれたファイルはlockファイルと呼ばれ様々な言語のデフォルトのバージョン管理システムで当たり前のように生成されます。

lockファイルには依存の依存のそのまた依存というような依存ツリーに現れるパッケージがバージョン名、ハッシュ値等と共に記載されています。同じパッケージ群の下で実行されることをきちんと保証するにはそれらの情報が必要です。

## 教育用に `requirements.txt` を使うことの是非

バージョン管理システムは現代の開発にはなくてはならないので、教育用の書き捨てスクリプトであってもちゃんとしたバージョン管理システムを使うべきだと考えています。確かに教育用のコードの多くは書き捨てではあると思いますが、 `requirements.txt` をその問題点についてのエクスキューズ無しに使うことは避けられるべきです。

自分はpoetryを使っています。poetryはそれほど新しくないですが、現代的なバージョン管理システムではあると思います。

## クラウド実行環境

[Colaboratory](https://colab.research.google.com/) と呼ばれるクラウド実行環境があり、numpyなどがimportするだけで使える点も特筆すべき点として挙げておきます。

Colaboratryは実際にはjupyter notebookのための環境であり、正確にはPythonの実行環境ではありませんが、実行可能なサンプルコードとマークダウンを同時に記述する形式としては優れています。

クラウドの実行環境が存在する言語は沢山ありますが、この点においてもPythonは教育用途において必要十分だと言えると思います。

# 感想

PHPやJavaでも値オブジェクト等の考え方などは使いますし教育用途には十分なんじゃないかと思います。強いて言えばclassのメンバやメソッドへのアクセス制御についてのPythonの考え方は独特かなとは思います。普通に `private` とか書かせてほしいんですが、Pythonのクラスに完全なprivateはありません。これは Guido Van Rossum が言ったとされる「We're all consenting adults here」というモットーに基づいていて、Pythonではユーザーの意思次第でクラスを自由に拡張できるべきだと考えられていることに由来します。全然わからん。

Pythonについて色々言いたいのは分かりますが、じゃあRで書かれた保守性のかけらもないシステムを保守したいんですか？という話だと思います。Pythonで書かれた酷いコードを減らすためにPythonを滅ぼしても他の言語で酷いコードを書くだけなんじゃないですかね。どう考えてもその層はRustとかには流れない。
