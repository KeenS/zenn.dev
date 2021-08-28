---
title: 代数的データ型とパターンマッチ入門
---
SMLにはもっと多くの機能がありますが、さしあたって必要になる知識は揃ったので、次は本題のパターンマッチに進みましょう。
本記事で扱うのはSMLのパターンマッチですが、言語に大きく依存する機能ではないので、他の言語にも当てはまる話が多いはずです。

## 代数的データ型{#sec:ads}

★ メモ: 構築子
★ メモ: 一般論として述べる場合は代数的データ型、SML固有の話をするときはデータ型と呼ぶ。

まず、そもそもパターンマッチとは何かを確認するために、パターンマッチで扱う代数的データ型について説明します。

**代数的データ型**（algebraic data type）とは、直和と直積を組み合わせて定義できるデータ型です。
**直和**は、可能な型をいくつか列挙したものです。
**直積**は、複数の型の値を同時に保持するデータ構造です。
これらを組み合わせて新しい型を定義したものが代数的データ型だといえます。

SMLで代数的データ型を定義するには、`datatype 〈名前〉 = 〈定義〉` という構文を使います。
この構文で、まずは直和の例を見てみましょう。

```{#lst:datatype-week1 numbers=none .sml caption="データ型weekの定義。コンストラクタを「|」で連結すると直和になる"}
datatype week = Sunday | Monday | Tuesday | Wednesday | Thursday | Friday | Saturday
```

[-@lst:datatype-week1]では、`datatype`の`〈定義〉`の部分に、曜日に対応する`Sunday`や`Monday`などを「`|`」で区切っていくつか列挙しています。
このような、代数的データ型の定義に渡すものを **コンストラクタ** と呼びます。
コンストラクタのことをヴァリアント（variant、列挙子とも）と呼ぶ言語もあります。

SMLでは、`Sunday`や`Monday`のように、コンストラクタの先頭を大文字にします。
コンストラクタは、ただの値として扱われるので、[-@lst:datatype-week2]のようにリストに格納したりすることが可能です。

```{#lst:datatype-week2 numbers=none .sml caption="すべてのコンストラクタはweek型なので1つのリストに収められる"}
val weekEnds = [Sunday, Saturday]
```

直和の次は直積です。
代数的データ型の直積は、「複数の型を同時に保持する」ものだと説明しました。
SMLでは、コンストラクタを `〈名前〉 of 〈型〉` のように与えると、指定した型の値を保持できます。
そこに複数の型の値を保持させるには、タプル（など）が使えます。

文字列と数値を保持する `person` 型を定義してみましょう。

```{#lst:datatype-person1 numbers=none .sml caption="データ型personの定義。コンストラクタはMkPersonで、文字列と数値を保持する"}
datatype person = MkPerson of string * int
```

[-@lst:datatype-person1]では、`person`型の定義を`MkPerson of string * int`としています。
このうち、先頭の`MkPerson` の部分は **コンストラクタの識別子**と呼びます。
そのあとの`of` に続く `string * int` の部分が、このコンストラクタの保持する型を表します。
ここでは `string` と `int` のタプル`string * int`としています。

コンストラクタは関数と同じように「適用」ができます。
`MkPerson ("κeen", 26)` では、 `("κeen", 26)` というタプルをコンストラクタに渡して`person`型の値を作っています。

```{#lst:datatype-person2 numbers=none .sml caption="MkPersonに文字列と数値のタプルを渡すとperson型のデータ型が作れる"}
val p1 = MkPerson ("κeen", 26)
```

直積と直和は組み合わせて使えます。
たとえば、次の `cell` というデータ型では、`of`以降のデータを持つコンストラクタ `Full` を直和の要素の1つにしています。

```{#lst:datatype-cell1 .sml caption="データ型cellの定義。値を持つコンストラクタも直和でつなげられる"}
datatype stone = Black | White
datatype cell = Empty | Full of stone

val board = [
    [Empty, Empty,      Empty,      Empty],
    [Empty, Full Black, Full White, Empty],
    [Empty, Full White, Full Black, Empty],
    [Empty, Empty,      Empty,      Empty]]
```

以上の説明をまとめると、代数的データ型は一般には以下のような形で定義されます。

```{.sml numbers=none}
datatype 〈名前〉 = 〈コンストラクタ〉 (| 〈コンストラクタ〉 | … | 〈コンストラクタ〉)

ここで
〈コンストラクタ〉 = 〈名前〉 (of 〈型〉) 
```

直和は互いに排他なので、どちらでもあるような値は作れません。したがって、「 `Sunday` でありかつ `Monday` である」ような値を考慮しなくて済みます。
また、直和は閉鎖的なので、いったん定義したあとに新たなコンストラクタを追加することはできません。
たとえば、 `stone` に存在しない第3のコンストラクタ `Mentos` を投入するとエラーになります。
正しくないプログラムをコンパイル時に検出できるのです。

```{#lst:datatype-cell2 .sml caption="新たに定義したMenthosを投入する（型エラーになる）"}
datatype third = Mentos

val bad_board = [
    [Empty, Empty,      Empty,      Empty],
    [Empty, Full Black, Full White, Empty],
    [Empty, Full White, Full Black, Empty],
    [Empty, Empty,      Empty,      Full Mentos]]
```

上記をコンパイルして実行すると次のようなエラーが表示されるでしょう。

```{.console numbers=none}
(interactive):30.36-30.46 Error:
  (type inference 023) operator and operand don't agree
  operator domain: stone
          operand: third
```

最後に、代数的データ型は再帰的に定義できます。つまり、自身と同じ型をデータに保持できるのです。
たとえば、ファイルシステムの要素を代数的データ型`entry`で表すとしたら、ファイル名を文字列で保持する`File`と、ディレクトリ名の文字列と子エントリのリスト`entry list`を保持する`Directory`との直和として、以下のように定義できます。

```{#lst:datatype-entry .sml caption="entry型の定義"}
datatype entry = File of string | Directory of string * entry list

val fs = Directory (".", [
                       Directory ("src", [File "main.rs", File "lib.rs"]),
                       File "README.md",
                       Directory ("target", [Directory ("debug", [])])
                   ])
```

このように、代数的データ型は、直積と直和を用いて多彩なデータ構造を表現できる仕組みです。

## パターンマッチ

パターンマッチとは、代数的データ型などのデータ型を **パターン** （pattern）を用いて分解し、値を変数に束縛する機能です[^destbind]。

[^destbind]: この機能を**分配束縛** （destructuring bind）と呼ぶ言語もあります。

SMLにおいてパターンマッチが使える箇所はいくつかありますが、まずはcase式を紹介しましょう。

SMLのcase式は次のような構文をしています。

```{.sml numbers=none}
case 〈対象式〉 of 
    〈パターン1〉 => 〈式1〉
  | 〈パターン2〉 => 〈式2〉 
  ...
``` 

case式では、`〈対象式〉` が `〈パターンn〉` にマッチすると、対応する `〈式n〉` が実行されます。
複数のパターンにマッチする場合は、先頭の式が選ばれます。

`〈パターンn〉`として指定するのは、リテラルや代数的データ型のコンストラクタと同じ形のものです。
たとえば、3つ組のタプルを作るときに `(〈式1〉, 〈式2〉, 〈式3〉)` と書くように、
3つ組のタプルにマッチさせたいときは `(〈パターン1〉, 〈パターン2〉, 〈パターン3〉)` のようなパターンの3つ組の形を指定します。

また、パターンの部分には変数も指定できます。そのような**変数パターン**は、任意の値に対してマッチします。変数にはマッチした値が束縛されます。

以降では、「`〈パターン〉 => 〈式〉`」の部分を **節**（clause）と呼び、その中の `〈式〉` の部分を節の **腕**（arm）と呼ぶことにします。

パターンマッチは、case式以外の構文でも使えます。ほかの例については、説明に必要な役者が揃ってから紹介します。

パターンとしても、ここで紹介したリテラルや代数的データ型のコンストラクタ、あるいは変数以外に、もう少し種類があります。それらも追い追い紹介していきます。

## パターンマッチの具体例

パターンマッチの具体例を見てみましょう。[-@sec:ads]項の代数的データ型をパターンマッチを用いて分配束縛していきます。

まずは `week` 型です。これは複数のコンストラクタを持つので複数の節を連ねます。
複数書いた節は最初にパターンにマッチしたものが選ばれます。

```{#lst:isweekend1 .sml caption="week型に対するパターンマッチ例1"}
fun isWeekend w = case w of
                      Sunday => true
                    | Monday => false
                    | Tuesday => false
                    | Wednesday => false
                    | Thursday => false
                    | Friday => false
                    | Saturday => true
```

この `isWeekend` に `Sunday` を与えると、一番最初の節にマッチするので `true` が返ります。 `Friday` を与えると6番めの節にマッチするので、 `false` が返ります。

``` console
# isWeekend Sunday; ⏎
val it = true : bool
# isWeekend Friday; ⏎
val it = false : bool
```

この式は多分岐しているので、 `if` 式の拡張と考えることもできるでしょう。

ところで、 `week` のようにコンストラクタの個数が多い型に対するパターンマッチを書くとき、すべてのコンストラクタを列挙すると、[-@lst:isweekend1]のように冗長になりがちです。
そういうときは、変数パターンは任意の値にマッチすること、複数の節を書いたときは最初にパターンにマッチしたものが選ばれることを利用して、以下のように書けます。

```{#lst:isweekend2 .sml caption="week型に対するパターンマッチ例2"}
fun isWeekend' w = case w of
                      Sunday => true
                    | Saturday => true
                    | other => false
```

[-@lst:isweekend2]では、週末以外のコンストラクタをひとまとめに `other` という変数で受けています。
もし節の順番を入れ替えて以下のように書くと、すべてのコンストラクタが `other` にマッチしてしまい、3行め以降の節には決してマッチしないので注意してください。

```{#lst:isweekend3 .sml caption="変数パターンにすべてマッチしてしまう"}
fun isWeekend' w = case w of
                      other => false
                    | Saturday => true
                    | Sunday => true
```

次は `person` 型へのパターンマッチの例です。
`person` 型は `MkPerson 〈タプル〉` として構築するのでした。この型の値に対するパターンも同様の見た目で `MkPerson 〈タプルのパターン〉` と書きます。

`person` 型にはコンストラクタが1つしかないので、パターンも1つだけ書きます。`person` 型の変数 `p1` に保持されている `string` 型の値をパターンマッチで取り出す式を書いてみましょう。

```{#lst:personexample .sml caption="person型へのパターンマッチ例"}
val name = case p1 of MkPerson (name, age) => name
```

この例では、`person` の保持している`string`と`int`の値を、変数パターンを用いてそのまま変数へと束縛しています。
`p1`が[-@lst:datatype-person2]のように定義されているとすると、これで `name` 変数は `"κeen"` に、 `age` 変数は `26` に束縛されます。

ちなみに、SMLのデータ型は不変なので更新がありません。生成とデータの読み出し、つまり構築と分解だけがSMLのデータ型に対して可能な操作です。

ところで、case式は変数を導入するのでスコープを作ります。
つまり、外にある変数と同じ名前の変数を導入すると、外にあるものをシャドーイングします。
たとえば、以下のように `case` の中で外にあるのと同じ `x` という変数を導入した場合、腕の式では `case` で導入されたほうの `x` が使われます。

``` sml
fun first x = case x of
                  (x, y) => x
```

同じ節の中であれば導入に順番はありません。
たとえば、以下のようにパターン内で同じ名前の変数を導入した場合はコンパイルエラーになります。

``` sml
fun first' x = case x of
                  (x, x) => x
```

``` console
algebraic_data_types.sml:77.22-77.22 Error:
  (name evaluation "201v") duplicate variable name: x
```

[-@lst:personexample]の例では、`person`型のパターンマッチの中で変数パターンへのマッチを行いましたが、このようにパターンはネストが可能です。
[-@lst:datatype-cell1]の`cell`型の例で、「3つめに石が置かれている」かつ「3つとも同じ色である」ことを検査する関数は、以下のように定義できます。

```{#lst:issequence .sml caption="cell型へのパターンマッチ例"}
fun isSequence s = case s of
                       (Full Black, Full Black, Full Black) => true
                    |  (Full White, Full White, Full White) => true
                    | other => false
```

この例では、タプルパターン `(〈pattern1〉, 〈pattern2〉, 〈pattern3〉)` の中に `cell` パターン `Empty | Full 〈pattern〉` が入り、さらにその中に `stone` パターン `Black | White` が入っています。
このようにパターンを組み合わせることで、一見すると複雑に思える条件も簡潔に書けます。

値だけでなくパターンも閉じているので、ここで`stone`型の定義に存在しない `Mentos` に対するパターンを混入させるとコンパイルエラーになります。

``` sml
fun badIsSequence s = case s of
                       (Full Black, Full Black, Full Black) => true
                    |  (Full White, Full White, Full White) => true
                    (* Menthos パターンが混入している *)
                    |  (Full Mentos, Full Mentos, Full Mentos) => true
                    | other => false
```

``` console
(interactive):35.24-35.27 Error:
  (type inference 050) operator and operand don't agree
  operator domain: stone
          operand: third
(interactive):35.37-35.40 Error:
  (type inference 050) operator and operand don't agree
  operator domain: stone
          operand: third
(interactive):35.50-35.53 Error:
  (type inference 050) operator and operand don't agree
  operator domain: stone
          operand: third
```

> note
> SMLでは、 `(*` から `*)` で囲まれた部分はコメント扱いされます。


最後に、再帰したデータ型を扱ってみましょう。 `entry` 型の値を受け取って、そこに含まれているファイル名を表示するプログラムです。

```{#lst:printPathname .sml caption="printPathname関数"}
(* printPathname: string -> entry -> unit *)
fun printPathname prefix e =
    case e of
        File name => print (prefix ^ name ^ "\n")
      | Directory (name, entries) => let val prefix = prefix ^ name ^ "/"
                                     (* app: (entry -> unit) -> entry list -> unit *)
                                     in app (printPathname prefix) entries end
```

受け取った`entry`型の値が`File name`というパターンにマッチするときは、マッチした値が`name`変数に入っているので、それに文字列結合のための`^`演算子を使ってプレフィクスと改行を付け、`print`関数で表示してます（4行め）。
`Directory (name, entries)`というパターンにマッチするときは、`name`変数の値に「`"/"`」を付けたものを新しいプレフィクスとし、`entries`の各エントリに対してライブラリ関数の`app`で再帰的に`printPathname`を繰り返し適用しています。

``` console
# printPathname "" fs; ⏎
./src/main.rs
./src/lib.rs
./README.md
val it = () : unit
```

ネストしたパターンを使うことで、ある程度までの深さの構造にマッチさせることもできますが、
無限の長さのパターンは書けないので、再帰的データ型には再帰関数を使います。
言い換えると、データ型からプログラムの構造が自然に決まるということです。
代数的データ型の設計は、プログラムの設計にも係わる大切な要素なのです。

## パターンの網羅性と非冗長性

ここで[-@lst:isweekend1]の `isWeekend` に再登場してもらいます。

``` sml
fun isWeekend w = case w of
                      Sunday => true
                    | Tuesday => false
                    | Wednesday => false
                    | Thursday => false
                    | Friday => false
                    | Saturday => true
```

もし月曜日のことをすっかり忘れて以下のように書いたらどうなるでしょう。

``` sml
fun isWeekend w = case w of
                      Sunday => true
                    | Tuesday => false
                    | Wednesday => false
                    | Thursday => false
                    | Friday => false
                    | Saturday => true
```

これは警告が出ます。

``` console
(interactive):119.18-125.37 Warning: match nonexhaustive
(interactive):119.18-125.37 Warning: match nonexhaustive
      Sunday  => ...
      Tuesday  => ...
      Wednesday  => ...
      Thursday  => ...
      Friday  => ...
      Saturday  => ...
```

コンパイラはパターンの **網羅性** （exhaustiveness）を検査しており、網羅的でないパターンには警告を出してくれるのです。
これは、代数的データ型が定義時に可能な値をすべて列挙しているから可能な機能です。

もし警告を無視して、この間違った `isWeekend` に `Monday` を渡すと、次のように例外 `Match` が出ます。

``` console
# isWeekend Monday; ⏎
uncaught exception Match at (interactive):119
```

網羅性検査は、もっと複雑なパターンでも必ず実施されます。
たとえば、リストのタプルに対するマッチを考えてみましょう。
リストに対するパターンマッチでは、 `::` と `nil` を使った書き方と、 `[〈pattern〉, …]` という形を使った書き方が可能です。要するにリストの構築と同じ見た目でパターンを書けるということですね。

いま、パターンマッチを使って、「`[1, 2, 3]` と `[4, 5, 6]` を受け取ったら `[(1, 4), (2, 5), (3, 6)]` を返す」という動作の関数 `zip` を次のように定義したとします。

```{#lst:zip1 .sml caption="zip関数の最初の定義"}
fun zip xs ys = case (xs, ys) of
                    ([], []) => []
                  | (x::xs, y::ys) => (x, y) :: zip xs ys
```

[-@lst:zip1]の定義は、一見すると良さそうですが、このコードをコンパイルすると次のような警告が出ます。
[-@lst:zip1]のパターンマッチでは `([], y::ys)` や `(x::xs, [])` の場合が考慮できていないからです。

``` console
algebraic_data_types.sml:103.16-105.56 Warning: match nonexhaustive
      (nil , nil ) => ...
      (:: (x, xs), :: (y, ys)) => ...
```

> note リストと代数的データ型
> リストに対して `::` と `nil` でパターンマッチできることからお気づきかもしれませんが、リストも代数的データ型の1つです。
> SMLの規格では `::` や `nil` が再定義不能になっているので、実際にこのように書いてコンパイルすることはできませんが、概ね以下のような定義になります。
> リストに対してパターンマッチするときに意識してみると、コードの見通しが良くなるかもしれません。
> 
> ``` sml
> datatype 'a list = :: of 'a * 'a list | nil
> 
> infixr 5 ::
> ```
> 
> 1行めの`'a`は、多相型（他言語でいうジェネリクス）のパラメータです。
> 3行めでは、`::`を中置演算子としてその結合順位を設定しています。

先ほどは足りないパターンがある例を見ましたが、余計なパターンがある場合も検査されます。再び `isWeekend` で `Sunday` を二度書いてしまったとしましょう。

```{#lst:isweekend4 .sml caption="パターンが多い例"}
fun isWeekend w = case w of
                      Sunday => true
                    | Monday => false
                    | Tuesday => false
                    | Wednesday => false
                    | Thursday => false
                    | Friday => false
                    | Saturday => true
                    | Sunday => true
```

この関数をコンパイルすると、SML♯は今度は警告ではなくエラーを出します[^matchredundant]。

[^matchredundant]: エラーを出すのはSML♯での挙動です。SMLの仕様ではエラーでも警告でもよいとされています。網羅性も同様の扱いです。

``` console
(interactive):132.18-140.35 Error: match redundant
      Sunday  => ...
      Monday  => ...
      Tuesday  => ...
      Wednesday  => ...
      Thursday  => ...
      Friday  => ...
      Saturday  => ...
  --> Sunday  => ...
```

よく見ると一番最後の `Sunday` の左側に `-->` という印が付いていますね。
パターンマッチは複数マッチするものがあれば上に書かれたものが優先されるのでした。
可能なあらゆる値が上にあるパターンにマッチしてしまい、該当のパターンに到達することがない場合は、冗長なパターンと扱われます。
たとえば、[-@lst:isweekend3]に示した`isWeekend`は、実際にはコンパイラがエラーを出すのでコンパイルできません。
コンパイラは、このようにパターンの **非冗長性** （irredundancy）も検査し、プログラマーのミスを早期に発見してくれるのです。

## ワイルドカードパターン

[-@lst:issequence]の`isSequence`の定義を思い出してみましょう。

```{#lst:issequence2 .sml caption="\ref{lst:issequence}の再掲"}
fun isSequence s = case s of
                       (Full Black, Full Black, Full Black) => true
                    |  (Full White, Full White, Full White) => true
                    | other => false
```

この定義では、値を `other` に束縛したあと `other` を利用していませんね。`other`はパターンをまとめているだけです。このような場合は、値を使わないことを表す「`_`」を使うと、より意図を正確に表現できます。

```{#lst:issequense3 .sml caption="ワイルドカードパターン"}
fun isSequence s = case s of
                       (Full Black, Full Black, Full Black) => true
                    |  (Full White, Full White, Full White) => true
                    | _ => false
```

このような「`_`」をワイルドカードパターンといいます。
ワイルドカードパターンが特に有用なのは、以下のように1つのパターン内で無視したい値が複数個ある場合です。

```{#lst:issequence4 .sml caption="複数回ワイルドパターンを使う"}
fun isFull s = case s of
                   (Full _, Full _, Full _) => true
                | _ => false
```

変数パターンと違って、ワイルドカードパターンは束縛を導入しないので、何度用いても衝突しません。
余計な束縛を導入しないので、思いがけないシャドーイングも防げます。

## パターンを書ける場所

ここまではcase式のみでパターンマッチを使っていましたが、ほかにもパターンが書ける箇所はいくつか存在します。

1つめは、 `fun` で定義する関数の引数です。
これまでは `fun 〈名前〉 〈引数〉 ... = 〈定義〉` のように書いて関数を定義していましたが、実は引数部分でパターンマッチができます。
いままで引数名を書いてると思っていたのは、実は変数パターンを書いていたのでした。
なので、関数定義は、より正しくは次のように書けます。

```sml
fun 〈名前〉 〈パターン〉 ... = 〈定義〉 
  | 〈名前〉 〈パターン〉 ... = 〈定義〉 
  | ...
```

この構文を用いて `isWeekend'` を書き直すと[-@lst:isweekend-pattern]のようになります。

```{#lst:isweekend-pattern .sml caption="関数定義の引数部分でパターンマッチを使う"}
fun isWeekend' Sunday = true
  | isWeekend' Saturday = true
  | isWeekend' _ = false
```

引数 `w` を受け取ってから `case` で分岐していた定義が、より宣言的に書けるようになりました。少し数学的な定義に似た構文といえるかもしれません。

パターンマッチは引数のどこででもできるので、 `printPathname` のように第2引数でパターンマッチする関数もこの構文で書けます。

``` sml
fun printPathname prefix (File name) = print (prefix ^ name ^ "\n")
  | printPathname prefix Directory (name, entries) =
    let val prefix = prefix ^ name ^ "/"
    in app (printPathname prefix) entries end
```

あるいは `zip` のように複数個の引数でパターンマッチする関数も書けます。

``` sml
fun zip _ [] = []
  | zip [] _ = []
  | zip (x::xs) (y::ys) = (x, y) :: zip xs ys
```

また、引数でのパターンマッチがあるので、タプルを使って次のような見た目の関数定義も書けます。
他の言語において複数の引数を取る関数を定義するときのような見た目ですね。

``` sml
fun add(x, y) = x + y
```

`val` による変数束縛の導入でもパターンマッチが使えます。実は本記事の冒頭で説明した`val`は、変数パターンへのマッチだったのです。

``` sml
val (x, y, z) = (1, 2, 3)
```

`val` でマッチする場合は、マッチしない可能性があると警告があがります。

```{#lst:val-warn .sml caption="SundayパターンにSundayをマッチ"}
val Sunday = Sunday
```

``` console
(interactive):149.4-149.18 Warning: binding not exhaustive
      Sunday  => ...
```

実際、マッチしない場合は例外 `Bind` が起きます。

```{#lst:val-bind .sml caption="SundayパターンにMondayをマッチ"}
val Sunday = Monday
```

``` console
(interactive):152.4-152.18 Warning: binding not exhaustive
      Sunday  => ...
uncaught exception Bind at (interactive):152
```

SMLで使えるパターンマッチについては、以上でいったん紹介を終わります。
本記事で紹介していない機能に対応するパターンや、紹介していない構文で使えるパターンマッチもありますが、長くなるのでこのくらいにしておきましょう。

★ メモ: 例外もパターンマッチをしているが、扱いが難しいので意図的にスルー

