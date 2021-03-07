---
title: List、Maybe、Eitherに親しむ
---

本章ではIdrisを書く上で一番使う3つのデータ型、 `List` 、 `Maybe` 、 `Either` を使っていきます。

前章では軽く標準ライブラリで定義された `List a` や `Maybe a` 、 `Either a b` を使ってみました。本章ではもう少し色々な操作を扱ってみます。これらのデータ型はIdrisに限らず関数型言語で非常によく使うデータ型なので関数型言語っぽさがでます。そういった点も踏まえてご覧下さい。

# List型と再帰

まずは `List` の操作の基本に馴染みましょう。これらは標準ライブラリで定義されているものですが、手習いとして再実装します。

復習すると `List a` はおおまかには以下のように定義されているのでした。

```idris
infixr 7 ::
data List a = Nil | (::) a (List a)
```

そして `::` は右結合なので `1 :: 2 :: 3 :: Nil` は `1 :: (2 :: (3 :: Nil))` と解釈され、先頭にどんどん値を追加できるのでした。逆に、パターンマッチをすると先頭から1つ値を取り出せるので要素を1つ1つ処理する関数が便利です。

例えばリストの長さを求める関数は1要素につき1足していけば求まります。

例： `List a` の長さを求めるコード

```idris
length : List a -> Integer
length Nil       = 0
length (_ :: tl) = 1 + (length tl)
```

```
Idris> length ["a", "b", "c"]
3 : Nat
```


あるいは要素全ての和を求める関数はこう書けます。

例： `List Integer` の全ての要素の和を求めるコード

```idris
sum : List Integer -> Integer
sum Nil        = 0
sum (hd :: tl) = hd + (sum tl)
```

```text
Idris> sum [1, 2, 3]
6 : Integer
```


ここで長さ0のリストの要素の和は0としました。これは `(sum list1) + (sum [])` == `sum (list1 ++ [])` という性質を満たす、つまり `sum` とリストの結合の順番を気にしなくてよくなるので扱いやすいからです。

ここで、2つのリストを結合する関数 `(++)` は以下のように定義できます。

例：2つの `List a` を結合する関数の定義

```idris
(++) : List a -> List a -> List a
(++) Nil        l = l
(++) (hd :: tl) l = hd :: (tl ++ l)
```

```
Idris> [1, 2, 3] ++ [4, 5, 6]
[1, 2, 3, 4, 5, 6] : List Integer
```

慣れてきましたか？もうパターンですよね。ではこういうのはどうでしょう：リストの全ての要素1つ1つに関数を適用する関数 `map` の定義。例えば以下のように使います。

例： `map` 関数の動作例

```text
-- 全ての要素に1を足すのでそれぞれ 2, 3, 4になる
Idris> map (1+) [1, 2, 3]
[2, 3, 4] : List Integer

-- 全ての要素の `length` をとるので
-- `[length [], length [1], length [1, 2, 3]]` を計算する
Idris> map length [[], [1], [1, 2, 3]]
[0, 1, 3] : List Nat
```

これは高階関数、すなわち関数を引数にとる関数です。最近だとメインストリームにある言語でもよく使うので説明不要でしょうか。これも定義してみましょう。

例：リストの要素1つ1つに関数を適用する関数の定義

```idris
map : (a -> b) -> List a -> List b
map _ Nil        = Nil
map f (hd :: tl) = f hd :: map f tl
```

これもまた見慣れたパターンですね。今まででてきた関数と比べると`map` は関数を引数に取る分汎用性が高いです。特に「リストの要素に何かしらの計算を施す」という頻出のパターン、ループ処理の一種を関数で表現できています。

## リストの基礎：Foldr

さて、今まで定義した3つの関数、同じようなパターンで書いていましたよね。再掲すると以下です。


```idris
length : List a -> Integer
length Nil       = 0
length (_ :: tl) = 1 + (length tl)

sum : List Integer -> Integer
sum Nil        = 0
sum (hd :: tl) = hd + (sum tl)

(++) : List a -> List a -> List a
(++) Nil        l = l
(++) (hd :: tl) l = hd :: (tl ++ l)

map : (a -> b) -> List a -> List b
map _ Nil        = Nil
map f (hd :: tl) = f hd :: map f tl
```

`(++)` と `map` は余計な引数がついていますが、実際にはそれらの引数には何も操作していないので一旦目を瞑ると以下のようなパターンが見えてきます。

```idris
func Nil        = <何かの値>
func (hd :: tl) = <何かの操作> hd (func tl)
```

これは `<何かの値>` と `<何かの操作>` を引数で与えることにすれば関数に抽象化できそうじゃないですか？実際できて、以下のように定義されます。

```idris
foldr : (a -> b -> b) -> b -> List a -> b
foldr _ init Nil        = init
foldr f init (hd :: tl) = f hd (foldr f init tl)
```

この関数は `foldr` と呼ばれます。気持としてはリストの `Nil` の `init` に、 `::` を `f` に置き換える操作です。例えば `1 :: (2 :: (3 :: Nil))` というリストであれば `f 1 (f 2 (f 3 init))` に置き換えます。

`foldr` を用いて今まで出てきた関数を書き換えられます。`length` は `init` に `0` 、 `f` に最初の引数を無視して1足す関数を据えると元と同じ動きですね。

```idris
length : List a -> Integer
length l = foldr (\_,len => 1 + len) 0 l
```

`sum` が一番分かりやすいというか `foldr` っぽいですね。`init` に `0` を、 `f` に `(+)` を据えます。

```idris
sum : List Integer -> Integer
sum l = foldr (+) 0 l
```

残りの2つは書けますか？こうです。

```idris
(++) : List a -> List a -> List a
(++) l1 l2 = foldr (::) l2 l1

map : (a -> b) -> List a -> List b
map f l = foldr (\hd,acc => f hd :: acc) Nil l
```

色々な関数が `foldr` で書き直せました。そういった意味で `foldr` はリスト操作の基礎になる関数です。

ここで `foldr` という名前に触れておきましょう。 `foldr` で実装した `sum` は `1 :: (2 :: (3 :: Nil))` を `1 + (2 + (3 + 0))` にする関数なのでした。複数の要素を1つに纏めるので「畳み込み」という意味で `fold` という名前がついています。そしてカッコの優先順位により `3 + 0` が最初に計算されますよね。一番右から計算するので右の r がついています。

では左から計算する `foldl` もあってよさそうですよね？実際存在します。

## ループとFoldl

`foldl` を実装していきましょう。 `1 :: (2 :: (3 :: Nil))` を `f 3 (f 2 (f 1 init))` に書き換えます。これはさっきまでのように簡単には実装できないので補助関数を使います。

```idris
foldl : (a -> b -> b) -> b -> List a -> b
foldl f init l = loop l init
where
  loop : List a -> b -> b
  loop []         acc = acc
  loop (hd :: tl) acc = loop tl (f hd acc)
```

ここで使われている `loop` 関数は常套手段です。リストの先頭から計算し、計算の結果を `acc` に積み上げて持ち回ります。結果を積み上げる（accumulate）ので累積引数などと呼ばれます。

名前からも分かるかと思いますが、 `foldl` は手続的なコードでのループによく似ています。例えばCで配列の和をとる処理はこう書けますね。

``` c
// arr = ...;
// length = ...;
int sum = 0;
for (usize_t i = 0; i < length ; i++) {
    sum += arr[i];
}
return sum;
```

それがIdrisの `foldl (+) 0 list` に対応します。まあ、これは `foldr` で書いた `sum` と結果は同じですね。

`foldl` の使いどころとしてはリストを逆順にする `reverse` 関数などがあります。

``` idris
reverse : List a -> List a
reverse l = foldl (::) [] l
```

`1 :: (2 :: (3 :: Nil))` を `3 :: (2 :: (1 :: Nil))` に書き換えるのでちゃんと逆順になっていますね。

## Foldl vs Foldr

`reverse` は `foldl` でしか書けないので明快でいいのですが、 `sum` のように `foldl` でも `foldr` でも書ける関数は `foldl` と `foldr` のどちらを使えばいいのでしょう。答えは `foldl` が使えるなら `foldl` を使います。これはパフォーマンス上の理由によります。

`foldr` で実装した `sum` と `foldl` で実装した `sum` の挙動を追い掛けて確認しましょう。`foldl (+) 0 [1, 2, 3]` と `foldr (+) 0 [1, 2, 3]`で比べます。

```
foldl (+) 0 [1, 2, 3]  | foldr (+) 0 [1, 2, 3]
loop [1, 2, 3] 0       | (+) 1 (foldr (+) 0 [2, 3])
loop [2, 3] ((+) 1 0)  | (+) 1 ((+) 2 (foldr (+) 0 [3]))
loop [2, 3] 1          | (+) 1 ((+) 2 ((+) 3 (foldr (+) 0 [])))
loop [3] ((+) 2 1)     | (+) 1 ((+) 2 ((+) 3 (0)))
loop [3] 3             | (+) 1 ((+) 2 3)
loop [] ((+) 3 3)      | (+) 1 5
loop [] 6              | 6
6                      |
```

注目してほしいのは途中で発生する式の大きさです。`foldl` の方は逐次 `+` を計算できるので式の大きさが一定です。一方で `foldr` はリストの最後にいくまで計算できないので途中式大きくなってしまいます。この途中式がスタックを消費するので効率が悪い上に、大きなリストを与えるとスタックオーバーフローが起きかねません。 `foldl` の方は途中式がないとコンパイラが見抜いてループに書き換えてくれます。なので `loop` 関数は本当にループなのです。どういう条件で書き換えてくれるかは **末尾再帰の最適化** などのワードで検索してみて下さい。

## Foldを使わないとき

`foldr` と `foldl` がリスト処理の基礎と説明しました。では、foldを使わないケースはあるのでしょうか。まあ、感覚で分かるかと思いますが、あります。foldはリストの全ての要素について繰り返すので、ループを途中で打ち切りたい場合なんかにはfoldを使えません。

例えばリスト内に指定した条件を満たす要素があるか調べる関数 `any` はこう実装することになるでしょう。

例：リスト内に指定した条件を満たす要素があるか調べる関数の定義

``` idris
any: List a -> (a -> Bool) -> Bool
any []         _ = False
any (hd :: tl) f = if f hd then True else any tl f
```

`::` のマッチのところで `f hd` が `True` かどうかで再帰をするかしないかを分岐しています。foldではこういう制御ができません。余談ですが分かりやすさのために `if` 式を使いましたがこの実装は `||` 演算子を使って `f hd || any tl f` と簡潔に書けます。

# Maybeを使う

さっきはあるかないかだけを返す関数でしたが、今度はそのインデックスを返す関数 `position` を定義してみましょう。この関数はIdrisの標準ライブラリにはないようでした。今度はインデックスを持ち回る必要があるので `loop` 補助関数を使います。

例：リスト内に指定した条件を満たす最初の要素のインデックスを返す関数の定義（部分）


``` idris
position: List a -> (a -> Bool) -> Nat
position l f = loop l 0
where
  loop : List a -> Nat -> Nat
  -- ...
```

`List a` と要素が条件を満たすか判定する `a -> Bool` 関数を引数にとり、返り値は `Nat` 、自然数を返します（[#0は自然数](https://twitter.com/search?q=%230は自然数&src=typed_query&f=live)）。リストの長さは0以上で上限は存在しないのでそのインデックスも0以上で上限が存在しない `Nat` を返すのが適切です。

`[]` と `::` で分岐しますが、 `::` の方は分かりやすいですね。こう実装します。

``` idris
  loop []         n = ?hole
  loop (hd :: tl) n = if f hd then n else loop tl (1 + n)
```

問題は `[]` のときの処理です。再帰が `[]` まできたということは、そのリストに該当する要素がないということです。存在しないもののインデックスは返せませんね。どうしましょう。言語によっては `-1` を返すようなAPIもありますが、それを知らずに使ってしまうとエラーの元になります。さらに今回は0以上の値しかとらない `Nat` を使っているのでそもそも `-1` は返せません。どうしたもんでしょう。

こういうときは `Maybe` を使います。`Maybe` 型を復習するとおおむね以下のように定義されているデータ型です。

``` idris
data Maybe a = Nothing | Just a
```

`Maybe` のときにデータがないことを表わし、 `Just` のときにデータがあることを表わします。ということで `Maybe` を使って `position` を実装するとこうなります。


``` idris
position: List a -> (a -> Bool) -> Maybe Nat
position l f = loop l 0
where
  loop : List a -> Nat -> Maybe Nat
  loop []         n = Nothing
  loop (hd :: tl) n = if f hd then Just n else loop tl (1 + n)
```

これを使う側は `case` 式で `Maybe` が `Nothing` なのか `Just` なのかを場合分けすることになります。

例： `position` の結果に対して `case` で場合分けするコード片

``` idris
let list = [1, 2, 3, 4] in
case position list (\n => n `mod` 2 == 0) of
  Nothing => "No evens"
  Just i  => "Found an even at " ++ (show i)
```


これは `position` で `2` が `(\n => n `mod` 2 == 0)` を満たすので `Just 1` が返り、 `case` 式全体としては `"Found an even at 1"` が返ります。

## Maybeを操作する

`Maybe` を操作する関数も色々とあります。先の `case` を使った式を関数で書き直してみましょう。

まずは `map` です。 `map` は `Nothing` には何もせず、 `Just` ならその値に関数を適用します。

例： `map` の動作例

``` text
Idris> map (1+) Nothing
Nothing : Maybe Integer
Idris> map (1+) (Just 0)
Just 1 : Maybe Integer
```

イメージとしては以下のような定義です。

``` idris
map : (a -> b) -> Maybe a -> Maybe b
map _ Nothing  = Nothing
map f (Just x) = Just (f x)
```

この `map` を使えば `Just` の方の値を制御できます。
つ
次は `fromMaybe` です。 `fromMaybe` は `Maybe a` が `Just x` なら `x` を、 `Nothing` なら引数で与えた `a` の値を返します。

例： `fromMaybe` の動作例

``` text
Idris> fromMaybe True (Just False)
False : Bool
Idris> fromMaybe True Nothing
True : Bool
```

イメージとしては以下の定義です。

``` idris
fromMaybe : a -> Maybe a -> a
fromMaybe def Nothing  = def
fromMaybe def (Just j) = j
```

さっきから「イメーイとしては」といっているのは実際には少しだけ複雑な定義をしているためです。複雑というよりまだ紹介していない機能を使った実装ですね。大枠としては変わらないので今はあまり気にしなくてよいです。

さて、これらを組み合わせれば先程の `position` の返り値で条件分岐するコードは条件分岐なしに書き換えられます。

``` text
let list = [1, 2, 3, 4] in
fromMaybe "No evens"
          (map (\i => "Found an even at " ++ (show i))
               (position list (\n => n `mod` 2 == 0)))
```

`map` と `fromMaybe` はよく使うのでえておいて下さい。因みにそんなによくは使わないんですが、今回のユースケースだとピタリとあてはまる関数に `maybe` があります。それを使えばもう少し短くなります。

``` idris
let list = [1, 2, 3, 4] in
maybe "No evens"
      (\i => "Found an even at " ++ (show i))
      (position list (\n => n `mod` 2 == 0))
```

`maybe` は要するに `Maybe` の `Nothing` と `Just` を別のもので置き換える関数ですね。ほぼ `Maybe` への `case` 式を抽象化したようなものです。

例： `maybe` 関数の定義イメージ

``` idris
maybe : b -> (a -> b) -> Maybe a -> b
maybe n j Nothing  = n
maybe n j (Just x) = j x
```

これも覚えておいて下さい。

# Eitherでエラー処理

maybeのcaseとかmapとかfromMaybeとか

foldと*でreturn
foldと/でeither とか？
