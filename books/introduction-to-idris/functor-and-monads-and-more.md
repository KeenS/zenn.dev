---
title: "ファンクタやモナドなどなど"
---

本章ではIdrisを学ぶ上で重要かつ関門となる機能、モナドに関連する機能について学びます。


# 高カインド多相とファンクタ

モナドを理解するのに一番難しいのが高カインド多相（高階多相）です。特に複雑という訳ではないんですが、抽象度が高いので慣れてないと理解に時間のかかる機能です。

`List a` や `Maybe a` のようにジェネリクスなデータ型がありますね？これに対してインタフェースを定義したいとします。例えば `map` なんかは分かりやすいでしょう。

例： `List a` と `Maybe a` に対する素朴な `map` の実装


``` idris
-- List
map : (a -> b) -> List a -> List b
map f []      = []
map f (x::xs) = f x :: map f xs

-- Maybe
map : (a -> b) -> Maybe a -> Maybe b
map f (Just x) = Just (f x)
map f Nothing  = Nothing
```

これを抽象化するインタフェースを定義したいです。するとパラメータになるのは `List` や `Maybe` の部分です。これらは型コンストラクタ、Idris的にいうと `Type -> Type` の値です。

``` idris
Idris> :t List
List : Type -> Type
Idris> :t Maybe
Maybe : Type -> Type
```

従来の型変数（Idris的にいうと `Type` の値）とは異なるのでインタフェースの定義に少し手を加えます。具体的には型変数が `Type` ではなく `Type -> Type` であることを表わすために型注釈を加えます。構文は `interface インタフェース名 (変数名: Type -> Type) where 本体` です。

例： プレリュードのインタフェース `Functor` の定義


``` idris
interface Functor (f : Type -> Type) where
    map : (func : a -> b) -> f a -> f b

```

`Functor` （関手）は `List` や `Maybe` などのように「`map` できる型」を抽象化する型です。`f` の部分に `List` や `Maybe` などが当て嵌ります。

`Functor` インタフェースのおかげでこのように `List` や `Maybe` の値に対して1を足す関数を適用できます。

``` text
Idris> map (1+) (Just 1)
Just 2 : Maybe Integer
Idris> map (1+) [1, 2, 3]
[2, 3, 4] : List Integer
```

このように `Type` ではなく `Type -> Type` などの複雑な「型の型」を持つものに対するジェネリクスを高カインド多相と呼びます（「型の型」はカインド（Kind）と呼ばれています）。

因みに `map` の代わりに `<$>` という演算子も使えます。 `<$>` はプレリュードで以下のように定義されている演算子です。

``` idris
infixr 4 <$>
(<$>) : Functor f => (func : a -> b) -> f a -> f b
func <$> x = map func x
```

以下のように使います。

``` text
Idris> (1+) <$> (Just 1)
Just 2 : Maybe Integer
Idris> (1+) <$> [1, 2, 3]
[2, 3, 4] : List Integer
```

後述するApplicativeと組み合わせるときに便利です。

余談ですがHaskellだとリスト専用の `map` と `Functor` で定義される `fmap` で分かれています。恐らくですが先にリストの `map` を作ったあとに `Functor` という抽象化に気付いたので要らぬ複雑性が入ってるんじゃないかと思います。その点IdrisはHaskellの後発なのもあってシンプルですね。

# 多引数関数とApplicative

`map` は便利ですが、痒いところに手が届かないことがあります。多引数関数には使いづらいのです。

例えば2引数関数 `(+)` を `Just 1` と `Just 2` に適用したいとしましょう。そこで素朴に `map` で適用しようとするとエラーになります。

例： `map` を使って素朴に `(+)` を `Just 1` と `Just 2` に適用した式

``` text
Idris> map (+) (Just 1) (Just 2)
(input):1:1-25:When checking an application of function Prelude.Functor.map:
        Type mismatch between
                Maybe a1 (Type of Just x)
        and
                (\uv => _ -> uv) a (Expected type)

        Specifically:
                Type mismatch between
                        Maybe
                and
                        \uv => _ -> uv
```

これは落ち着いて型を考えるとエラーになる理由が分かります。（`Maybe` に対する）`map` の型は `(a -> b) -> Maybe a -> Maybe b` です。 `(+)` の型は `Integer -> Integer -> Integer` で、これは `Integer -> (Integer -> Integer)` です。これらを組み合わせると、 `map (+)` は `Maybe Integer -> Maybe (Integer -> Integer)` になります。これを `Just 1` に適用すると `Maybe (Integer -> Integer)` になります。ここで関数ではなくて `Maybe` 型の値が出てきてしまいました。これでは `Just 2` に適用できません。

しかしながらみなさんは無理矢理適用させる実装を思い付くんじゃないでしょうか。以下のようにパターンマッチで取り出してしまえばいいのです。

例： `Maybe` に包まれた関数を無理矢理適用してしまうコード

``` idris
ap: Maybe (a -> b) -> Maybe a -> Maybe b
ap (Just f) (Just x) = Just (f x)
ap _        _        = Nothing
```

実現できそうなのでインタフェースで抽象化しましょう。 `Functor` を拡張したインタフェースにするのが具合がよさそうです。これはプレリュードで `Applicative` と呼ばれています。

例：プレリュードの `Applicative` の定義

``` idris
infixl 3 <*>
interface Functor f => Applicative (f : Type -> Type) where
    pure  : a -> f a
    (<*>) : f (a -> b) -> f a -> f b
```

`ap` ではなく `<*>` という演算子になっていますが、やってることは先程定義した `ap` と同じものです。

これに対する `Maybe` の実装は以下のようになっています。

例：プレリュードの `Applicative` の `Maybe` への実装

``` idris
Applicative Maybe where
    pure = Just

    (Just f) <*> (Just a) = Just (f a)
    _        <*> _        = Nothing
```

ちゃんと `<*>` の実装が `ap` と同じものになっていますね。

`map` を `<$>` と書けることと組み合わせて、以下のように使えます。

例： `Functor` と `Applicative` の利用

``` idris
Idris> (+) <$> (Just 1) <*> (Just 2)
Just 3 : Maybe Integer
```

`Applicative` はインタフェースであるからには複数の型（型コンストラクタ）に実装されている訳です。例えば `List` での実装がどうなっているかというと、全ての要素に対して繰り返すようになっています。

例： `List` での `Functor` と `Applicative` の利用

``` idris
Idris> (+) <$> [1, 2, 3] <*> [10, 11, 12]
[11, 12, 13, 12, 13, 14, 13, 14, 15] : List Integer
```

ところで `Applicative` に `pure` というのがいますね。これの役割に触れておきましょう。`func` を `x` に適用するとします。`func` と `x` の型がそれぞれ `Maybe` （一般化して `f`）に包まれている/いないで4つの組み合わせがありますね？それぞれどう適用するか見てみましょう。

| 関数         |  引数 |     適用     |
|--------------|-------|--------------|
| `a -> b`     | `a`   | `func x`     |
| `a -> b`     | `f a` | `map func x` |
| `f (a -> b)` | `a`   | ????         |
| `f (a -> b)` | `f a` | `func <*> x` |

`f (a -> b)` を `a` に適用する場合だけまだ出てきてませんね。この隙間を `pure` が埋めてくれます。 `func <*> (pure x)` と書けばいいのです。

気付いた方もいるかもしれませが `Functor` を使った `map func x` も `(pure func) <*> x` と `Applicative` の機能だけで書くことができますね。そういった意味で `Applicative` は `Functor` の拡張になっています。

# プログラムとモナド

`Applicative` で `Maybe` などのジェネリクス型に包まれているデータや関数に対して操作できるようになりました。では新しく包む操作についてはどうでしょう。

例えば割る数が0以外では割った商を、0では `Nothing` を返す `safeDiv` を考えます。

例：`safeDiv` 関数

``` idris
safeDiv : Integer -> Integer -> Maybe Integer
safeDiv _ 0 = Nothing
safeDiv d m = d `div` m
```

新たに `Maybe` で包んで返す関数を `<$>` や `<*>` と組み合わせることもできますが、結果はあまり嬉しくありません。

例：`safeDiv` を `Functor` と `Applicative` と一緒に使った例

```text
Idris> safeDiv <$> (Just 1) <*> (Just 0)
Just Nothing : Maybe (Maybe Integer)
```

返り型が `Maybe (Maybe Integer)` と `Maybe` が二重に出てきてしまいました。そして返り値も `Just Nothing` になってしまっています。これは数値が返ってきていないという意味では `Nothing` と変わりません。`Maybe Integer` に「潰せ」たらうれしいですよね。

そういう操作は一般に、flatten、あるいはjoinと呼ばれますね。さらにflattenの派生型であると嬉しいのがflatMapです。Idrisの型で書くと `Maybe a -> (a -> Maybe b) -> Maybe b` です。flatten（join）とflatMapは片方があればもう片方を定義できるので双子のような存在です。

そんなjoinとflatMapをインタフェースにしたのがモナドです。

例：プレリュードでの `Monad` の定義

```Idris
infixl 1 >>=

interface Applicative m => Monad (m : Type -> Type) where
    ||| Also called `bind`.
    (>>=)  : m a -> ((result : a) -> m b) -> m b

    ||| Also called `flatten` or mu
    join : m (m a) -> m a

    -- default implementations
    (>>=) x f = join (f <$> x)
    join x = x >>= id
```

この `Monad` を使えば先程の `safeDiv` は `Maybe Integer` を返すように使えるようになります。

```text
Idris> join (safeDiv <$> (Just 1) <*> (Just 0))
Nothing : Maybe Integer
Idris> Just 1 >>= \d => (Just 0 >>= \m => safeDiv d m)
Nothing : Maybe Integer
```

返ってくる値は望み通りなのですが今度は `>>=` やラムダ式などが乱舞して読みづらくなりました。これを改善する構文がIdrisにはあります。

## `do` 記法

先程の例、 `Just 1 >>= \d => (Just 0 >>= \m => safeDiv d m)` は見づらいですよね。演算子や無名関数が乱舞してどこに何が書いてあるのか分かりません。そこでこれを書きやすくする `do` 記法というのがあります。do記法は `do` に続いてオフサイドルールでブロックを書きます。ブロックで書ける構文は `変数 <- 式` や `式` 、 `let 変数 = 式` などいくつかあります。

先程のコードを `do` 記法で書き直すと以下のようになります。

例： `safeDiv` を `Maybe` の値に適用する計算を `do` 記法で書いたコード

``` idris
do
  d <- Just 1
  m <- Just 0
  safeDiv d m
```

これだとぐっと見やすくなりますね。 `変数 <- 式 次の行` が `式 >>= \変数 => 次の行` と同じ意味になります。 `Just 1 >>= \d => (Just 0 >>= \m => safeDiv d m)` と対応をとってみると分かりやすいかもしれません。

ところで基本文法のところで触れ忘れたんですが、オフサイドルールには別の記法もあります。`{記述1; 記述2; ...}` と `{}` で包んでそれぞれの記述を `;` で分ける記法です。こうすることで1行でも書けるようになります。REPLなどでは1行で書きたいケースもあると思うのでお試し下さい。

例： `do` 記法をREPLで使うコード

``` text
Idris> do {d <- Just 1; m <- Just 0; safeDiv d m }
Nothing : Maybe Integer
```

モナドを使うときは大抵 `do` 記法を使うことになるでしょう。

`Monad` もまたインタフェースなので `Maybe` 以外の型も実装を持ちます。例えば `List` はその要素について繰り返します。リスト内包表記のようなことを `do` 記法でもできるのです。

例：九九の左斜め下半分を `do` 記法で計算するコード

``` idris
do
  x <- [0..9]
  y <- [0..x]
  pure (x * y)
```

# モナドはDSL?

`Functor` 、 `Applicative` 、 `Monad` で何かに包まれた値を計算できるようになりました。では、包まれた値から取り出すにはどうしたらいいでしょう。残念ながらいい方法はありません。 `Nothing` なんかは値がないから `Nothing` な訳で、そこから値を取り出せません。

逆に言うとモナドにすることで操作を「閉じ込めて」しまうことができます。閉じ込めることでモナドの作者が想定した使い方しかできないようにできます。使える操作は `Functor` と `Applicative` で「持ち上げた」操作と、 `Monad` で「結合」できる `a -> m b` の型の関数のみです。

そういった意味でモナドはDSLと捉えることができます。ライブラリなんかでもモナドを提供し、主な操作は `do` 記法でやるものが多くあります。


# IOモナド

IOモナドはIdrisで一番重要なモナドなのでここで同時に触れます。いままで、まともにHollo Worldを解説してませんでしたね。それはIO操作もモナドで書かれているからです。

ということでモナドを知った今、改めてHello Worldをしてみましょう。 `putStrLn` は以下のような型をしています。

``` text
Idris> :t putStrLn
putStrLn : String -> IO ()
```

そしてIdrisは `main : IO ()` な値からプログラムの実行を始めます。何度か出てきたHello Worldを改めて見直しましょう。

例：IdrisでのHello World

``` idris:HelloWorld.idr
main : IO ()
main = putStrLn "Hello, World"
```

`main` を `IO` モナドの値に束縛しています。これを `Hello.idr` として保存し、以下のように実行します。

例：Hello Worldをコンパイル・実行するコマンド

``` shell-session
$ idris -o HelloWorld HelloWorld.idr
$ ./HelloWorld
Hello, World
```

`-o` オプションをつけて `idris` コマンドを起動するとREPLではなくコンパイラが起動し、 `-o` で指定したファイルへとコンパイル結果を出力します。

`IO` モナドを使ってもうちょっと複雑なことをしましょう。 `getLine: IO String` で標準入力から1行取得できます。これと `putStrLn` で入力をエコーバックするプログラムはこう書けます。

例： `getLine` と `putStrLn` を使ってユーザの入力を表示するプログラム

``` idris:Echo.idr
main : IO ()
main = getLine >>= \s => putStrLn ("Your input is " ++ s)
```

あるいは、 `do` 記法でこう書くこともできます。

例： `getLine` と `putStrLn` を使ってユーザの入力を表示するプログラムを `do` 記法で書いたもの

``` idris:Echo.idr
main : IO ()
main = do
  s <- getLine
  putStrLn ("Your input is " ++ s)
```

これを `Echo.idr` に保存し、コンパイル、実行すると以下のように動作します。

``` shell-session
$ idris -o Echo Echo.idr
$ ./Echo
echooooo
Your input is echooooo
```

IOモナドを使うことで出入力ができることが分かりました。

## ところでIOって何？

`IO` の型にちょっと違和感を覚えた方もいるんじゃないかと思います。 `main` の型は `IO ()` という値です。関数じゃありません。同じく `getLine` も `IO String` という値です。これだと書いたそばから実行されてしまわないでしょうか。まあ、動いてるからにはそうならないんのは分かるんですが、どういう仕組みなんでしょう。

実はIdrisのプログラムからは `IO` の値を実行することができません。 `getLine` と書いたからといって即座に標準入力から文字列を取り出したりしないのです。唯一 `main` に書いた `IO` の値のみが処理系側で実行されます。処理系側で実行されてはじめて標準入力から文字列を取り出すというアクションが行なわれます。 `IO` は実行される前のプログラムのようなものなのです。

`IO` を実行できるのは `main`の1箇所のみとなると、複数のIO処理をしたいときは `IO` の値を合成する必要があります。その仕組みに選ばれたのがモナドという訳です。`>>=` は別名 bind （結合）ですが、先程の `getLine` と `putStrLn` のように複数のIO処理を結合するのに使われているのです。

## 純粋関数型言語とIO

さて、 `main` でしか `IO` を実行できないとなると他の関数内でIO処理をしたい場合はどうすればいいのでしょう。

1つの答えは「そういう関数は設計が悪いから書くな」です。純粋関数型言語であるIdrisの基本方針として、IOや変数への破壊的代入などの計算以外の処理はよくないものとされています。関数を呼んだときに何が起こるか分からなくなるからです。なので関数内でIO処理を書きたくなったときはまずは「計算部分とIO部分に分離できないか」と考えてみましょう。

もう1つの答えは 「全て `IO` モナドの中で書く」です。 `IO` モナドをリレーのように `main` まで伝えればIOを実行できます。なので関数の中でIOをしたければ `IO` モナドの中で書くことにすれば実現できます。とはいえやっぱりIOの中でプログラムを書くのは面倒なので基本的には純粋な計算部分とIO部分に分けて、IO部分でだけ `IO` モナドを使うようになります。

じゃあデバッグプリントを関数の中に仕込みたかったらどうなるの、などの疑問はあるかもしれません。まあ、普通に `IO` を使ってそれを呼ぶ関数を全て `IO` モナドの中で書くように変更します。しかしちょっと面倒ですよね。一応そういった用途のためのバックドアの機構があるにはあります。後の章で紹介することにしましょう。

# 本章のまとめ

Idris入門の最難関、高カインド多相と `Monad` などのいくつかのインタフェースについて学びました。さらに重要なモナドであるIOモナドについても学びました。ちょっと駆け足で分かりづらかったかもしれませんが頭で理解するより使って馴染んだ方が分かりやすいと思うので分からなければそのまま進んでみて下さい。
