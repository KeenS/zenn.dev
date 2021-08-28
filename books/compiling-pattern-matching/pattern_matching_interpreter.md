---
title: パターンマッチのインタプリタ実装
---

前回はパターンマッチの意味について調べ、パターンが値にマッチするとはどういうことか、マッチせずに次のパターンにフォールバックするとはどういうことかを明確にしました。
本項ではこの意味に基いてインタプリタを実装します。
また、本記事を通してコンパイルのソースとなるSML風のパターンマッチ言語 `match` を導入します。
インタプリタを実装することで実行イメージが沸き、意味が分かりやすくなるでしょう。

#### パターンマッチの意味の復習
まずはパターンが値にマッチするとはどういうことか思い出しましょう。
前回以下のような道具を用意したのでした。

* パターンが値にマッチするか判定する関数 `does_match`
* 単体の節の挙動を表わす仮想的な式 $\mathit{match}(\angledtt{cond}, \angledtt{pattern}, \angledtt{expr})$
* パターンマッチに失敗したことを表す仮想的な式 $\mathit{FAIL}$
* 節を繋ぐ仮想的な中置演算子 $\|$

ここでそれぞれは以下のように与えられるのでした。

`does_match`

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


$\mathit{match}(\angledtt{cond}, \angledtt{pattern}, \angledtt{expr})$

$$\small
\begin{multlined}
\mathit{match}(\angledtt{cond}, \angledtt{pattern}, \angledtt{expr}) \\
\qquad =
\begin{cases}
\angledtt{expr}, & \mathtt{does\_match(\angledtt{pattern}, \angledtt{cond})}\text{の場合} \\
\mathit{FAIL}, &   \text{それ以外の場合} \\
\end{cases}
\end{multlined}
$$

$\|$ と $\mathit{FAIL}$

$$\small
\begin{array}{l@{\ }c@{\ }lcl}
\mathtt{expr1} & \| & \mathtt{expr2} & \rightarrow & \mathtt{expr1} \\
\mathit{FAIL}  & \| & \mathtt{expr} & \rightarrow & \mathtt{expr} \\
\mathtt{expr}  & \| & \mathit{FAIL} & \rightarrow & \mathtt{expr} \\
\mathit{FAIL}  & \| & \mathit{FAIL} & \rightarrow & \mathit{FAIL}
\end{array}
$$

これらを基にして次に進みます。

#### 式のデータ表現

この $\mathit{match}$ の挙動のイメージを掴むために簡単なSMLのサブセットの言語を定義し、インタプリタを実装してみましょう。
この言語では`タプル` 、 `コンストラクタ` 、`変数` 、 `case` 式、 `タプルパターン` 、 `コンストラクタパターン` 、 `変数パターン` が使えます。

これらを組み合わせると例えば以下のような式が書けることになります。


``` sml
(* datatype bool = True | False *)
case  (True, False) of
    (True, True) => False
  | (True, False) => True
  | (False, True) => True
  | (True, True) => False
```

論理演算のXOR相当の式が書けました。

こういうコードを表現できるデータ型を定義していきます。
前回紹介したとおり、代数的データ型はタグとデータに分離されるのでした。

データはそのままですが、タグは数値に変換されます。
つまり `datatype bool = True | False` と定義したときに
`True` は0に、 `False` は1に翻訳されます。

その前提を踏まえた上で、ASTを表現するデータ型をRustで定義していきましょう。

まずはASTの前に、ユーティリティとしてシンボル、型IDのデータ型を用意します。

``` rust
#[derive(Debug, Clone, Hash, PartialEq, Eq)]
/// 名前を表わすデータ型
pub struct Symbol(String, u32);

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
/// 型を表わすデータ型
pub struct TypeId(String);
```

`struct Name { /* definition */ }` は構造体を定義する構文です。
その前にある `#[...]` はアトリビュートで、 `derive(...)` アトリビュートはあると便利なメソッドを自動で定義するアトリビュートです。
本稿では本質的でないのであまり気にする必要はありません。

前回、変数 `x` 、 `y` などは事前に処理されて `x1` 、 `y2` のように番号つきで解釈すると解説しました。
それを直接表現したのが `struct Symbol(String, u32)` です[^symbol_name]。

[^symbol_name]: 現実のコンパイラでは名前の `String` の箇所は省かれて、数値だけになることが多いようです。ここではデバッグするときの分かりやすさのために名前も保持します。

`TypeId` も `Symbol` と同様に型名を表わすデータ型です。例えば `TypeId("bool")` などを表現します。
こちらは今回のケースでは名前被りを想定しなくていいので付番がなく、名前のみです。

`TypeId` にはコンストラクタも用意しておきましょう。

``` rust
impl TypeId {
    pub fn new(s: impl Into<String>) -> Self {
        Self(s.into())
    }
}
```

`impl Name { /* definition */ }` はデータ型にメソッドを定義する構文です。

`Symbol` は添字を被りなく生成したいので単純なコンストラクタは用意せず、後程システマティックにシンボルを生成する仕組みを導入します。

準備が整ったので、本記事で扱うミニ言語たちのASTを表現するデータ型を定義していきます。

##### `case` 言語のデータ表現

まずはSMLの小さなサブセットになる `case` 言語を定義します。

``` rust
mod case {
    use super::*;

    /// 式
    #[derive(Debug, Clone)]
    pub enum Expr {
        /// (expr1, expr2, ...)
        Tuple(Vec<Expr>),
        /// inj<descriminant>(data)
        Inject {
            descriminant: u8,
            data: Option<Box<Expr>>,
        },
        /// case cond of pattern1 => expr1 | pattern2 => expr2 ...
        Case {
            cond: Box<Expr>,
            ty: TypeId,
            clauses: Vec<(Pattern, Expr)>,
        },
        /// x
        Symbol(Symbol),
    }

    /// パターン
    #[derive(Debug, Clone)]
    pub enum Pattern {
        /// (pat1, pat2, ...)
        Tuple(Vec<Pattern>),
        /// c<descriminant>(pattern)
        Constructor {
            descriminant: u8,
            pattern: Option<Box<Pattern>>,
        },
        /// x
        Variable(Symbol),
    }
}
```

ASTを表わすデータ型は式（expression）を表わす（ `Expr`）とパターン（pattern）を表わす（`Pattern`）のシンプルな構成です。
いくつか解説を加えておきます。

このあと出てくる別の言語と名前が被らないように `mod case { /* ... */ }` で名前空間を区切っています。

`enum Name { /* definition */ }` は代数的データ型を定義します。
ヴァリアント（Rustにおけるコンストラクタ）の定義方法が3種類あります。
引数なし、`()` 囲みのカンマ区切り（例えば `Expr::Tuple`）、 `{}` 囲みの `名前: 型`（例えば `Expr::Inject`）です。
SMLと異なり、ヴァリアントの引数は複数とれます。


随所にでてくる `Box<...>` はRustでのポインタ型（の一種）です。データ構造の制約からこれが必要になりますが、本稿ではあまり気にする必要はありません。
`Expr::case` に出てくる `(Pattern, Expr)` は `Pattern` と `Expr` のタプル（組）を表わす型です。

`Expr::Inject` はコンストラクタの適用です。適用するコンストラクタを表わす数値と、必要であれば引数の式を受け取ります。
見て分かるとおり、代数的データ型がどのコンストラクタで作られたかを表わすデータはdescriminant（判別子）と呼んでいます。
どの代数的データ型にinject（注入）するかの情報、つまり型情報は引数にありません。引数に含む設計もありますが、パターンマッチのコンパイルだけなら不要なので省きました。

`Pattern` はパターンを表わすデータ型です。一見式（`Expr`）と似ていますが、似ているだけで別物です。
似ているからとまとめてしまうとパターンマッチをコンパイルするときに困ることになります。

##### `case` 言語の表現例

定義だけだと何ができるか分かりづらいのでこのデータ型の定義を用いて `case` 言語のコードを表現してみましょう。
先程のXORを計算するコードは以下のデータ型で表せます。

``` rust
use case::*;
use Expr::*;

// `false` （値）のつもり
let falsev = Expr::Inject {
    descriminant: 1,
    data: None,
};
// `true` （値）のつもり
let truev = Expr::Inject {
    descriminant: 0,
    data: None,
};
// `false` （パターン）のつもり
let falsep = Pattern::Constructor {
    descriminant: 1,
    pattern: None,
};
// `true` （パターン）のつもり
let truep = Pattern::Constructor {
    descriminant: 0,
    pattern: None,
};

// 2要素タプルの便利メソッド
fn tuple2p(p1: Pattern, p2: Pattern) -> Pattern {
    Pattern::Tuple(vec![p1, p2])
}

// case (false, false) of
//     (true, true) => false
//   | (true, false) => true
//   | (false, true) => true
//   | (false, false) => false
Case {
    cond: Box::new(Tuple(vec![falsev.clone(), falsev.clone()])),
    ty: TypeId::new("tuple2"),
    clauses: vec![
        (tuple2p(truep.clone(), truep.clone()), falsev.clone()),
        (tuple2p(truep.clone(), falsep.clone()), truev.clone()),
        (tuple2p(falsep.clone(), truep.clone()), truev.clone()),
        (tuple2p(falsep.clone(), falsep.clone()), falsev.clone()),
    ],
}
```

素のデータ表現なので慣れないと読みづらいかもしれませんが、これが `case` 言語の内部表現になります。


##### `case` 言語のインタプリタ

次にこの `case` 言語のインタプリタを書きます。
多少言語処理系に覚えのある方だとこのミニ言語のインタプリタは造作なく書けるでしょう。
今回の実装もそんなに大きくはありません。

まずはインタプリタのデータ型を用意します。
前回紹介したとおり `case` 式がスコープを導入するのでそれを管理するためにハッシュマップを保持します。

``` rust
struct CaseInterp {
    scope: HashMap<Symbol, case::Value>,
}

impl CaseInterp {
    pub fn new() -> Self {
        Self {
            scope: HashMap::new(),
        }
    }

   // ...
}
```

インタプリタの実装はこの `CaseInterp` のメソッドとして実装する訳です。

インタプリタで評価した結果は値（value）になります。式とは違って `case` 式などの計算がありません。値も `case` 言語の一部として定義しましょう。

``` rust
mod case {
    // ...

    /// 値
    #[derive(Debug, Clone)]
    pub enum Value {
        /// (value1, value2, ...)
        Tuple(Vec<Value>),
        /// c<descriminant>(value)
        Constructor {
            descriminant: u8,
            value: Option<Box<Value>>,
        },
    }
}
```

インタプリタの実装に戻りましょう。
パターンマッチにでてくる $\mathit{FAIL}$ に相当するデータ型 `Fail` と、最終的にパターンマッチに失敗したことをユーザに伝えるデータ型 `Match` を定義します。

``` rust
struct Fail;
struct Match;
```

準備が整ったのでインタプリタを実装しましょう。
インタプリタの心臓部分、 `eval` は以下のように書けます。

``` rust
impl CaseInterp {
    pub fn eval(&mut self, expr: case::Expr) -> Result<case::Value, Match> {
        use case::Expr::*;
        // どの式かでパターンマッチ
        match expr {
            // タプルならそれぞれの要素を評価したあとにタプルを作る
            Tuple(tuple) => tuple
                .into_iter()
                // case::Expr -> Result<case::Value, Match>
                .map(|e| self.eval(e))
                // 普通に集めると Vec<Result<case::Value, Match>> だが、
                // Rustは注釈を与えればResultとVecを入れ替えられる
                .collect::<Result<Vec<_>, Match>>()
                .map(case::Value::Tuple),
            // `case` 式ならパターンマッチをする（後述）
            Case { cond, clauses, .. } => {
                let cond = self.eval(*cond)?;
                let ret = clauses
                    .into_iter()
                    .map(|(pat, arm)| self.pattern_match(pat, cond.clone()).map(|()| arm))
                    .fold(Err(Fail), |acc, current| acc.or(current));
                match ret {
                    Ok(e) => self.eval(e),
                    Err(Fail) => Err(Match),
                }
            }
            // シンボルはスコープから名前を解決する（後述）
            Symbol(s) => Ok(self.resolve(&s)),
            // コンストラクタなら引数があれば評価し、値を作る
            Inject { descriminant, data } => Ok(case::Value::Constructor {
                descriminant,
                value: match data {
                    None => None,
                    Some(d) => Some(Box::new(self.eval(*d)?)),
                },
            }),
        }
    }

    // ...
}
```

概ねコメントに書いた通りですが、2点補足があります。

まずはシンボルの扱いについて。
`case` 言語はコンパイラによる前段処理が済んだあとを想定した言語です。
シンボルは同じものは添字が同じに、見た目上の名前が同じでも違うものは添字が異なっていることを想定しています。
すると名前被りを心配する必要がないのでどこでシャドイングされているかなどを気にせずにスコープにある変数から値を取り出すだけでよくなります。

ということで `Symbol` の腕で使われている `resolve` メソッドは以下のように定義できます。

``` rust
impl CaseInterp {
    // ...

    fn resolve(&mut self, s: &Symbol) -> case::Value {
        self.scope[s].clone()
    }

    // ...
}
```

シャドイングを気にしなくていいのでスコープから値を取り出すだけです。

ハッシュマップに対して `hashmap[key]` でアクセスしていますが、これはRustでキーに対応する値があればそれを、対応する値がなければパニック（プログラムの異常終了）する記法です。
`case` 言語では存在しない変数の参照はコンパイルの前段で弾かれている想定なので、対応する値は必ずあります。
想定から外れて値がなかった場合はバグなのでパニックしても問題ありません。

次に `Case` についてです。主題がパターンマッチについてなのでここの比重は大きくなりますね。
`Case` の節の腕を抜き出すと以下のようになっています。

``` rust
// 条件式を評価
let cond = self.eval(*cond)?;
// それぞれの節について、最初から順番にパターンマッチを試す。
// マッチした節の腕を取り出す
let ret = clauses
    .into_iter()
    .map(|(pat, arm)| self.pattern_match(pat, cond.clone()).map(|()| arm))
    .fold(Err(Fail), |acc, current| acc.or(current));
// マッチした節があればその腕を評価する。なければパターンマッチエラー。
match ret {
    Ok(e) => self.eval(e),
    Err(Fail) => Err(Match),
}
```

まず、条件部分を評価します。
評価して、条件部分の値が決まったらそれを上から順にパターンマッチにかけます。
パターンマッチに成功したものがあれば、それの腕を取り出して評価します。

上記の処理をイテレータを使って実装しています。
イテレータを使ったコードに読み慣れない方は以下のような手続的なコードの方が読みやすいかもしれません。

``` rust
// let ret = clauses
//     .into_iter()
//     .map(|(pat, arm)| self.pattern_match(pat, cond.clone()).map(|()| arm))
//     .fold(Err(Fail), |is_match, ret| ret.or(is_match));
// 上記コードは以下に等価

let mut ret = Err(Fail);
for (pat, arm) in clauses {
    let current = self.pattern_match(pat, cond.clone()).map(|()| arm);
    ret = ret.or(current);
}
```

ある程度好みはありますが、Rustではイテレータを使った書き方の方がよく使われるようです。

上記で登場した、まだ紹介していない関数 `pattern_match` の返り型は `Result<Expr, Fail>` です。
$\mathit{match}$ はマッチに成功すれば腕の式を、失敗すれば $\mathit{FAIL}$ を返すのでこれに対応した実装が `Result` という訳です。

TODO: `Result` の説明必要？

さらに、ここで使われている `Result::or` は $\|$ と同じ挙動をします。
すなわち、以下の表明が全て成り立ちます。

``` rust
assert_eq!( Ok(expr1).or( Ok(expr2)), Ok(expr1));
assert_eq!( Ok(expr) .or(Err(Fail)),  Ok(expr));
assert_eq!(Err(Fail) .or( Ok(expr)),  Ok(expr));
assert_eq!(Err(Fail) .or(Err(Fail)),  Err(Fail));
```

これで $\|$ の実装として `Result::or` を使っていいことが分かりました。

総合して、`Case` の腕の式は `pattern_match` が $\mathit{match}$ 相当の挙動をすると仮定すれば $\mathit{match}$ 、 $\|$ 、 $\mathit{FAIL}$ の挙動を実装できていることが確かめられす。

では、ここで使われている `pattern_match` の実装を見てみましょう。

``` rust
impl CaseInterp {
    // ...

    fn pattern_match(&mut self, pattern: case::Pattern, value: case::Value) -> Result<(), Fail> {
        match (pattern, value) {
            (case::Pattern::Variable(s), v) => {
                self.scope.insert(s, v);
                Ok(())
            }
            (case::Pattern::Tuple(ps), case::Value::Tuple(vs)) => ps
                .into_iter()
                .zip(vs)
                .map(|(p, v)| self.pattern_match(p, v))
                .fold(Ok(()), |is_match, ret| ret.and(is_match)),
            (
                case::Pattern::Constructor {
                    descriminant: c1,
                    pattern,
                },
                case::Value::Constructor {
                    descriminant: c2,
                    value,
                },
            ) => {
                if c1 == c2 {
                    match (pattern, value) {
                        (None, None) => Ok(()),
                        (Some(p), Some(v)) => self.pattern_match(*p, *v),
                        _ => unreachable!("internal error, maybe type checker's bug"),
                    }
                } else {
                    Err(Fail)
                }
            }
            _ => unreachable!("internal error, maybe type checker's bug"),
        }
    }

    // ...
}
```

`match` がRustの予約語なので `pattern_match` という名前を使っていますが、 $\mathit{match}$ を意識した関数です。
概ね、パターンマッチの挙動を定義するときに使った仮想的な関数 `does_match` と $\mathit{match}$ を融合させたような実装です。
`does_match` と異なり変数があればそれをマッチした値に束縛し、スコープに入れています。
また、マッチの結果をbooleanではなく腕の式または `Fail` で表現しています。

これで `pattern_match` の中身は $\mathit{match}$ と同じような挙動であることが分かりました。

インタプリタの心臓部分、 `eval` ができたのでこのインタプリタを実際に動かして試すことができます。

先程のコード(をデータ型で表わしたもの)を実行してみます。
細かな実行部分は省いて実行したコードと結果だけ貼ると以下のようになります。
<!-- 実行部分のコードあった方がいい？ -->

``` console
case (inj <1>(), inj <1>(), ) of
   | (<0>(), <0>(), ) => inj <1>()
   | (<0>(), <1>(), ) => inj <0>()
   | (<1>(), <0>(), ) => inj <0>()
   | (<1>(), <1>(), ) => inj <1>()

evaled to

c <1>()
```

ここで `inj<descrimiant>(data)` は `Expr::Inject` を表わしていて、 `c<descriminant>(data)` は `Value::Constructor` を表わしています。
`datatype bool = True | False` で `True` が `0` 、 `False` が `1` でしたので、`inj <1>()` と `c <1>()` は `False` を表わします。
`(False, False)` を評価して `False` が返ってきたので正しく実装できているようです。
他にも引数のある代数的データ型や変数なども各々試してみて下さい。

このインタプリタはナイーブなものですがその分挙動としては理解しやすいでしょう。
中には現実のコンパイラが生成したコードもこれと同じ動き、すなわちパターンマッチでパターンを1つ1つマッチするか検査をすると思っている方もいるかもしれません。
しかし、多方向分岐を上手く使うことでこれよりもずっと良質なコードを生成できます。
これからこれの挙動を保ったままコンパイルするコンパイラを書いていきます。
