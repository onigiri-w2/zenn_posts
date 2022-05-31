---
title: "pytest 最低限入門"
emoji: "📖"
type: "tech"
topics: ["python", "pytest"]
published: true
---

pytestの使い方を毎回忘れてしまうので、よく使う関数や機能・ライブラリをまとめておく。
今後もまた忘れた時にこのページを見返すことにする。

## 基本中の基本
### `assert`
`pytest`の基本中の基本。この関数で結果の検証を行う。
検証するコードがTrueを返すならテスト成功、Falseを返すならテスト失敗にする。
個人的には、これだけでテストできるシンプルさが好き。
テストライブラリによくある`assertEquals`のような検証関数が沢山あるのは苦手。
```py
def test_ok():
    actual = 2
    assert actual == 2
    
def test_fail():
    actual = 2
    assert actual == 1
```

### `pytest.raises`
これも`pytest`の基本中の基本。算数で言う引き算くらいの基本。
例外が発生するかのチェックを行う。
このブロックの中で例外が発生したらテスト成功、発生しなければテスト失敗。
```py
def test_error():
    # 発生してほしいErrorクラスをraisesに引数として与える
    with pytest.raises(ValueError):
	# 例外が発生する関数実行
        func_raise_error()
```

## 絶対見逃せない便利機能：テストコードの短縮化・保守性向上
### `pytest.mark`
#### 基本：テストのカテゴライズ機能
`@pytest.mark.database`などとテストケースに付与すると、カテゴリ別にテスト実行することが可能になる
以下のテストコードがある状態で、`pytest -m database`とCLIから実行すると、`@mark.database`とデコレートされてるものしか実行されなくなる。
```py
@pytest.mark.database
    def test_database_access():
        assert ...
    @pytest.mark.no_database
    def test_print():
        assert ...
```
#### `pytest.mark.parameterize`
「同じようなテストするけど、データだけ違う...テストコードが太る...」ていう状況を解決してくれる。
同じテストコードに対して、似たようなデータを1つのケースでテストできる。
テストの保守性向上・可読性向上につながる。
```py
　  @pytest.mark.parametrize("data1,data2", [
　  	(1, "a"),
　  	(2, "b")
　  ])
　  def test_sample(data1, data2):
　  	assert sut(data1, data2) == True
```


### `pytest.fixture`
様々なテスト共通で利用する処理をまとめたり、事前処理、事後処理なんかも受け持てる。
#### 基本中の基本
`@pytest.fixture`で定義した関数の名前を、テストケースの引数に与えて利用する。
ここも`assert`の時と一緒で自由度が高い。
`fixture`関数で様々な処理をテストケースから分離できる。
```py
@pytest.fixture
def readable_file():
    with open("test_file.txt") as f:
        return f

def test_read_file(readable_file):
    assert reaable_file.read() == '...'
```
#### 事後処理のやり方
答えは決まっていて、`yield`を使えばいい。
テスト前は`yield`までの処理が実行され、テスト後に`yield`以降の処理が実行される。
```py
@pytest.fixture
def readable_file():
    f = open("test_file.txt")
    yield f
    f.close()
    
def test_read_file(readable_file):
    assert reaable_file.read() == '...'
```

#### 引数を与える方法
ラップを利用して引数付きの関数を返すようにすればいい。
```py
@pytest.fixture
def readable_file():
    def inner(path):
        with open(path) as f:
        return f
    return inner

def test_read_file(readable_file):
    assert reaable_file("test_file.txt").read() == '...'
```
#### 全テスト共通の処理を定義したいなら
testsディレクトリの配下に`conftest.py`を作成し、そこに`fixture`関数を記載すればいい。
`conftest.py`にfixtureを定義すれば、様々なテストコードから利用可能になる。
```py
# conftest.pyにreadable_fileを定義してる状態

# 同じファイルにfixture関数を定義してなくても利用可能。conftest.pyから参照してる。
def test_read_file(readable_file):
    assert reaable_file("test_file.txt").read() == '...'
```
#### 引数に与えずとも、自動で実施させたい
`autouse=True`を利用すればいい。
テストケースにわざわざ引数を与えずとも、影響する全テストケースの実行前に処理が走る。
```py
@pytest.fixture(autouse=True)
    def all_test_config():
        print('all use')
```
## 便利ライブラリ
### `pytest-describe`：Rspec(ruby)のようにテストコードを階層構造で書ける
`pip install pytest-describe`でインストール可能。
`describe_*`という名前で関数を書けば、永遠に階層構造にできる。
`describe_*`配下にある、それ以外の関数はテストケースとして扱われる。
```py
def describe_parent():
    def describe_child():
        def it_test():
            assert ...
        def describe_grandchild():
            def it_grand_test():
                assert ...
    def describe_child2():
        def it_test():
            assert ...
```

### `pytest-mock`：モック ライブラリ
モックを簡単に作り出せるライブラリ。必要なら本コードの一部をモック化することも可能
mockを利用するにあたって、最低限覚えておくと便利なことだけ記載しておく
#### 準備
1.`pytest-mock`をインストールする
```bash
pip install pytest-mock
```
2.テストケースの引数に`mocker`を与える（`fixture`の応用）
```py
def test_sample(mocker):
    ...
```
3.`mocker`を利用して色々モック化する
```py
def test_sample(mocker):
    mocker.patch("src.sample.add", return_value=2)
        ...
```

#### 便利機能1：mocker.Mock -- いろんな形のモックを作る --
##### なぜわざわざMockを使うのか？
例えば、自分で`SampleMock`のようなクラスを作って実装すればいいのでは？
A. 自分で作るのは面倒臭い + Mockの便利関数が利用できる
- 流石に自分でMockを作るのは面倒臭いし、テストコードの可読性が一気に落ちる
- また、Mockには「何回呼び出されたか？」がわかるような便利関数も用意されてる

##### 基本①：return_value  --  mockが呼出された時の返り値を設定する
```py
>>> a = mocker.Mock(return_value='hoge')
>>> a()
'hoge'
```
##### 基本②：side_effect  --  mockが呼び出された時の関数そのものや例外を設定する
```py:関数まるごと入替
>>> m = mocker.Mock(side_effect=lambda: 'hoge')
>>> m()
'hoge'
```
```py:例外発生
>>> m = mocker.Mock(side_effect=Exception)
>>> m()
raise Exception(...)
```

##### 基本③：return_value、side_effectの両方設定する場合、side_effectが優先される
```py
>>> a = mocker.Mock(return_value='hoge', side_effect=lambda: 'boge')
>>> a()
'boge'
```

##### 応用①：クラスを似せたオブジェクト作成
```py
def test_sample(mocker):
    m = mocker.Mock()
    # 「m」にprop_hogeプロパティを設定（値は1）
    m.prop_hoge = 1
    # 「m」にfunc_hogeというメソッドを設定（返り値は"aaaaaa"）
    m.func_boge = mocker.Mock(return_value="aaaaaa")

    # return_valueにMockを与えるってのも、利用する場面ある
    return_mock = mocker.Mock()
    return_mock.banana = "バナナ"

    # 「m」にreturn_mockを返すモック関数を設定
    m.func_return_mock = mocker.Mock(return_value=return_mock)
```
インスタンスのモックを作る時の基本はこれだけ
`return_value`に更にモックを設定してもいいし

#### 便利機能2：mocker.MagicMock ---- Mockのサブクラス、利用する場面は多い
Mockと昨日はほとんど同じ。
Mockとの違いは、特殊メソッドをデフォルトで実装してるかどうかくらい。
参考：[Python めざせモックマスター①（Mockオブジェクトのふるまいを指定する） - Qiita](https://qiita.com/kanae_y/items/4019fbfe4db4a07521cd#mock%E3%81%AE%E3%82%B5%E3%83%96%E3%82%AF%E3%83%A9%E3%82%B9magicmock)

#### 便利機能3：mocker.patch ---- 本番コードの関数、クラス、モジュールをモック化できるやばいやつ
テストを実行する時だけ、本番コードの内容を変えることができる
対象がクラス、メソッド、関数、モジュールでもなんでも、モック化できる本当にやばくてcoolなやつ
基本的な入れ替え方を知っておけば、あとは大体応用が効く
##### 関数をモックと入れ替える
```py:sample.py
def func1(name):
    return f"hello!!{name}"
def func2(name):
    return func1(name)
```
```py:test_sample.py
from sample import func2

def test_func2(mocker):
    value = "no hello"
    mocker.patch("src.sample.func1", return_value=value)
    assert func2("...") == value
```

##### クラス丸ごとモックと入れ替える
```py:src/sample.py
def func1():
    s = Sample()
    return s.execute()

class Sample:
    def execute(self):
        return "execute"
```
```py:tests/test_sample.py
from sample import Sample

def test_func1(mocker):
    text = "execute2!!"
    m = mocker.Mock(spec=Sample)
    m.execute.return_value = text
    mocker.patch("src.sample.Sample", return_value=m)
    assert func1() == text
```

##### importしてるモジュールもモックと入れ替えちゃう
```py:src/sample.py
import requests

def func1(url):
    return func2(url)

def func2(url):
    try:
        return requests.get(url).status_code
    except:
        raise Exception()
```
```py:tests/test_sample.py
from src.sample import func1

def test_sample(mocker):
    status_code = 200
    url = "http://hoge.com"
    m = mocker.Mock()
    m.status_code = status_code
    mocker.patch("src.sample.requests.get", return_value=m)
    assert func1(url) == status_code
```

#### 便利機能4：mocker.patch.object ---- 痒い所に手が届くmocker.patchを少し変えたやつ
`mocker.patch`と役割は一緒なのだが、少し使い方が異なる
「テスト対象が利用してるクラスの一部のメソッドだけモック化したい」とかいう場面で使うかも
使い方だけ見ておけば、あとはノリで行けるはず
`mocker.patch`と違うのは、モジュールをモック化したいなら、そのモジュールをテストファイルでimportする必要があるという点

以下は`requests.get`を`mocker.patch.object`でモック化するパターン
```py:src/sample.py
import requests

def func1(url):
    return func2(url)

def func2(url):
    try:
        return requests.get(url).status_code
    except:
        raise Exception()
```
```py:tests/test_sample.py
import requests
from src.sample import func1

def test_sample(mocker):
    status_code = 200
    url = "http://hoge.com"
    m = mocker.Mock()
    m.status_code = status_code
    mocker.patch.object(requests, "get", return_value=m)
    assert func1(url) == status_code
```


## 参考
- [/PythonOsaka/Pythonのテストフレームワークpytestを使ってみよう](https://scrapbox.io/PythonOsaka/Python%E3%81%AE%E3%83%86%E3%82%B9%E3%83%88%E3%83%95%E3%83%AC%E3%83%BC%E3%83%A0%E3%83%AF%E3%83%BC%E3%82%AFpytest%E3%82%92%E4%BD%BF%E3%81%A3%E3%81%A6%E3%81%BF%E3%82%88%E3%81%86)
- [pytestの使い方 - note](https://note.com/npaka/n/n84de488ba011)
- [pytestでMockを使う - Qiita](https://note.com/npaka/n/n84de488ba011)