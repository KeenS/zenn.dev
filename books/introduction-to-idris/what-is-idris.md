---
title: "Idrisってどんな言語？"
---

まずは肩肘張らずにIdrisがどんな言語か雰囲気を掴みましょう。

# コード例

ひとまずコードを見てみましょう。


## Hello World

``` idris:HelloWorld.idr
main : IO ()
main = putStrLn "Hello World"
```

``` shell-session:ターミナル
$ idris HelloWorld.idr -o HelloWorld
$ ./HelloWorld
Hello World
```


## FizzBuzz

``` idris:FizzBuzz.idr
import Data.String

data FizzBuzz = F | B | FB | I Integer

Show FizzBuzz where
  show F     = "fizz"
  show B     = "buzz"
  show FB = "fizzbuzz"
  show (I n)  = show n

fizzBuzz : Integer -> FizzBuzz
fizzBuzz n = case (n `mod` 3, n `mod` 5) of
               (0, 0) => FB
               (_, 5) => B
               (0, _) => F
               _      => I n
fizzBuzzSeq : Integer -> List FizzBuzz
fizzBuzzSeq n = map fizzBuzz [1..n]

main : IO ()
main = do
  [_, arg] <- getArgs
    | _ => putStrLn "prease specify N"
  let Just n = parseInteger arg
    | Nothing => putStrLn "arg must be an integer"
  for_(fizzBuzzSeq n) (putStrLn . show)
```


``` shell-session:ターミナル
$ idris FizzBuzz.idr -o FizzBuzz
$ ./fizz_buzz 15
1
2
fizz
4
5
fizz
7
8
fizz
10
11
fizz
13
14
fizzbuzz
```

# REPL

IdrisにはREPL（インタラクティブシェル）もあります。 `Idris>` で始まる行が入力で、続く行が出力です。

``` shell-session
$ idris
     ____    __     _
    /  _/___/ /____(_)____
    / // __  / ___/ / ___/     Version 1.3.3
  _/ // /_/ / /  / (__  )      https://www.idris-lang.org/
 /___/\__,_/_/  /_/____/       Type :? for help

Idris is free software with ABSOLUTELY NO WARRANTY.
For details type :warranty.
Idris> 1 + 1
2 : Integer
Idris> "Hello, " ++ "REPL"
"Hello, REPL" : String
```

# Idrisとは
[Idris](https://www.idris-lang.org/index.html)とは *型駆動開発* のために設計されたプログラミング言語です。静的型付きの関数型言語で、コンパイル方式の処理系を持ちます。

大きな特徴としては型駆動開発のために強力な型、特にプログラミング言語としては珍しい依存型を持つこと、文法がHaskellに似ていることが挙げられます。

色々キーワードが出てきましたが本書の続きで追い追い紹介するとして、ここでは依存型とは何かを紹介します。

# 依存型とは

依存型とは項でインデックス付けされた型です（TaPLより）。

ざっくりと言うと型を書く場所に値を書けます。

例えばこういう関数の実装の型を考えてみましょう。

``` idris
foo True  = "True"
foo False = 0
```

引数が `True` のときに `String` 型の値を返して、 `False` のときに `Integer` 型の値を返しています。これは大抵の言語では型づけできません[^ts]。

[^ts]: TypeScriptのように型付けできる変態もいますが…

しかしIdrisなら簡単に型付けできます。まさしく「引数が `True` のときに `String` 型、 `False` のときに `Integer` 型」と記述するだけです。

``` idris
foo: (b: Bool) -> if b then String else Integer
foo True  = "True"
foo False = 0
```

どうですか？面白くないですか？

もうちょっとユースケースが分かりやすい例に[ベクタ型](https://www.idris-lang.org/docs/current/base_doc/docs/Data.Vect.html)などもあります。リスト型のようですが、 `Vect n a` と型引数にその長さ `n` を保持できます。

例えばそのベクタを結合する関数 `++` は以下のような型をしています。

``` idris
(++) : (xs : Vect m elem) -> (ys : Vect n elem) -> Vect (m + n) elem
```

長さ `m` のベクタと長さ `n` のベクタを結合すると長さ `m + n` のベクタになるというシンプルですが強力な表明が書けます。

あるいは要素 `n` 個を取り出す関数 `take` は以下のような型をしています。

``` idris
take : (n : Nat) -> Vect (n + m) elem -> Vect n elem
```

`n` 個取り出すからには `n` 個以上のベクタを渡さないといけないんですね。

このように詳細な制約を書けるのが特徴です。もう一歩進めるとこの制約を使って数学的な証明を書いたりもできますし、プログラムにバグがないことも証明できます。

「え〜、バグがないことの証明？ウソくさ〜」って思ったかもしれません。普段のプログラミングでは頑張ってテストを書いてもせいぜいバグがなさそうと分かるだけで、証明なんてできませんよね。ですがIdrisのように理論的背景のある言語だとこれが大真面目にできるんです。原理とかを説明し始めると長くなってしまうので証明の章までとっておくことにしましょう。


# 本章のまとめ

Idrisについてかるーく紹介しました。
