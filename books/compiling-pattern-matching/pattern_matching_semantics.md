---
title: パターンマッチの意味
---

ここまでの説明では、「パターンが値にマッチする」という概念が何を意味するかを明らかにしてきませんでした。本章では、「パターンが値にマッチする」とはどういうことかを明確にします。また、マッチせずに「次のパターンにフォールバックする」とは何かも明確にします。これらの概念を明確にすることで、最適化やコンパイルをするときに自信をもって変換が正しいといえるようになります。

まずは、「パターンが値にマッチする」とはどういうことかを考えます。そのために、まずは「パターン `p` が値 `v` にマッチするか」を検査する仮想的な関数 `does_match` を定義することを目標とします。

```sml
(* does_match : pattern -> value -> bool *)
does_match p v = ...
```

パターン`p`の種類ごとに、どのような値にマッチするかを考えていきましょう。**変数パターン**、**ワイルドカードパターン**、**タプルパターン**、それに代数的データ型に対する**コンストラクタパターン**の順番に考えていきます。

変数パターンについては明白です。どのような値についてもマッチします。

ワイルドカードパターンは、主に名前に関係するので、マッチの挙動は変数パターンと同じと考えてかまいません。実際、ワイルドカードパターンを実装するとしたら、「コンパイラが生成した適当な名前の変数に置き換える」という挙動になるでしょう。つまり、以下のようなワイルドカードパターンがあったとしたら、

``` sml
_ => 〈expr〉
```

以下のように適当な変数（この例では`_anonymous_variable`）を使って変数パターンに置き換えればいいということです。

```sml
_anonymous_variable => 〈expr〉
```

次はタプルパターンです。これも比較的簡単です。列挙したパターンと値がそれぞれマッチすればマッチします。パターンと値の要素数が違うケースは考えなくていいことに注意してください。タプルの要素数についてはパターンマッチをする前に型で検査されているからです。

最後のコンストラクタパターンは少し考慮が必要です。まず、パターンのコンストラクタと値のコンストラクタが一致しなければマッチしません。両者のコンストラクタが一致した場合には、引数同士のマッチを確認します。それもマッチすれば、全体もマッチしています。マッチしていなければ、全体もマッチしません。

まとめると、`c`および`c'`をコンストラクタ、 `p`および`p1` 〜 `pn` をパターン、 `v`および`v1` 〜 `vn` を変数とすれば、`does_match`を以下のような疑似コードで書けます。

```sml
  (* 変数なら必ずマッチ *)
fun does_match v v = true
  (* タプルは各パターンがマッチすればマッチ *)
  | does_match (p1, p2, ...,pn) (v1, v2, ..., vn) =
      does_match p1 v1 andalso does_match p2 v2 andalso
                   ... andalso does_match pn vn
  (* パターンと値のコンストラクタが異なればマッチしない *)
  | does_match (c p) (c' v) = false
  (* パターンと値のコンストラクタが一致すればそれぞれの引数同士を検査 *)
  | does_match (c p) (c v) = does_match p v
```

`does_match`の定義方法がわかったところで、今度はcase式全体について考えます。case式は、条件式と、「`|` で結合された複数の節」で構成されています。そして複数の節について、「1つめの節のパターンにマッチすればその腕を評価した結果を返し、マッチしなければ次の節を試す」という挙動です。

この挙動は、いかにも `if` と `else` で書けそうです。しかし、それではあまりにも特定の実装に依存してしまいます。ここでは具体的な手続きを求めたいのではなく、挙動を定義したいだけなので、もう少し抽象化して `case` 式の意味を定義することを考えてみましょう。`case` 式の意味を定義するためには、やや天下り的ですが、以下の3つの道具が必要です。


$$
\newcommand{\angledtt}[1]{\langle\mathtt{#1}\rangle}
$$

* 単体の節の挙動を表す仮想的な式 $\newcommand{\angledtt}[1]{\langle\mathtt{#1}\rangle}\mathit{match}(\angledtt{cond}, \angledtt{pattern}, \angledtt{expr})$
* パターンマッチに失敗したことを表す仮想的な式 $\mathit{FAIL}$
* 節をつなぐ仮想的な中置演算子 $\|$

1つめの道具である $\newcommand{\angledtt}[1]{\langle\mathtt{#1}\rangle}\mathit{match}(\angledtt{cond}, \angledtt{pattern}, \angledtt{expr})$ は、 `does_match` を使った以下のような規則により、節の挙動を表現します。

$$
\newcommand{\angledtt}[1]{\langle\mathtt{#1}\rangle}

\mathit{match}(\angledtt{cond}, \angledtt{pattern}, \angledtt{expr}) 
\qquad =
\begin{cases}
\angledtt{expr}, & \mathtt{does\_match(\angledtt{pattern}, \angledtt{cond})}\text{の場合} \\
\mathit{FAIL}, &   \text{それ以外の場合} \\
\end{cases}

$$

上記の定義では2つめの道具である $\mathit{FAIL}$ も使っています。$\mathit{FAIL}$ は、3つめの道具である中置演算子 $\|$ の規則でも登場します。中置演算子 $\|$ の規則を以下に示します。

$$
\begin{align*}
\mathtt{expr1} & \| & \mathtt{expr2} & \rightarrow & \mathtt{expr1} \\
\mathit{FAIL}  & \| & \mathtt{expr} & \rightarrow & \mathtt{expr} \\
\mathtt{expr}  & \| & \mathit{FAIL} & \rightarrow & \mathtt{expr} \\
\mathit{FAIL}  & \| & \mathit{FAIL} & \rightarrow & \mathit{FAIL}
\end{align*}
$$

中置演算子の先頭の規則から、パターンマッチが「最初にマッチした節を優先する」というルールが見てとれると思います。また、これらの規則からは、この中置演算子が結合的であることも見てとれるでしょう。そこで以後では、特に $(e1\ \|\ e2)\ \|\ e3$ などと括弧書きをせず、 $e1\ \|\ e2\ \|\ e3$ と書くことにします。

これら3つの道具を使うと以下の `case` 式は、

```sml
case 〈cond〉 of
    〈pattern1〉 => 〈expr1〉
  | 〈pattern2〉 => 〈expr2〉
  ...
```

以下のような仮想的なコードへと書き換えられます。

$$
\newcommand{\angledtt}[1]{\langle\mathtt{#1}\rangle}

\begin{align*}
& \mathit{match}(\angledtt{cond}, \angledtt{pattern1}, \angledtt{expr1}) \\
\| & \mathit{match}(\angledtt{cond}, \angledtt{pattern2}, \angledtt{expr2})
\end{align*}
$$

さて、こうして変換した仮想的な式にどういう意味があるのか、具体例で確かめてみましょう。いま、以下のようなSMLの式を考えます。

``` sml
case x of
  Empty => 0 + 1
  Full _ => 1 + 1
```

このcase式を3つの道具に従って書き換えると以下の仮想的なコードになります。

$$
\mathit{match}(\mathtt{x}, \mathtt{Empty}, \mathtt{0 + 1})\ \|\ \mathit{match}(\mathtt{x}, \mathtt{Full\ \_}, \mathtt{1 + 1})
$$

ここで `x` が `Empty` だったとしましょう。すると `Empty` に対しては `does_match` なので `0 + 1` になりますが、 `Full` に対しては `does_match` でないので$\mathit{FAIL}$になります。そして $\|$ の規則から左辺の式 `0 + 1` になり、それを評価して値 `1` になります。

$$
\begin{array}{cl}
& \mathtt{0 + 1}\ \|\ \mathit{FAIL} \\
\Rightarrow & \mathtt{0 + 1} \\
\Rightarrow & \mathtt{1}
\end{array}
$$

このように、書き換えられた仮想的なコードは、元のcase式のパターンマッチの挙動を表現できていることがわかると思います。

では、なぜこのような書き換えを考えるのでしょうか？`does_match` を使うのであれば、このような書き換えをする必要はなく、 `if` / `else` で定義するほうが簡単に思えるかもしれません。中には現実のコンパイラが生成したコードも`if` / `else` と同じ動きをすると思っている方もいるでしょう。

しかしこのような書き換えを考えると、$\mathit{FAIL}$をうまく実装することで処理が効率的になったり、代数的データ型の性質と合わせて多方向分岐を上手く使うことで良質なコードを生成したりできるようになるのです。

