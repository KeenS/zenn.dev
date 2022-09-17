---
title: "型駆動開発"
---

本章ではIdrisで推奨されるプログラムの開発方法、型駆動開発について学びます。

# 型駆動開発とは

型駆動開発（Type Driven Development）とはその名の通り型を基軸にしてプログラムを書き進める開発方法です。まずプログラムの型を書き、その型に合うプログラムをHoleを使いつつ書き始めます。少し書き進めるとコンパイラから型エラーや環境情報などの形でフィードバックが得られるので、そのフィードバックを元にプログラムをさらに書き進めるという形で進みます。以下の3つのプロセスがあります。

* Type: 最初の型書く、あるいはHoleの型を調べて次の進め方を決める
* Define: 型に合う関数を定義する。トップレベルの関数のこともあればサブ関数であることもある
* Refine: プログラムを書き進める、あるいは型を修正してより適切な形にする

典型的にはType→Define→Refineの順で進みますがType→Refine→Type→Refineのような進行もありえます。

型はプログラムの正しさを保証する仕組みの1つなので、型駆動開発はテスト駆動開発にも似ています。しかし型情報は処理系が詳しく知っているので精細なフィードバックやツールのサポートが得られる点が面白いところです。

# Idrisの型駆動開発サポート

Idrisは型駆動開発のために開発された言語ということもあって各種サポートがあります。

* Hole
  + Holeの型を調べられる
  + 未完成のプログラムも書けるようになる
* 依存型による詳細な型
  + 型を詳細に書けるほどプログラムを書くときのガイドとしての力が強くなる
* 名前つきパラメータ
* 型定義から本体定義の自動生成
* 処理系による編集コマンド
  + 型を元にしたrefineを一部自動でやってくれる

さらにはIdrisの作者自身[Type Driven Development with Idris](https://www.manning.com/books/type-driven-development-with-idris)という本を著しています。これは無料でも読めるので是非一読下さい。既に型駆動開発を題材にした書籍があるので、本書では雰囲気を掴める程度、「型駆動開発って面白い」って思える程度に紹介していけてらなと思います。

## エディタサポート

Idrisのインストールの章でも紹介しましたが、各種エディタに編集コマンドのサポートが用意されています。それを再掲します。

| Command       | Emacs     | Vim              | Atom         | Sublime |
|---------------|-----------|------------------|--------------|---------|
| `addclause`   | `C-c C-s` | `<LocalLeader>d` | `ctrl-alt-a` | `⌃ ⌘ V` |
| `casesplit`   | `C-c C-c` | `<LocalLeader>c` | `ctrl-alt-c` | `⌃ ⌘ C` |
| `addmissing`  | `C-c C-m` | `<LocalLeader>m` | ?            | `⌃ ⌘ M` |
| `proofsearch` | `C-c C-a` | `<LocalLeader>p` | `ctrl-alt-s` | `⌃ ⌘ S` |

ここで、各コマンドは以下のような意味です。

* `addclause`：関数の型定義から実装のテンプレートを作成する
* `casesplit`：変数の値で場合分けする
* `addmissing`：不足している場合分けを補う
* `proofsearch`：Holeに該当する証明を探索して自動で埋める


これらのコマンドを使って型駆動開発を進めていきます。


# 型駆動開発体験：VectのAppend

まずは小さな型駆動開発のサイクルを回して型駆動開発を体験してみましょう。私が好きな題材は `Vect` の `append` です。 `Vect` の `append` は標準ライブラリで定義されているので名前が衝突しないように気をつけます。

`Data.Vect` から `Vect` データ型のみインポートします。


```idris
import Data.Vect using (Vect)
```

それでは `Vect` の `append` を型駆動で実装していきます。

Type: `append` の型を定義する。長さ `n` の `Vect` と長さ `m` の `Vect` を結合すると長さ `n + m` になるので型は `Vect n a -> Vect m a -> Vect (n + m) a` となります。ここで、引数は名前つき引数にしてそれぞれ `xs` と `ys` とします。

```idris
append : (xs : Vect n a) -> (ys : Vect m a) -> Vect (n + m) a
```


Define: `append` の本体を書き始めます。ここで編集コマンドの `addclause` が使えます。 `append` 識別子にカーソルを合わせて `addclause` コマンドを叩きます。すると関数本体がHole付きで生成されます。

```idris
append : (xs : Vect n a) -> (ys : Vect m a) -> Vect (n + m) a
append xs ys = ?append_rhs
```

`xs` と `ys` は名前付き引数から採られています。

Type: ここでHoleが登場したので一旦Holeの型を確かめましょう。プログラムをロードするとIdrisがHoleの型を教えてくれます。

```
- + Main.append_rhs [P]
 `--                a : Type
                    m : Nat
                   ys : Vect m a
                    n : Nat
                   xs : Vect n a
     -------------------------------------
      Main.append_rhs : Vect (plus n m) a
```


`---` の上が環境、つまり使える道具で `---` の下がHoleの型です。 `a` 、 `m` 、 `n` は型の情報なので残る `xs` と `ys` を使って `Vect (plus n m) a` を返すということが読み取れます。

さて、Refineしましょう。

Refine: `xs` でパターンマッチをする。エディタサポートがあるので引数の `xs` に合わせて `casesplit` する。

```idris
append : (xs : Vect n a) -> (ys : Vect m a) -> Vect (n + m) a
append [] ys = ?append_rhs_1
append (x :: xs) ys = ?append_rhs_2
```

`xs` でパターンマッチをするのはやや天下り的ですが、再帰的データ型にはパターンマッチと再帰関数を使う、引数の先頭から試してみる、くらいの経験則です。 `xs` で試して上手くいかなかったら `ys` でパターンマッチします。

Type: 新しいHoleが2つ出現したのでそれぞれ型を見ていきましょう。まずは `[]` の節の `append_rhs_1` 。

```
- + Main.append_rhs_1 [P]
 `--                  a : Type
                      m : Nat
                     ys : Vect m a
     ------------------------------
      Main.append_rhs_1 : Vect m a
```

`a` 、 `m` は型情報なので `ys` を使って `Vect m a` を作ります。…これはそのま `ys` を返すだけですね。このくらいの簡単なプログラムならエディタサポートで自動で穴埋めしてくれます。

Refine: `append_rhs_1` にカーソルを合わせて `proofsearch` をします。

```idris
append : (xs : Vect n a) -> (ys : Vect m a) -> Vect (n + m) a
append [] ys = ys
append (x :: xs) ys = ?append_rhs_2
```

すると目論見通り `ys` をIdrisが埋めてくれます。

Type: `append_rhs_2` の型をみてみます。


```
- + Main.append_rhs_2 [P]
 `--                  a : Type
                      x : a
                      m : Nat
                     ys : Vect m a
                    len : Nat
                     xs : Vect len a
     ---------------------------------------------
      Main.append_rhs_2 : Vect (S (plus len m)) a
```

`a` 、 `m` 、 `len` は型なので `x` 、 `ys` 、 `xs` を使って `Vect (S (plus len m)) a` を作ります。これは再帰呼び出しに気付けば簡単にプログラムが書けます。

Refine: `append_rhs_2` にカーソルを合わせて `proofsearch` をします。

```idris
append : (xs : Vect n a) -> (ys : Vect m a) -> Vect (n + m) a
append [] ys = ys
append (x :: xs) ys = x :: append xs ys
```

Idrisが `(x :: xs)` の節の本体も生成してくれました。

ということで `append` の型を書くだけであとは1文字もプログラムを書くことなく `append` の本体を生成できました。参考に動画も用意したので手元に開発環境がない方も雰囲気だけでも感じ取って下さい。

https://www.youtube.com/watch?v=uauu1mCwbeo


もちろん、今回の例はかなり極端な部類です。普通は型のみからプログラムを導出するのはほぼできません。型の情報が足りないからですね。例えば `append` と同じ型で最初の引数を逆順に結合する `revAppend` だってあるので型だけからどんなプログラムを書こうとしているのか判断するのが難しそうなことは納得できるかと思います。 `append` はたまたま正しい実装になる例を採り上げただけです。

それでも実装の補助として型を使うのは依然有効な手段です。特に型が強力で詳細な仕様を型で表現できるIdrisなら強力なサポートが受けられます。「時には勢い余って全部実装してしまうくらい強力なサポート」くらいの気持で向き合っていきましょう。

# 型駆動開発：行列の転置

VectのAppendで雰囲気を掴めたと思うのでもう少し難しい題材をやってみましょう。TDD本の3.3の練習問題1にある、行列の転置操作を型駆動開発で実装してみます。今度はちゃんと自分の手で実装する部分もあるので開発っぽさを味わえると思います。

行列の転置は行と列を転置する操作です。例えば以下のデータがあるとして

```idris
[[1, 2, 3],
 [4, 5, 6]]
```

転置をするとこうなります。

```idris
[[1, 4],
 [2, 5],
 [3, 6]]
```

これを実装していきましょう。

Type: `n` 行 `m` 列の行列を `Vect n (Vect m a)` で表現するとして転置する関数 `transposeMat` の型を定義します。

```idris
transposeMat: (matrix: Vect n (Vect m a)) -> Vect m (Vect n a)
```


Define: `addclase` と `casesplit` を適用します。

```idris
transposeMat: (matrix: Vect n (Vect m a)) -> Vect m (Vect n a)
transposeMat [] = ?transposeMat_rhs_1
transposeMat (x :: xs) = ?transposeMat_rhs_2
```

Type: それぞれのHoleの型を調べます。まずは `transposeMat_rhs_1` 。

```idris
- + Main.transposeMat_rhs_1 [P]
 `--                     a : Type
                         m : Nat
     ------------------------------------------
   Main.transposeMat_rhs_1 : Vect m (Vect 0 a)
```

`Vect m (Vect 0 a)` を返さないといけないようですね。

Refine: `Vect 0 a` は `[]` のことなので `[]` を `m` 個用意すればよさそうです。同じ値を複数個用意するのは `replicate` 関数があるのでそれを使いましょう。

```idris
import Data.Vect using (Vect, replicate)
-- ファイル先頭のimportに `replicate` を追加する

transposeMat: (matrix: Vect n (Vect m a)) -> Vect m (Vect n a)
transposeMat {m} [] = replicate m []
transposeMat (x :: xs) = ?transposeMat_rhs_2
```


Type: `transposeMat_rhs_2` の型も見ます。


```idris
- + Main.transpose_rhs_2 [P]
 `--                     a : Type
                         m : Nat
                         x : Vect m a
                       len : Nat
                        xs : Vect len (Vect m a)
     ------------------------------------------------
   Main.transposeMat_rhs_2 : Vect m (Vect (S len) a)
```

おおまかには `Vect len (Vect m a)` をベースに `Vect m (Vect (S len) a)` を作ることになりそうです。 `m` と `len` が逆になってるので扱いづらいですね。ところで今実装している `transpose` がまさに `m` と `len` を入れ替える操作をするのでそれを使いましょう。

Refine: `transposeMat` を再帰呼出します

```idris
transposeMat: (matrix: Vect n (Vect m a)) -> Vect m (Vect n a)
transposeMat {m} [] = replicate m []
transposeMat (x :: xs) = let mat = transposeMat xs in
                         ?transposeMat_rhs_2
```

Type: `transposeMat_rhs_2` の型を調べます

```idris
- + Main.transposeMat_rhs_2 [P]
 `--                        a : Type
                            m : Nat
                            x : Vect m a
                          len : Nat
                           xs : Vect len (Vect m a)
                          mat : Vect m (Vect len a)
     ---------------------------------------------------
      Main.transposeMat_rhs_2 : Vect m (Vect (S len) a)
```

`mat: Vect m (Vect len a)` から `Vect m (Vect (S len) a)` を作るようになりました。かなり近くなりましたね。

Refine: ここで練習問題の指示で `zipWith : (a -> b -> c) -> Vect n a -> Vect n b -> Vect n c` を使うことになっています。 `mat` と同じ要素数の `Vect` は `x` なので `x` と `mat` を `zipWith` で組み合わせてあげることになります。3つの型を眺めてみましょう。

```idris
      x : Vect m a
    mat : Vect m (Vect len a)
zipWith : (a -> b -> c) -> Vect n a -> Vect n b -> Vect n c
------------------------------------------------------------
 result : Vect m (Vect (S len) a)
```

`a` と `Vect len a` から `Vect (S len) a` を作れる関数を用意すればできそうですね。そしてそれは `::` です。つまり以下のような実装になります。

```idris
transposeMat: (matrix: Vect n (Vect m a)) -> Vect m (Vect n a)
transposeMat {m} [] = replicate m []
transposeMat (x :: xs) = let mat = transposeMat xs in
                         zipWith (::) x mat
```

これでコンパイルが通ります。REPLで試してみましょう。

```idris
Idris> transposeMat [[1, 2, 3], [4, 5, 6]]
[[1, 4], [2, 5], [3, 6]] : Vect 3 (Vect 2 Integer)
```

正しく動いていそうです。

型駆動開発で行列の転置操作を実装できました。

# 本章のまとめ

本章では型駆動開発に触れました。その神髄に届く程ではないですが、十分雰囲気が伝わったのではないでしょうか。更なる情報はIdris作者による書籍を参考にして下さい。

[Type Driven Development with Idris](https://www.manning.com/books/type-driven-development-with-idris)
