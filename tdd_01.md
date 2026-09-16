# テスト駆動開発入門

---

## この章のゴール

- pytestというツールの基本操作を身につける
- テストコードを先に書く、というTDD(テスト駆動開発)のサイクル(Red → Green → Refactor)を体験する
- 「境界値」という考え方に触れ、テストケースをどう増やしていくかの感覚をつかむ

この章では、「純粋関数」(入力を渡すと、決まった値を返すだけの関数)だけを題材にします。TDDの基本を学びます。

---

## 0. 準備:pytestを使ってみる

### インストール
仮想環境を構築後、pytestをインストールします。

```bash
pip install pytest
```

### ディレクトリ構成

```
tdd_practice/
├── bmi.py
├── test_bmi.py
├── cart.py
└── test_cart.py
```

実装ファイルとテストファイルを同じ階層に置くだけで大丈夫です。pytestは`test_`から始まるファイルを自動的に見つけて実行してくれます。

### 最初のテスト(BMI計算)

```python
# bmi.py
def calculate_bmi(weight_kg, height_m):
    pass
```

```python
# test_bmi.py
from bmi import calculate_bmi

def test_calculate_bmi():
    assert calculate_bmi(70, 1.75) == 22.86
```

- `assert 式` は「式が`True`であること」を確認する文です。`True`でなければテストは失敗します。
- テスト関数の名前は必ず`test_`から始めます。これがpytestへの合図です。

実行してみます。

```bash
pytest
```

まだ実装が空なので失敗します。この「動かしてから失敗を確認する」体験がRedの入り口です。

実装を書いて、もう一度実行します。

```python
def calculate_bmi(weight_kg, height_m):
    return round(weight_kg / (height_m ** 2), 2)
```

```bash
pytest
```

`1 passed`と表示されれば成功です。

### 複数のテストケースを書く(合計金額計算)

```python
# cart.py
def total_price(items):
    return sum(items)
```

```python
# test_cart.py
from cart import total_price

def test_total_price_with_multiple_items():
    assert total_price([100, 200, 300]) == 600

def test_total_price_with_single_item():
    assert total_price([500]) == 500

def test_total_price_with_no_items():
    assert total_price([]) == 0
```

```bash
pytest
```

`3 passed`と表示されます。1つのファイルに複数のテスト関数を書けること、`pytest`コマンド一発でまとめて実行できることを確認します。

> **コラム:TDDを使わなくていい例**
> BMI計算・合計金額計算はどちらも、計算式が一発で決まってしまう題材です。最初から正解のコードが思いつくような処理に、律儀にRed→Green→Refactorのサイクルを回すのは、かえって手間を増やすだけになります。TDDは「あらゆる処理に適用すべきもの」ではなく、「使うかどうかを見極める判断」自体も大事なスキルです。

---

## 型(Red → Green → Refactor)の習熟

ここからは、あえて似た構造の題材を並べ、「テストを書く → 実装する → 整理する」というサイクルそのものに集中します。

### ディレクトリ構成の切り替え

問題数が増えるので、実装とテストを分けます。

```
tdd_practice/
├── pyproject.toml
├── src/
│   ├── average.py
│   ├── discount.py
│   └── shipping.py
└── tests/
    ├── test_average.py
    ├── test_discount.py
    └── test_shipping.py
```

```toml
# pyproject.toml
[tool.pytest.ini_options]
pythonpath = ["src"]
testpaths = ["tests"]
```

これで`from average import average`のように、`src.`を付けずにインポートできます。実行方法は変わらず、プロジェクトルートで`pytest`と打つだけです。

> 注意: `pythonpath`オプションはpytest 7.0以降の機能です。`pytest --version`で確認しておきましょう。

### ① 平均点計算

**Red(1回目)**
```python
def test_average_of_three_scores():
    assert average([80, 90, 100]) == 90
```

**Green(1回目)**
```python
def average(scores):
    return sum(scores) / len(scores)
```

**Refactor(1回目)**
特に整理することはありません。毎回リファクタが必要とは限りません。

**Red(2回目)**
```python
def test_average_of_empty_list_raises_error():
    with pytest.raises(ValueError):
        average([])
```
空リストを渡すと`ZeroDivisionError`になり、意図した`ValueError`ではないため失敗します。ここで初めて「空リストならどうなる?」という境界に気づきます。

**Green(2回目)**
```python
def average(scores):
    if not scores:
        raise ValueError("scores must not be empty")
    return sum(scores) / len(scores)
```

### ② 割引後価格計算

**Red(1回目)**
```python
def test_discount_10_percent():
    assert discounted_price(1000, 0.1) == 900
```

**Green(1回目)**
```python
def discounted_price(price, discount_rate):
    return price * (1 - discount_rate)
```

**Red(2回目)**
```python
def test_discount_rate_over_100_percent_raises_error():
    with pytest.raises(ValueError):
        discounted_price(1000, 1.5)
```
150%割引という不正な値に気づく境界。実行すると`-500`が返ってしまい失敗します。

**Green(2回目)**
```python
def discounted_price(price, discount_rate):
    if not (0 <= discount_rate <= 1):
        raise ValueError("discount_rate must be between 0 and 1")
    return price * (1 - discount_rate)
```

**Red(3回目、テストは通ってしまう例)**
```python
def test_negative_discount_rate_raises_error():
    with pytest.raises(ValueError):
        discounted_price(1000, -0.1)
```
このテストは実装を変えなくても通ります。2回目のガード節がすでにカバーしていたためです。「テストは実装のためだけでなく、将来の変更を検知する保険でもある」ことを伝えられる場面です。

**Refactor**
```python
def _validate_discount_rate(discount_rate):
    if not (0 <= discount_rate <= 1):
        raise ValueError("discount_rate must be between 0 and 1")

def discounted_price(price, discount_rate):
    _validate_discount_rate(discount_rate)
    return price * (1 - discount_rate)
```

### ③ 送料計算(仮実装=Fake Itを体験)

> **先に仕様を決めておく**
> TDDは「仕様を発見する手法」ではなく「決まった仕様を検証しながら実装する手法」です。送料が無料になる条件などのビジネスルールは、テストを書き始める前に確定させておく必要があります。ここでは次の仕様で進めます。
> - 3000円未満: 送料500円
> - 3000円以上10000円未満: 送料300円
> - 10000円以上: 送料無料
>
> 以下のRed→Green→Refactorは、この決まった仕様を「どの順番でテストとして裏付けていくか」という着手順の話であり、テストを書きながら仕様そのものを発見しているわけではない点に注意してください。

**Red(1回目、まず仕様の一部からテストにする)**
```python
def test_fee_when_under_3000():
    assert shipping_fee(1000) == 500
```

**Green(1回目、あえての仮実装)**
```python
def shipping_fee(amount):
    return 500
```
テストを通すためだけの、明らかに"仮"の実装です。最初から正しい実装を書かなくていい、という心理的ハードルを下げる狙いがあります。

**Red(2回目、仕様の次の条件をテストにする)**
```python
def test_fee_free_when_amount_is_10000_or_more():
    assert shipping_fee(15000) == 0
```

**Green(2回目)**
```python
def shipping_fee(amount):
    if amount >= 10000:
        return 0
    return 500
```

**Red(3回目、仕様の残りの条件をテストにする)**
```python
def test_fee_discounted_between_3000_and_10000():
    assert shipping_fee(5000) == 300
```

**Green(3回目)**
```python
def shipping_fee(amount):
    if amount >= 10000:
        return 0
    elif amount >= 3000:
        return 300
    return 500
```

**Red(4回目、境界値ちょうどを狙う)**
```python
def test_fee_at_exact_boundaries():
    assert shipping_fee(3000) == 300
    assert shipping_fee(9999) == 300
    assert shipping_fee(10000) == 0
```
`>=`と`>`を間違えるだけで壊れる、境界値分析そのものの確認です。

**Refactor**
```python
_FEE_TIERS = [(10000, 0), (3000, 300)]

def shipping_fee(amount):
    for threshold, fee in _FEE_TIERS:
        if amount >= threshold:
            return fee
    return 500
```

> **コラム:テストが仕様を歪めてしまう危険**
> 仕様を先に確定させずにテストファーストで進めると、「テストを通すためだけの条件」が実装に紛れ込み、テストが後から仕様を作ってしまうことがあります。これは実務でもよく起きる失敗です。テスト駆動開発の「駆動」は、仕様に向かって実装を駆動させる、という意味であり、仕様そのものをテストで発見していくという意味ではありません。

---

## フェーズ1-B:境界の"概念"に触れる

「境界にはいくつかの種類の"概念"がある」

### ① 数値の境界:消費税の端数処理判定

> **仕様**
> - `tax_included_price(price, tax_rate, rounding)` は、税込み価格を計算する
> - `rounding="floor"` のとき、小数点以下を切り捨てる
> - `rounding="ceil"` のとき、小数点以下を切り上げる
> - `rounding` に `"floor"` `"ceil"` 以外の値が渡された場合は `ValueError` を送出する

```python
def test_tax_included_price_rounds_down():
    assert tax_included_price(998, 0.08, rounding="floor") == 1077

def test_tax_included_price_rounds_up():
    assert tax_included_price(998, 0.08, rounding="ceil") == 1078

def test_tax_included_price_invalid_rounding_raises_error():
    with pytest.raises(ValueError):
        tax_included_price(998, 0.08, rounding="unknown")
```
伝えたい概念: 同じ計算式でも「丸め方」という選択肢によって正解が変わる。数値計算は一意に決まらない場合がある。

### ② 文字列・組み合わせの境界:パスワード強度判定

> **仕様**
> `is_strong_password(password)` は、次のすべてを満たすときだけ `True` を返す。
> - 8文字以上である
> - 数字を1文字以上含む
> - 記号(`!` `?` `#` など)を1文字以上含む

```python
def test_password_too_short_is_invalid():
    assert is_strong_password("Ab1!") is False

def test_password_minimum_length_is_valid():
    assert is_strong_password("Ab1!efgh") is True

def test_password_without_symbol_is_invalid():
    assert is_strong_password("Abcdefg1") is False

def test_password_without_digit_is_invalid():
    assert is_strong_password("Abcdefg!") is False
```
伝えたい概念: 文字列の境界は「長さ」だけでなく「含まれる文字の種類」という軸でも発生する。

### ③ 集合・状態の境界:在庫リストの重複マージ

> **仕様**
> `merge_inventory(items)` は、`{"name": 商品名, "qty": 数量}` の辞書のリストを受け取り、商品名をキー、数量の合計を値とする辞書を返す。同じ商品名が複数回登場する場合は数量を合算する。空リストを渡した場合は空の辞書を返す。

```python
def test_merge_inventory_with_no_duplicates():
    items = [{"name": "apple", "qty": 2}, {"name": "banana", "qty": 3}]
    assert merge_inventory(items) == {"apple": 2, "banana": 3}

def test_merge_inventory_with_duplicates_sums_quantity():
    items = [{"name": "apple", "qty": 2}, {"name": "apple", "qty": 3}]
    assert merge_inventory(items) == {"apple": 5}

def test_merge_inventory_with_empty_list():
    assert merge_inventory([]) == {}
```
伝えたい概念: 「同じキーが複数回出てくる」というデータの"集まり方"に注目する視点。

### ④ 日付特有のルール:うるう年判定

> **仕様**
> `is_leap_year(year)` は、次のルールでうるう年かどうかを判定する。
> - 4で割り切れる年はうるう年
> - ただし100で割り切れる年はうるう年ではない
> - ただし400で割り切れる年はうるう年

```python
def test_divisible_by_4_is_leap_year():
    assert is_leap_year(2024) is True

def test_divisible_by_100_is_not_leap_year():
    assert is_leap_year(1900) is False

def test_divisible_by_400_is_leap_year():
    assert is_leap_year(2000) is True

def test_not_divisible_by_4_is_not_leap_year():
    assert is_leap_year(2023) is False
```
伝えたい概念: 一見単純に見えて、例外規則が入れ子になっているルールがある。境界値は数値の大小だけでなく、条件の重なりでも生まれる。
