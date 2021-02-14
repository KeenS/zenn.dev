---
title: "基本文法"
---

本章でははIdrisの基本文法を解説します。


# コメント
## 1行コメント

`--` ではじめます。

例：

``` idris
-- これはコメントです
```

## 複数行コメント

`{-` から `-}`

例：

``` idris
{-
複数行で
コメントが
書けます
-}
```


### ドキュメントコメント

`|||` ではじまる1行コメントです。
詳しくは別の章で紹介します。

# 関数と変数

ドキュメントには書かれてないのですが、コンパイラのコードを見る限り識別子に使えるのは `[[:alphabet:]_][[:alphanum:]'_.]*` のようです。ここで `[:alphabet:]` に入るのはUnicodeで[Alphabetic](https://www.unicode.org/reports/tr44/#Alphabetic)なもので、`[:alphanumeric:]` は Alphabetic + Numeric全般のようです。例えば `hog'.'e12_` や `漢字の識別子１` は適格なIdrisの変数です。

## グローバル変数

変数は以下の構文で定義します。

``` idris
名前 : 型
名前 = 式
```

例：数値100を指すグローバル変数 `version` の定義

```idris
version : Integer
version = 100
```

## 関数

関数は以下の構文で定義します。

``` idris
名前 : 型
名前 引数1 引数2 .. 引数n = 式
```

関数の型は `引数1の型 -> 引数2の型 -> 引数2の型 -> 返り型` です。

例：2つの `Integer` である `x` と `y` を受け取り、 `Integer` であるその和を返す関数 `add` の定義

```idris
add : Integer -> Integer -> Integer
add x y = x + y
```


変数や関数はcamelCaseの命名が慣例です。コンパイラ側でも先頭に小文字がきたら変数と思って処理している箇所があるようですので、特別な理由がない限りは小文字始まりの変数名を使うとよいでしょう。。

## ローカル変数

`let 変数 = 式 in 続く式` の構文で定義します。

例： `x` と `y` の和をローカル変数 `tmp` に束縛したあと `tmp` と `z` の和を計算するコード

``` idris
add3 : Integer -> Integer -> Integer -> Integer
add3 x y z = let tmp = x + y in
             tmp + z
```

他にも式の後ろに `where` を続けて書く記法もあります。

例： `where` を使って `x` と `y` の和をローカル変数 `tmp` に束縛したあと `tmp` と `z` の和を計算するコード


``` idris
add3 : Integer -> Integer -> Integer -> Integer
add3 x y z = tmp + z
where
  tmp : Integer
  tmp = x + y
```

Idrisは [オフサイドルール](https://ja.wikipedia.org/wiki/オフサイドルール)を採用しているのでインデントが同じなら同じブロックとみなしてくれます。
`where` のあとに続く定義はインデントを揃えれば複数書けます。


## ローカル関数

`where` の記法で定義できます。

例： `where` を使って3つのローカル関数 `isM4` 、 `isM100` 、 `isM400` を定義し、それらでうるう年を判定するコード

``` idris
isLeapYear : Integer -> Bool
isLeapYear y = isM4 y && not (isM100 y) && isM400 y
where
  isM4 : Integer -> Bool
  isM4 y = y `mod` 4 == 0

  isM100 : Integer -> Bool
  isM100 y = y `mod` 100 == 0

  isM400 : Integer -> Bool
  isM400 y = y `mod` 400 == 0
```

## 無名関数

`\引数 => 式` の構文で作れます。

例： `double` を値を2倍にする無名関数に束縛し、それを2回適用することで値を4倍にするコード

``` idris
quatro: Integer -> Integer
quatro n = let double = \i => i * 2 in
           double (double n)
```

# 制御構造
## `if`

`if` 式は `if 条件 then then節 else else節` の構文をしています。

例：

``` idris
if n == 0
then "Zero"
else "Not zero"
```

Idrisは式指向言語なので `if` も値を返します（なので `if` 「式」と呼ばれます）。他の言語でいういわゆる三項演算子のようなものは必要ありません。

例： `if` が式であることを使って `if` の返り値をそのまま別の関数に渡すコード

``` idris
main : IO ()
main =
  putStrLn (if 1 == 0 then "Zero" else "Not zero")
```

## パターンマッチ

パターンマッチは `case 条件 of パターン => 式 ...` の構文をしています。
`パターン => 式` の部分にはオフサイドルールが適用されます。


例：

``` idris
case n of
  1 => "one"
  2 => "two"
  3 => "three"
  _ => "many"
```

最後の `_` は特殊なパターンで、どんな値にもマッチしてその値を無視します。

パターンマッチはもうちょっと複雑なこともできます。値の構造がパターンに合致すればマッチ成功となります。さらに、その値を変数に束縛できます。

例：リストに対してパターンマッチし、分配束縛するコード。

``` idris
case list of
  [] => 0
  [x, y] => x + y
  [x, y, z] => x + y + z
  _ => -1
```

このコードに登場する `list` が `[1, 2, 3]` に束縛されているならば `[x, y, z]` の節にマッチして `x + y + z` 、つまり `1 + 2 + 3` が計算されて6が返ります。

### 関数の引数でのパターンマッチ

関数の引数でもパターンマッチが可能です。関数の名前を連ねる形になります。

例：$n$ 番目の[フィボナッチ数](https://ja.wikipedia.org/wiki/フィボナッチ数)を計算するコード

``` idris
fib: Integer -> Integer
fib 0 = 1
fib 1 = 1
fib n = fib (n - 1) + fib (n - 2)
```


## ループ、 `return` 、 `break`
ないよ。

Idrisは関数型言語なのでループは関数を使います。自身を呼び出すことでループを作れるのです（再帰関数）。`break` や `return` は必要ありません。自身を呼び出すことをやめれば自然とループが止まりますし、その場で値が返ります。

例： `1` から `n` までの和を求める関数

``` idris
sumFromOne: Integer -> Integer
sumFromOne n = loop 1 n 0
where
  loop: Integer -> Integer -> Integer -> Integer
  loop i end sum =
    let sum = sum + i in
    if i == end
    then sum
    else loop (i + 1) end sum
```

関数 `sumFromOne` の定義でローカル関数として `loop` を定義しています。この関数を再帰呼び出しすることでループを実現します。 `loop i end sum = ...` ではじまって `loop (i + 1) end sum` を呼んでいるので `i` を1つづつ増やしていっているのが読み取れるでしょうか。最初が `loop 1 n 0` なので `loop 2 n sum` 、 `loop 3 n sum` …と呼び出していって `loop n n sum` まできたところでループが終了します。

# 演算子

Idrisには組み込みの演算子がありません。もうちょっというと演算子というものはありません。代わりに、関数を中置記法で書けるようにする方法が2つあります。ユーザで中置演算子を自由に作れる訳ですね。今まで使ってきた `+` なんかもこれに該当します。

2つの方法のうち1つ目の方法が `` ` `` 〜 `` ` `` で関数を囲むもの。

例： 2引数関数 `add` を `` ` `` 〜 `` ` `` で囲んで中置演算子として使うコード

``` idris
add : Integer -> Integer -> Integer
add x y = x + y

1 `add` 2
```

逆に、中置演算子は `()` で囲って普通の関数のように使うこともできます。

``` idris
(+) 1 2
```

2つ目の方法が `infix` 系の構文です。
`infix` 、 `infixl` 、 `infixr` があります。
`infixl 4 +,-` のように `infix 優先順位 演算子1,演算子2..` と書きます。

``` idris
infixl 4 -?
prefix 2 -!

(-?) : Integer -> Integer -> Integer
(-?) x y = x - y

(-!) : Integer -> Integer
(-!) x = 0 - x


-! 1 -? 2 -? 3
-- これは -! ((-?) ((-?) 1 2) 3) と解釈されて4になる
```

## セクション

中置演算子を部分適用したいことがたまにあります。例えば `1 +` と書いて1を足す関数ができてほしいですよね。実際、そういう使い方ができます。

``` idris
-- 1を足す関数(1+)を2に適用
(1+) 2
-- -> 3


-- 3で割った余りを返す関数(`mod` 3)を5に適用
-- (`mod` 3) 5
-- -> 2
```

覚えておくと便利な構文ですね。


# データ型

`data 名前 = 定義` の構文で定義します。定義のところには `ヴァリアント | ヴァリアント…` と書きます。ヴァリアントには `コンストラクタ 引数の型 …` と書きます。ヴァリアントは少なくとも1つ、コンストラクタの引数は0以上を書きます。

例：引数のないコンストラクタのヴァリアントを2つ持つデータ型

``` idris
data Bool = True | False
```



例：引数の2つあるコンストラクタのヴァリアントを1つ持つデータ型

``` idris
data Person = MkPerson Int String
```

引数ありのコンストラクタのヴァリアントを1つ持つデータ型は頻出パターンで、構造体のように使えます。
そのときのコンストラクタが構造体のコンストラクタのようになります。こういうときは `MkHoge` と `Mk` （makeの略）を前置するのが慣例です。


例： 引数のあるコンストラクタや引数のないコンストラクタのヴァリアントのあるデータ型

``` idris
data FizzBuzz = F | B | FB | I Integer
```


データ型は語ることが多いので章を新ためて説明しようと思います。

# Hole（穴）

Idrisの重要かつ初心者にとって役に立つ機能にHole（穴）があります。未完成のプログラムを作ることができるのです。実装が難しい部分をHoleにしておくことで後で実装したり、処理系からHoleについての情報を得たりできます。さらにはエディタサポートも受けられます。

 `?` に続いてシンボルを書くとHoleとして扱われます。例えば以下のように書きます。

例：Holeを使ったプログラム

``` idris
awesomeFunction: String -> Integer
awesomeFunction s = ?hole
```

この `?hole` の部分がholeの記法で、 `hole` という名前のholeを作っています。さらにここから `hole` の方を処理系から取得することもできます。

例：`hole` の型を処理系から取得した際の出力

``` text
    s : String
---------------
 hole : Integer
```

これは `hole` の部分を埋めるにあたって、 `String` 型である `s` という変数が利用できて、 `hole` は `Integer` 型であることが要求される、ということを表しています。

Holeは式の途中で置くことも可能です。例えば以下のように `awesomeFunction` の引数を `arg` というHoleにすることもできます。

例：式の途中にHoleを置くコード

``` idris
awesomeValue : Integer
awesomeValue =
  let tmp = awesomeFunction ?arg
  in tmp + 2
```

先程と同様に `arg` の型を取得することができます。

例：`arg` の型を処理系から取得した際の出力

``` text
-------------
arg : String
```

こちらは引数がないので分数表記の上の方は空ですね。

触れてませんでしたがHoleを使って書いたコードも後の関数から使うことができます。上の `awesomeValue` がそうですね。このコードを読み込むとコンパイラがHoleについての情報を出すのでHoleを埋め忘れる心配はありません。

例：Holeのあるプログラムを読み込んだときの出力

``` text
Holes: Main.arg, Main.hole
```

Holeについてはまだまだ語るところがあるのですが構文の学習からは少し離れてしまうのでこのあたりにしておきましょう。

# まとめ

Idrisの基本文法を学びました。
