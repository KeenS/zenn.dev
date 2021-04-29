---
title: "インタフェース"
---

本章ではIdrisのインタフェースや重要なインタフェースである `Monad` 関連の機能について学びます。

<!--more-->

# インタフェース

色々な言語にあるやつとだいたい一緒です。厳密にいうとアドホックポリモーフィズムのための機構なのでHaskellの型クラスやRustのトレイトに例えた方がいいのですが、細かい話は置いておきましょう。

インタフェースは `interface インタフェース名 型変数 where 本体` の構文で定義します。本体の部分には値の型や実装などを書きます。値とは関数も含みます。例えば値を文字列にするインタフェース `Show` の定義は以下のように書けます。

例：インタフェース `Show` の定義

```idris
interface Show a where
    show : a -> String
```

`a` が `Show` を実装する型を表します。そして `show` には実装がありません。型だけ示しているのでインタフェースっぽいですね。この `Show` はプレリュードに定義されているので自分で書かなくても使えます。

これを実装するには `インタフェース名 型名 where 本体` の構文を使います。

例： `Name` 型にインタフェース `Show` を実装するコード

```idris
record Name where
  constructor MkName
  -- 型が同じフィールドは `,` で並べることでまとめて書ける
  firstName, lastName: String

Show Name where
  show (MkName firstName lastName) = firstName ++ " " ++ lastName
```

実装の方には逆に型の宣言がありません。

実装した `Show` は普通の関数のように呼び出すだけで使えます。

```text
Idris> show (MkName "Edwin" "Brady")
"Edwin Brady" : String
```

## デフォルト実装

先ほどインタフェースの本体には関数の型や *実装* を書くと説明しました。インタフェースに実装を持つこともできるんですね。

例えばプレリュードで定義されている等価比較を行なうためのインタフェース `Eq` はデフォルト実装を持ちます。

例：インタフェース `Eq` の定義

```idris
interface Eq a where
    (==) : a -> a -> Bool
    (/=) : a -> a -> Bool

    x /= y = not (x == y)
    x == y = not (x /= y)
```

等しい（ `==` ）は等しくない（ `/=` ）の逆だしその逆もまた然りという定義ですね。どちらか一方だけ実装すればもう一方は自動でついてくる仕組みです。


## 関連型

Idrisでは特に特別なものではないんですが、HaskellやRustで関連型（associated type）と呼ばれているものも書けます。

例えばとある別の型からその型の値を取得できる `Extract` というインタフェースを考えてみましょう。それは以下のように定義できます。

例：インタフェース `Extract` の定義

``` idris
interface Extract a where
  From: Type
  extract: From -> a
```

この `From` が関連型です。Idrisでは型も値なので関数や値のように普通にメンバーに書けばそれで済みます。

上記の型を `Name` に実装してみましょう。まず準備として `Name` を保持する型 `Person` を定義しておきます。

例：`Person` 型の定義

``` idris
record Person where
  constructor MkPerson
  age: Int
  name: Name
```

すると `Person` から `Name` を `extract` できるので、以下のように `Extract` を `Name` に実装できます。

例：`Extract` を `Name` に実装するコード

``` idris
Extract Name where
  From = Person
  extract = name
```

実装は `Person` レコードの定義からついてくる `name` をそのまま使うだけですね。

## 多パラメータのインタフェース

インタフェースの型パラメータは複数書くことができます。例えばある型から別の型に変換するインタフェース `Cast` はプレリュードで以下のように定義されています。

例：プレリュードでのインタフェース `Cast` の定義

``` idris
interface Cast from to where
    cast : (orig : from) -> to
```

`from` と `to` の2つのパラメータがありますね。実装するときも2つのパラメータを指定します。

例： `Cast` を `Double` と `Int` に定義するときの書き出し

``` idris
Cast Double Int where
  -- ...
```

`Cast` のパラメータに `Double` と `Int` を指定しています。`cast` の実装はプリミティブの呼び出しになるので省略しました。


# インタフェースを実装できる条件

インタフェースの実装は1つの型につき1つしか持てません [^1]。また関数はインタフェースを実装できません。例えば `Name` に対してもう1つの `Show` のインタンスを追加しようとするとコンパイルエラーになります。


[^1]: 名前付き実装という機能を使えばその限りではないのですが、発展的機能なので一旦置いておきます。


例：2つ目の `Show` インタフェースを `Name` に実装しようとした際に出るエラー

``` text
- + Errors (1)
 `-- (no file) line 0 col -1:
     interface.idr:21:1-9:Main.Name implementation of Prelude.Show.Show already defined
```


逆に、意外と実装できるケースにプリミティブを含む既存の型にインタフェースを実装できるというものが挙げられます。以下にその例を示します。

例：インタフェース `Zero` を定義し、それをプリミティブ `Int` に実装するコード

``` idris
interface Zero a where
  zero : a

Zero Int where
  zero = 0
```

プリミティブである `Int` に対してインタフェースを実装できました。

多パラメータのインタフェースはパラメータのどちらかが異なれば新たに実装できます。

例：プレリュードで `Cast` の実装が複数ある例

``` idris
Cast String Int where
    -- ...
Cast Char Int where
    -- ...
Cast Double String where
    -- ...
Cast Double Integer where
    -- ...
```

# ジェネリクスとインタフェース

ジェネリクスで扱う型に特定のインタフェースを実装していることを要求したい場合があります。

## ジェネリクス関数へのインタフェース制約

例えば引数を2つ受け取って、その小さい方、大きい方の順で並べて返す関数を定義したいとします。そのときに `<` で比較する必要がありますよね。
今までの知識で関数を定義すると `<` が実装されていないのでコンパイルエラーになります

例： `a` 同士を比較できないためエラーになるコード例

``` idris
ordered : a -> a -> (a, a)
ordered a b =
  if a < b
  then (a, b)
  else (b, a)

```

``` text
- + Errors (1)
 `-- Ordered.idr line 35 col 2:
     When checking right hand side of ordered with expected type
             (a, a)
     
     When checking argument b to function Prelude.Bool.ifThenElse:
             Can't find implementation for Ord a
```


そういうときは特定のインタフェースを実装している型のみ受け付ける制約を書きます。`インタフェース名 変数名 => 型` の構文です。`<` 演算子は `Ord` インタフェースで定義されているため、上記の `ordered` を修正すると以下のようになります。

``` idris
ordered: Ord a => a -> a -> (a, a)
ordered a b =
  if a < b
  then (a, b)
  else (b, a)
```

また、複数の制約を書きたい場合は `(インタフェース名 変数名, インタフェース名 変数名, ....) => 型` と丸括弧で括ってカンマで区切って書きます。

例：ジェネリクスの型変数に複数のインタフェース制約を書いたコード

``` idris
orderedMsg: (Ord a, Show a) => a -> a -> String
orderedMsg a b =
  let (a, b) = ordered a b in
  (show a) ++ " < " ++ (show b)

```

## インタフェース実装へのインタフェース制約

インタフェース自身にも関数を含みますからインタフェースの実装にインタフェース制約を加えたいというのも自然な要求です。実際、そういう機能があります。 `インタフェース制約 => インタフェース名 型名 where 本体` の構文です。以下に例を示します。

例：プレリュードのタプルへの `Eq` の実装例

``` idris
(Eq a, Eq b) => Eq (a, b) where
  (==) (a, c) (b, d) = (a == b) && (c == d)
```

余談ですが、Idrisの3つ組以上のタプルは2つ組の組み合わせの糖衣構文となっています。例えば `(A, B, C)` は `(A, (B, C))` と書くのと同じです。なので上記の2つ組のタプルの実装で全てのタプルの実装をカバーできるのです。

## インタフェースの拡張

インタフェース制約をインタフェースの宣言に書くこともできます。これは事実上既存のインタフェースを拡張した新しいインタフェースを定義していると捉えることもできますね。 `interface インタフェース制約 => インタフェース名 型変数 where 本体` の構文になります。

例えばプレリュードの `Neg` は `Num` を拡張したインタフェースです。

例：プレリュードの `Num` と `Neg` のコード


``` idris
||| The Num interface defines basic numerical arithmetic.
interface Num ty where
    (+) : ty -> ty -> ty
    (*) : ty -> ty -> ty
    ||| Conversion from Integer.
    fromInteger : Integer -> ty

||| The `Neg` interface defines operations on numbers which can be negative.
interface Num ty => Neg ty where
    ||| The underlying of unary minus. `-5` desugars to `negate (fromInteger 5)`.
    negate : ty -> ty
    (-) : ty -> ty -> ty

```


# 本章のまとめ

Idrisのインタフェースの機能について学びました。 `interface` で宣言し、 `implementation` で型に実装するのでした。
