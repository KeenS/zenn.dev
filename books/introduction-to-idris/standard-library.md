---
title: "標準ライブラリざっと見"
---

Idrisの標準ライブラリであるプレリュードとbaseをサクっと眺めていきます。名前空間についてはまだ学んでないのでそれぞれのタイトルにある名前空間はあまり気にせず進めていって下さい。

# プレリュード
## `Builtins`

コンパイラから特別扱いされている型などが入っています。例えば `()` に対応する `Unit` 型や `(A, B)` に対応する `Pair` 型など。

まだ紹介していないものをいくつか紹介しましょう。

* `Inf : Type -> Type`: （遅延した）無限の計算を表わす。`Stream` ででてくる。
* `replace : {a:_} -> {x:_} -> {y:_} -> {P : a -> Type} -> x = y -> P x -> P y`: `x = y`ならば `P x` を `P y` に書き換えられる。証明向け。
* `sym : {left:a} -> {right:b} -> left = right -> right = left`: 等式の左右を入れ替える。 `rewrite sym $ ... in ...` の形で使うことが多い。証明向け。

## `IO`

IOモナド関連の内部実装。直接お世話になることは少ないですかね。

## `Prelude.Algebra`

2つのインタフェースが定義されています。

```idris
interface Semigroup ty where
  (<+>) : ty -> ty -> ty

interface Semigroup ty => Monoid ty where
  neutral : ty
```

それぞれ[半群](https://ja.wikipedia.org/wiki/半群)と[モノイド](https://ja.wikipedia.org/wiki/モノイド)です。

特に `Semigroup` に定義されている二項演算は強力で、リストの結合など、色々な操作を抽象化しています。

```text
Idris> [1, 2, 3] <+> [4, 5, 6]
[1, 2, 3, 4, 5, 6] : List Integer
```

## `Prelude.Functor`

ファンクタが定義されています。

```idris
interface Functor (f : Type -> Type) where
    map : (func : a -> b) -> f a -> f b
```

## `Prelude.Applicative`

既に紹介した `Applicative` が定義されています。

```idris
infixl 3 <*>

interface Functor f => Applicative (f : Type -> Type) where
    pure  : a -> f a
    (<*>) : f (a -> b) -> f a -> f b

```

もう1つ重要なのが、 `Alternative` です。

```idris
infixr 2 <|>
interface Applicative f => Alternative (f : Type -> Type) where
    empty : f a
    (<|>) : f a -> f a -> f a
```

例えば `Maybe` のように失敗するかもしれない計算上で `||` 演算子のような働きをします。

```text
Idris> Just 1 <|> Just 2
Just 1 : Maybe Integer
Idris> Just 1 <|> Nothing
Just 1 : Maybe Integer
Idris> Nothing <|> Just 1
Just 1 : Maybe Integer
Idris> the (Maybe Int) Nothing <|> Nothing
Nothing : Maybe Int
```

他には `guard` や `when` などの処理もあります。

```idris
guard : Alternative f => Bool -> f ()
guard a = if a then pure () else empty

when : Applicative f => Bool -> Lazy (f ()) -> f ()
when True f = Force f
when False f = pure ()
```

これらはモナドと一緒に使うと便利です。

```idris
do
  n <- [1..10]
  guard $ n `mod` 2 == 0
  pure n
-- [2, 4, 6, 8, 10] : List Integer
```

## `Prelude.Monad`

モナドが定義されています。

```idris
infixl 1 >>=

interface Applicative m => Monad (m : Type -> Type) where
    (>>=)  : m a -> ((result : a) -> m b) -> m b

    join : m (m a) -> m a

    -- default implementations
    (>>=) x f = join (f <$> x)
    join x = x >>= id
```

## `Prelude.Foldable`

`foldl` と `foldr` 、他の言語でいうIterable相当のインタフェースを提供します。

```idris
interface Foldable (t : Type -> Type) where
  foldr : (func : elem -> acc -> acc) -> (init : acc) -> (input : t elem) -> acc

  foldl : (func : acc -> elem -> acc) -> (init : acc) -> (input : t elem) -> acc
  foldl f z t = foldr (flip (.) . flip f) id t z
```

イテレータにありがちな操作も提供しています。

```idris
concat : (Foldable t, Monoid a) => t a -> a
concat = foldr (<+>) neutral

concatMap : (Foldable t, Monoid m) => (a -> m) -> t a -> m
concatMap f = foldr ((<+>) . f) neutral

and : Foldable t => t (Lazy Bool) -> Bool
and = foldl (&&) True

or : Foldable t => t (Lazy Bool) -> Bool
or = foldl (||) False

any : Foldable t => (a -> Bool) -> t a -> Bool
any p = foldl (\x,y => x || p y) False

all : Foldable t => (a -> Bool) -> t a -> Bool
all p = foldl (\x,y => x && p y)  True
```

`concat` は先程紹介したとおり `List` なんかが `Monoid` を実装しているので以下のように使えます。

```text
Idris> concat [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
[1, 2, 3, 4, 5, 6, 7, 8, 9] : List Intege
```


あとは `Alternative` を使ったAPIもあります。

```idris
choice : (Foldable t, Alternative f) => t (f a) -> f a
choice x = foldr (<|>) empty x

choiceMap : (Foldable t, Alternative f) => (a -> f b) -> t a -> f b
choiceMap f x = foldr (\elt, acc => f elt <|> acc) empty x
```

これは最初の成功した値を取り出します。

```text
Idris> choice [Nothing, Nothing, Just 1, Nothing, Just 2]
Just 1 : Maybe Integer
Idris> the (Maybe Int) $ choice [Nothing, Nothing, Nothing]
Nothing : Maybe Int
```

## `Prelude.Basics`

かなり基本的な操作が入っています。

```idris
Not : Type -> Type
Not a = a -> Void

id : a -> a
id x = x

the : (a : Type) -> (value : a) -> a
the _ = id

const : a -> b -> a
const x = \value => x

-- タプルの左側を取り出す
fst : (a, b) -> a
fst (x, y) = x

-- タプルの右側を取り出す
snd : (a, b) -> b
snd (x, y) = y
```

`(.)` は関数合成をします。

```idris
infixr 9 .

(.) : (b -> c) -> (a -> b) -> a -> c
(.) f g = \x => f (g x)
```

`flip` は2引数関数の引数の順序を入れ替えます。`(.)` と組み合わせて使うときなんかに便利ですね。

```idris
flip : (f : a -> b -> c) -> b -> a -> c
flip f x y = f y x
```

## `Prelude.Bits`

`Bits8` などのビット数指定の型の軽い処理が入っています。


## `Prelude.Cast`

`Cast` インタフェースが定義されています。

```idris
interface Cast from to where
    cast : (orig : from) -> to
```

以外な型同士にも `Cast` が定義されているのは紹介したとおりです。

```idris
Cast Double String where
   -- ...
```


## データ型
`Prelude.Bool`, `Prelude.Chars` `Prelude.Either`, `Prelude.Double`, `Prelude.List`, `Prelude.Maybe`, `Prelude.Nat`, `Prelude.Stream`, `Prelude.String`

それぞれのデータ型や関連するコードが定義されています。このうち `Stream` だけ触れてないので紹介します。

### `Stream`

リストと似ていますが、終わりのないデータ型です。

```idris
data Stream : Type -> Type where
  (::) : (value : elem) -> Inf (Stream elem) -> Stream elem
```

例えば無限に1が続くデータ型なんかを作れます。

```text
Idris> :let ones = the (Stream Int) $ repeat 1
Idris> take 10 ones
[1, 1, 1, 1, 1, 1, 1, 1, 1, 1] : List Int
```

素数一覧みたいに無限に続くデータ型をひとまず作っておいて、必要になったら `take` や `index` などで取り出すのが主な使い方ですかね。それこそFizzBuzzなんかもそうですね。

## `Prelude.File`

ファイル操作が入っています。

## `Prelude.Interactive`

`putStrLn` などが入っています。

## `Prelude.Interfaces`

`Eq` などの基本的なインタフェースが入っています。

## `Prelude.Show`

`Show` インタフェースが定義されています。

```idris
interface Show ty where
  partial
  show : (x : ty) -> String
  show x = showPrec Open x -- Eta expand to help totality checker

  partial
  showPrec : (d : Prec) -> (x : ty) -> String
  showPrec _ x = show x
```


## `Prelude.Traversable`

`Foldable` と `Applicative` を組み合わせるときに使います。 `Foldable` の要素の1つ1つに操作を加えます。

```idris
interface (Functor t, Foldable t) => Traversable (t : Type -> Type) where
  traverse : Applicative f => (a -> f b) -> t a -> f (t b)
```


まあ、一番分かりやすいのは `for_` 関数ですかね。

```idris
for_ : (Foldable t, Applicative f) => t a -> (a -> f b) -> f ()
for_ = flip traverse_
```

こう使います。

```idris
Idris> :exec for_ [1..10] $ \n => printLn n
1
2
3
4
5
6
7
8
9
10
```


# base

baseの方が複雑なんですが便利なライブラリがあるんですよね。頑張って紹介します。

## `Control.Catchable`

失敗するかもしれない計算の失敗を処理する機能を提供します。

```idris
interface Catchable (m : Type -> Type) t | m where
    throw : t -> m a
    catch : m a -> (t -> m a) -> m a
```

`catch` を 中置記法で使うとそれっぽいですかね。

```text
*Control/Catchable> Right "Correct" `catch` \_ => Left "Failed"
Right "Correct" : Either String String
*Control/Catchable> the (Either String String) $ Left "Incorrect" `catch` \_ => Left "Failed"
Left "Failed" : Either String String
```

## `Control.IOExcept`

`IO (Either err a)` のパターンが頻出なのでそれをまとめた型です。


```idris
record IOExcept' (f:FFI) err a where
     constructor IOM
     runIOExcept : IO' f (Either err a)
```

## `Data.IORef`

変更可能な変数のデータ型を定義しています。

```idris
||| A mutable variable in the IO monad.
export
data IORef a = MkIORef a
```

APIは以下です。

```idris
newIORef : a -> IO (IORef a)
readIORef : IORef a -> IO a
writeIORef : IORef a -> a -> IO ()
modifyIORef : IORef a -> (a -> a) -> IO ()
```

以下のようにして使います。

```idris
do
  result <- newIORef 0
  for_ [0..10] $ \n =>
    modifyIORef result (+n)
  readIORef result
-- 55
```

純粋関数型言語の中でどうしても変更可能な値が欲しくなったときに使います。とはいっても `IO` でしか動かないのでIdris自体の純粋性は保たれます。


## `Data.Buffer`

バッファが定義されています。主にファイルとの一括IOで使うのが用途のようです。IdrisでバイナリIOをしたくなると使うことになると思います。

## `Data.Complex`

複素数を定義しています。

```idris
infix 6 :+
data Complex a = (:+) a a
```

## `Data.Vect`

長さの決まっているリストを定義しています。

```idris
data Vect : (len : Nat) -> (elem : Type) -> Type where
  Nil  : Vect Z elem
  (::) : (x : elem) -> (xs : Vect len elem) -> Vect (S len) elem
```

## `Data.Fin`

有限の自然数を定義しています。

```idris
data Fin : (n : Nat) -> Type where
    FZ : Fin (S k)
    FS : Fin k -> Fin (S k)
```

例えば `Fin 3` なら3未満の整数しかとれません。

```text
Idris> :module Data.Fin
*Data/Fin> the (Fin 3) 0
FZ : Fin 3
*Data/Fin> the (Fin 3) 1
FS FZ : Fin 3
*Data/Fin> the (Fin 3) 2
FS (FS FZ) : Fin 3
*Data/Fin> the (Fin 3) 3
(input):1:13:When checking argument prf to function Data.Fin.fromInteger:
        When using 3 as a literal for a Fin 3
                3 is not strictly less than 3
```

`Vect` に対するインデックスのように `len` 未満の値を指定したいときに便利ですね。

```idris
index : Fin len -> Vect len elem -> elem
index FZ     (x::xs) = x
index (FS k) (x::xs) = index k xs
```


## `Debug.*`
`Debug.Error` と `Debug.Trace`はそれぞれデバッグ用に使います。 `IO` 文脈でなくてもIO処理ができてしまう魔法の関数です。


```idris
import Debug.Trace

add : Integer -> Integer -> Integer
add x y = trace "debbuging" $ x + y

main : IO ()
main = printLn $ 3 * (add 1 2)
```


```shell-session
$ idris -o add add.idr
$ ./add
debbuging
9
```

中身は `unsafePerformIO` というヤバい機能で実装されているのでデバッグ用途以外では使わないようにしましょう。

## `System`

`getEnv` や  `exit` などのシステム関連の機能です。

## `System.Info`

`backend` などの処理系情報です。


# 本章のまとめ

Idrisの標準ライブラリをそれぞれ軽く眺めました。いくつか細かいと思ったものは省いたので詳しくは[baseライブラリのドキュメント](https://www.idris-lang.org/docs/current/base_doc/)をあたって下さい。

