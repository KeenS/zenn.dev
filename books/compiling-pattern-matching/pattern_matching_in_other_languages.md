---
title: 代数的データ型がない言語とパターンマッチ
---

ここまでの説明では、代数的データ型の存在を前提としてパターンマッチの仕組みを見てきました。しかし既存の言語には代数的データ型そのものがないものも多くあります。そして、そうした既存の言語にもパターンマッチを入れようとする動きがあります。代数的データ型がない言語ではパターンマッチをどのように考えればいいのでしょうか。

ここまでの説明では、パターンマッチと代数的データ型を一緒に使ってきました。パターンマッチと代数的データ型を一緒に使うことによる利点は、以下のように要約されると思います。

1. 構築（コンストラクタ）と分解（パターン）が同じ見た目で書ける
2. コンパイル時に網羅性が検査される
3. コンパイル時に非冗長性が検査される
4. 直観的なパターンを書いて、人間が手で書くのが煩らわしいような最適なマッチを実行するためのコードが生成されるようにできる

実は、これらの恩恵の一部を諦めると、代数的データ型がない言語でもパターンマッチを便利に使えます。この節では、そうした言語におけるパターンマッチについて概略を紹介します。

## Common Lisp

マクロで構文を拡張できるLispでは、言語仕様にない機能でもユーザの手で実装できます。
実際、Common Lisp[^sbclspec]には、パターンマッチをサポートするライブラリがあります！

[^sbclspec]: 本項の内容はSBCL 1.5.6に基づくものです。

少し古いライブラリですが、そのようなライブラリのひとつとして、optima [^optima]というものを紹介します。Common Lispのライブラリは、quicklisp[^quicklisp]という仕組みを使って、次のように簡単にインストールできます。

[^optima]: [https://github.com/m2ym/optima](https://github.com/m2ym/optima) （現在は開発者によってアーカイブ済み）
[^quicklisp]: [https://www.quicklisp.org/beta/](https://www.quicklisp.org/beta/)

``` lisp
(ql:quickload :optima)
```

optimaの `match` マクロをインポートすると、すぐにパターンマッチが使えるようになります。

``` lisp
(shadowing-import 'optima:match)
```

基本的な挙動を確認してみましょう。 `x` に `1` が渡されたら `"one"` を、 `2` が渡されたら `"two"` を、 `3` が渡されたら `"three"` を、 それ以外が渡されたら `"other"` を返すシンプルな関数をパターンマッチで書いてみます。

``` lisp
(defun trivial-match (v)
  (match v
    (1  "one")
    (2  "two")
    (3  "three")
    (_  "other")))
```

これを試すと、意図どおりの結果を返します。

``` console
CL-USER> (trivial-match 1) ⏎
"one"
CL-USER> (trivial-match 2) ⏎
"two"
CL-USER> (trivial-match 3) ⏎
"three"
CL-USER> (trivial-match 4) ⏎
"other"
```

もう少し複雑な例も紹介します。次の例は、3つまでの要素を持つリストに対してマッチし、その要素の和を返すというマッチです。

``` lisp
(match v
  ((list x)     x)
  ((list x y)   (+ x y))
  ((list x y z) (+ x y z)))
```

Common Lispは動的型付言語なので、リストとベクタのそれぞれに対するパターンを1つのマッチに書くこともできます。SMLではありえない例ですが、Common Lispではこういうパターンマッチが必要になる場合もあるでしょう。

``` lisp
(match v
  ((list x)   "list")
  ((vector x) "vector")
  (x          "other"))
```

このように、optimaを使うとCommon Lispでパターンマッチを書けるようになります。しかし、optimaによるパターンマッチには、SMLにあったような網羅性検査や非冗長性検査は付いてきません。たとえば、以下のコードには冗長なパターンがあります。もちろん、すべてのリストが網羅されているわけではないですし、先ほど紹介した「まったく別の型の値（数値型やベクタ型など）に対するマッチ」も含まれていません。

``` lisp
(match v
  ((list x)     x)
  ((list x)     x)
  ((list x y)   (+ x y))
  ((list x y z) (+ x y z)))
```

それでも、上記のコードは警告もなしにコンパイルが通ります。そして、たとえば引数に数値である `3` を与えて実行すると、 `nil` が返ります。

```console
CL-USER> (match 3
  ((list x)     x)
  ((list x)     x)
  ((list x y)   (+ x y))
  ((list x y z) (+ x y z))) ⏎
NIL
```

マッチが失敗したときの挙動については、 `match` の代わりに`ematch` というマクロを使うことで、エラーにすることはできます。それでも、コンパイル時に不要なパターンや不当なパターンを自動で検出してくれるといった仕組みはありません。

生成されるコードについてはどうでしょうか。optimaの `match` はマクロなので、Common Lisp処理系を用意すれば、展開（コンパイル）後のコードを確認できます。

例として、リスト4.12の `isSequence` 関数の定義に使ったパターンマッチをoptimaで書き直してみて、その展開結果を調べてみましょう。

``` lisp
(match v
  ((list 'black 'black 'black) t)
  ((list 'white 'white 'white) t)
  (otherwise                 nil))
```

見てのとおり、SMLの場合と遜色なくパターンマッチを書けることがわかると思います。

上記のコードをCommon Lisp処理系に展開させたあと、大文字を小文字に直したり、パッケージ名の修飾や自動生成された名前などを取り除いたりしてコードの見た目を整えると、下記のようになります。

``` lisp
(let ((tmp1 v))
  (%or
   (%if (consp tmp1)
        (let ((tmp2 (car tmp1))
              (tmp3 (cdr tmp1)))
          (%or
           (%if (%equals tmp2 black)
                (%if (consp tmp3)
                     (let ((tmp4 (car tmp3))
                           (tmp5 (cdr tmp3)))
                       (%if (%equals tmp4 black)
                            (%if (consp tmp5)
                                 (let ((tmp6 (car tmp5))
                                       (tmp7 (cdr tmp5)))
                                   (%if (%equals tmp6 black)
                                        (%if (%equals tmp7 nil)
                                             (%or t (optima:fail))
                                             (optima:fail))
                                        (optima:fail)))
                                 (optima:fail))
                            (optima:fail)))
                     (optima:fail))
                (optima:fail))
           (%if (%equals tmp2 white)
                (%if (consp tmp3)
                     (let ((tmp8 (car tmp3))
                           (tmp9 (cdr tmp3)))
                       (%if (%equals tmp8 white)
                            (%if (consp tmp9)
                                 (let ((tmp10 (car tmp9))
                                       (tmp11 (cdr tmp9)))
                                   (%if (%equals tmp10 white)
                                        (%if (%equals tmp11 nil)
                                             (%or t (optima:fail))
                                             (optima:fail))
                                        (optima:fail)))
                                 (optima:fail))
                            (optima:fail)))
                     (optima:fail))
                (optima:fail))))
        (optima:fail))
   (%or
    (%or (let ((otherwize tmp1))
           nil)
         (optima:fail))
        nil)))
```

SMLにおけるパターンマッチをC言語へと展開した結果（リスト6.16）によく似た見た目ですね。optimaという名前が示唆するように、最適化された（optimized）コードが手に入ります。

なお、よく見るとリスト6.16と違う部分もあります。これは、リスト6.16の生成に使われているのとは異なるアルゴリズムが使われているからです。具体的には、リスト6.16だと `SML_FALSE` だった部分が `(optima:fail)` になっています。「[パターンマッチの意味](pattern_matching_semantics)」で説明に使った$\mathit{FAIL}$と同じような働きをマクロとして実装しているわけです。この `(optima:fail)` （および一緒に使われている`&or`）については、記事を改めて説明する予定です。


## Java

Javaでもパターンマッチの導入が議論されているようです[^javaspec]。これがなかなか面白いというか、うまく既存の言語機能とバランスを取って設計されているので、簡単に紹介します。

[^javaspec]: 本項の内容はJava openjdk 1.8.0_222およびScala 2.11.12に基づくものです。

Javaにも代数的データ型はありませんが、抽象クラス（またはインタフェース）とサブクラス（または実装クラス）を使えば、複数種類のデータを統一的に扱うことはできます。しかし、抽象クラスとサブクラスには、パターンマッチとの関連で重要な代数的データ型の次のような性質がありません。

* 構築（コンストラクタ）と分解（パターン）が同じ見た目で書ける
* サブクラスが互いに排他である（サブクラスAでありかつサブクラスBであるようなものがない）
* サブクラスの種類がコンパイル時にわかる

このことをコードで確認してみましょう。
リスト4.5で代数的データ型として定義した `stone` と同じものは、抽象クラスとそのサブクラスを使うと、次のようなJavaのコードとして実装できます。

``` java
abstract class Stone {}
class Black extends Stone {}
class White extends Stone {}

abstract class Cell {}
class Full extends Cell {
    Stone value;
    Full(Stone value) {
        this.value = value;
    }
}
class Empty extends Cell {}
```

これは代数的データ型が持つような性質は満たしません。たとえば `Stone` は誰でも継承できるので、以下のように第三の勢力を投入することができます。

```java:リスト8.1
class Menthos extends Stone {}
```
*第三の値`Menthos`を追加*


あるいは `Full` を継承する新たなクラス `Corner` を定義すると `Full` でありかつ `Corner` であるようなクラスを作れてしまいます。

```java:リスト8.2
class Corner extends Full {
    Corner(Stone value) {
        super(value);
    }
}
```
*新たなクラス`Corner`を`Full`でもあるように追加*

さらに、抽象クラスのコンストラクタは自由にオーバーライドできるので、見た目と作られたデータが異なるようにすることもできます。

```java
class Full extends Cell {
    Stone value;
    Full(Stone value) {
        if (value instanceof White) {
            this.value = new Black();
        } else {
            this.value = new White();
        }
    }
}
```
*コンストラクタは自由にオーバーライド可能*


Javaでは拡張可能性が重要視されますが、それがそのまま代数的データ型とパターンマッチの導入に対する壁になっているのです。

このまま `isSequence` も実装してみましょう。説明の都合上、Visitor Patternなどは使わず、 `instanceof` で実装していきます。

``` java
boolean isSequence(Cell c1, Cell c2, Cell c3) {
    if (c1 instanceof Full) {
        Full f1 = (Full)c1;
        if (c2 instanceof Full) {
            Full f2 = (Full)c2;
            if (c3 instanceof Full) {
                Full f3 = (Full)c3;
                return ((f1.value instanceof Black) && 
                        (f2.value instanceof Black) && 
                        (f3.value instanceof Black)) ||
                    ((f1.value instanceof White) && 
                     (f2.value instanceof White) && 
                     (f3.value instanceof White));
            }
        }
    }
    return false;
}
```

この中には、以下のような「型検査をしてからダウンキャスト」するというパターンが何度も現れます。

```java:リスト8.3
if (obj instanceof Full) {
  Full x = (Full)obj;
  // ...
}
```
*型検査をしてからダウンキャスト*


この記述はちょっと冗長ですね。特に、1つのイディオムの中で `Full` を3回もタイプしています。さらにこれら3つの `Full` はすべて同じクラス名です。冗長なだけでなく、3つの `Full` を「プログラマーが間違えないように気をつけて書く」方針ではバグの温床にもなりかねません。

このような `instanceof` による型検査とキャストは頻出のパターンなので、もう少し短く書けるようにしたいところです。そこで提案されているのがパターンマッチです。Javaにもパターンマッチが導入されるかもしれないのです！

まず、JEP 305[^jep305]として、リスト8.3のようなボイラープレートを以下のような見た目で書けるようにすることが提案されています。

[^jep305]: [http://openjdk.java.net/jeps/305](http://openjdk.java.net/jeps/305)

```java:リスト8.4
if (obj instanceof Full x) {
    // ...
}
```
*JEP 305提案*


さらに、JEP 354[^jep354]では、`switch` 文への拡張 （ `switch` 式）が提案されています。この `switch` の拡張と組み合わせて、以下のようなパターンも書けるようにしたいとされています[^Bierman]。

[^jep354]: [http://openjdk.java.net/jeps/354](http://openjdk.java.net/jeps/354)、Java 13でプレビュー機能としてリリース済み。
[^Bierman]: [http://cr.openjdk.java.net/~briangoetz/amber/pattern-match.html](http://cr.openjdk.java.net/~briangoetz/amber/pattern-match.html)

``` java
switch (obj) {
    case Full x  -> // ...;
    case Empty x -> // ...;
}
```

型による分岐だけですが、これで多少はSMLなどのパターンマッチに見た目が近付きましたね。

しかし、この構文が導入されたところで、代数的データ型の性質から得られるようなパターンマッチの性質は満たしません。たとえば網羅性については、 `White` または `Black` で以下のように分岐したとして、第三の勢力にマッチする可能性を排除できません（実際、 `Stone` クラスを継承して `Menthos` クラスを作れたことを思い出しましょう）。

``` java
boolean isBlack(Stone s) {
    return switch(s) {
        case White w -> false;
        case Black b -> true;
        default -> ????;
    };
}
```

サブクラスに由来する面倒もあります。代数的データ型と異なり、「AでありかつBであるような値は存在しない」という性質が成り立ちません。そのため、たとえば「`Cell` が `Full` または `Empty` であることに加えて、`Full`を継承した`Corner`になる可能性もある」という分岐をパターンマッチで書くとき、プログラマーがよく気をつけないと、無用なパターンを書いてしまう可能性があります。実際、以下のコードには無用なパターンがありますが、わかるでしょうか？

``` java
switch (c) {
    case Full f -> // ...;
    case Corner c -> // ...;
    case Empty e -> // ...;
    default -> // ...;
}
```

サブクラスも `instanceof` で `true` を返してしまうので、上記の `switch` 式では、 `Full` のサブクラス `Corner` にマッチする場合は2行めの `Full f` というパターンに吸収されてしまいます。
したがって、3行めの `Corner c` にマッチすることはありません。無用なパターンを避けるには「プログラマーが気をつけて書く」しかないのです。

さて、ここまでは代数的データ型に近いものをJavaで書くのに抽象クラスとサブクラスを使ってきましたが、Java には列挙型 `enum` もあります。`enum` を使うと、 `Black` もしくは `White` であるような `Stone` を以下のように書けます。

``` java
enum Stone { BLACK, WHITE }
```

この `enum` を使ったコードは、コンパイルすると、概ね以下のようなクラスへと展開されます。

``` java
final class Stone extends Enum<Stone> {
    private Stone(String name, int ordinal) {
        super(name, ordinal);
    }

    public static final Stone BLACK = new Stone("BLACK", 0);
    public static final Stone WHITE = new Stone("WHITE", 1);

    private static final Stone ENUM$VALUES[] = {BLACK, WHITE};
}
```

展開後のクラスで注目してほしいのは以下の点です。

* `final` を付けることで勝手な継承ができなくなっている
* 列挙子 `BLACK` と `WHITE` は定数になっている
* コンストラクタをプライベートにすることで勝手にインスタンスを作れなくなっている

つまり、`enum` を使って `Stone` を定義すれば、勝手に第三の勢力を増やしたり、列挙子のサブクラスを増やしたりすることはできなくなります。

では、なぜわざわざ抽象クラスとサブクラスを使ってデータを表現したのでしょうか？実は、`Stone` は上記のように `enum` でうまく定義できるのですが、同じリスト4.5で定義していた `cell` に相当するデータ型（リスト8.5）は、残念ながら `enum` ではうまくいきません。

```sml:リスト8.5
datatype cell = Empty | Full of stone
```
*リスト4.5で定義しているcell*

これをJavaで `enum` を使って定義しようとしても、`Full` に値を保持する必要があるので、定数としてエンコードできないのです。

しかし、クラスを `final` にする手は使えそうです。いま、 `Stone` と同じ方法で継承クラスとして `Cell` を定義し、継承したクラスたちに `final` を付けておくことにしましょう。

``` java
abstract class Cell {}
final class Full extends Cell {
    Stone value;
}
final class Empty extends Cell {}
```

これで勝手に新しいクラス（たとえば `Corner` クラス）が作られる可能性はなくなりました。しかし、まだ新たな `Cell` のサブクラスが作られる可能性はあります。

``` java
final class Pod extends Cell {
    Stone stones[];
}
```

これは残念ながら現在のJavaの機能では防ぐことができないようです。

そこで考えられているのが `sealed` な型という機能です[^sealed]。`sealed` な型は、以下のようにして継承できる型を制限できるようです。

[^sealed]: [http://cr.openjdk.java.net/~briangoetz/amber/datum.html](http://cr.openjdk.java.net/~briangoetz/amber/datum.html)

``` java
// 継承できるクラスを `Full` と `Empty` に制限
sealed abstract class Cell permits Full, Empty {}
final class Full {
    Stone value;
}
final class Empty extends Cell {}

// `Full`, `Empty` 以外で継承しようとするとエラー
// final class Pod extends Cell {
//     Stone stones[];
// }
```

`Cell` を `sealed` な型とすることで、以下のように `switch` 式における `default` 節が不要になります。

``` java
switch (c) {
    case Full f -> // ...;
    case Empty e -> // ...;
    // default節不要
}
```

クラスを `final` にする手法と `sealed` の導入により、Javaのパターンマッチでも、代数的データ型がもたらすパターンマッチの3つの特長（下記に再掲）のうち、後者の2つはクリアされました。

* 構築（コンストラクタ）と分解（パターン）が同じ見た目で書ける
* コンストラクタが互いに排他である（コンストラクタAでありかつコンストラクタBであるようなものがない）
* コンストラクタの種類がコンパイル時にわかる

残るは「構築（コンストラクタ）と分解（パターン）が同じ見た目で書ける」です。これについては、データ型定義のための`record`という新しい構文を用意し、生成も分解も同じ構文でできるような提案が考えられているようです[^record]。

[^record]: `sealed` が提案されている文書および次のJEP：[http://openjdk.java.net/jeps/8222777](http://openjdk.java.net/jeps/8222777)

`Cell` を題材に`record`の例を見てみましょう。`record`は継承ができないようなので `Cell` をインタフェースで実装すると、次のように`Cell`に対するパターンマッチのコードが書けます。

``` java
// 定義
sealed interface Cell {}
record Full(Stone s) implements Cell {}
record Empty() implements Cell {}

// 生成とパターンマッチ
Cell c = new Empty();
switch (c) {
    case Full(Stone s) -> // ...;
    case Empty()       -> // ...;
}
```

ここまでくると、かなり代数的データ型に対するパターンマッチに近い表現力が得られますね。

ただし、ここで紹介した機能はリリース版のJavaではまだ利用できません。Scalaでは本質的に同じ機能がすでに利用できるので、Scalaで試してみましょう。

``` scala
sealed trait Stone;
case class Black() extends Stone;
case class White() extends Stone;

sealed trait Cell;
case class Full(s: Stone) extends Cell;
case class Empty() extends Cell;

class PatternMatch {
  def isSequence(c1: Cell, c2: Cell, c3: Cell): Boolean = (c1, c2, c3) match {
    case (Full(Black()), Full(Black()), Full(Black())) => true
    case (Full(White()), Full(White()), Full(White())) => true
    case _ => false
  }
}
```

この場合、冗長なパターンや非網羅的なパターンを書くと、Scalaのコンパイラが警告します。試しに、13行め（`case _ => false`）を削除してコンパイルしてみましょう。

```console
$ scalac PatternMatch.scala ⏎
atternMatch.scala:10: warning: match may not be exhaustive.
It would fail on the following inputs: (Empty, Empty, Empty), (Empty, Empty, Full(Black)), (Empty, Empty, Full(White)), (Empty, Full(Black), Empty), (Empty, Full(Black), Full(Black)), (Empty, Full(Black), Full(White)), (Empty, Full(White), Empty), (Empty, Full(White), Full(Black)), (Empty, Full(White), Full(White)), (Full(Black), Empty, Full(Black)), (Full(Black), Empty, Full(White)), (Full(Black), Full(White), Full(Black)), (Full(Black), Full(White), Full(White))
  def isSequence(c1: Cell, c2: Cell, c3: Cell): Boolean = (c1, c2, c3) match {
                                                          ^
warning: there was one unchecked warning; re-run with -unchecked for details
```
*case _ => false を削除*


冗長な場合についても試してみましょう。リスト8.6は、`case (Full(White()), Full(White()), Full(White())) => true`を追加してコンパイルした結果です。

```console:リスト8.6
$ scalac PatternMatch.scala ⏎
PatternMatch.scala:13: warning: unreachable code
    case (Full(White), Full(White), Full(White)) => true
                                                    ^
one warning found
```
*case (Full(White()), Full(White()), Full(White())) => trueを追加*


どちらの場合もコンパイラが警告を出してくれることが確認できました。

なお、筆者が上記のコードを `scalac -optimise -print` でコンパイルし、出力を確認したところ `switch` を使った高速化は施されていないようでした。JVMの内部ではクラスIDに相当する数値を持っていると考えられるので、それをキーにした `switch` が書けるといいのですが、APIがないからか、そういった実装にはなっていないようです。

以上、Javaのように言語設計当初からパターンマッチを持っていなかった言語でも、拡張の工夫次第であとからパターンマッチが導入できる例を紹介しました[^scala]。ここで見たように、継承による拡張が可能というJavaの設計は「網羅性や冗長性をコンパイラにより事前に検査できる」というパターンマッチの恩恵とは相性が悪いものです。しかし、それらを制限する機能を使うことで、この恩恵を受けられるようになります。さらに、`record`という、構築子と保持するデータの形が一致しているデータ型の導入によって、代数的データ型に対するパターンマッチに近い有用性が得られることを確認しました。

[^scala]: 最後の動くコードはScalaになってしまいましたが、同じ道具立てなので、本質は変わりません。
