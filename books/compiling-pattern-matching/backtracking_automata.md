---
title: パターンマッチのコンパイル：バックトラック法
---

まず紹介するのはバックトラック法です。ここで紹介するアルゴリズムは参考文献1（[後半まとめの章](conclusion_of_second_half)参照）に基くものですが、SMLのパターンマッチに合わせて多少の改変を加えています。具体的にはタプルを追加したのと代数的データ型の各コンストラクタの引数を高々1つに制限しています。

バックトラック法でコンパイルされたパターンマッチは、マッチ判定中にパターンを先に進んだり後に戻ったりするのが特徴です。そのため最悪のケースで `case` 式に書かれた全てのパターンを辿ることもあります。一方でコードはそれ以上複雑にならないので全てのパターンの量に比例したサイズのコードにしかなりません。

実装コードだけみても分かりづらいと思うのでまずはアルゴリズムの基本的な部分を押さえ、そして実装に移りたいと思います。

## 動作イメージ

まずはコンパイルする前に、どういう動作をするコードを出力したいのかのイメージを掴みましょう。

以下のコードを例にとって動作を考えます。

``` sml
case ([], [true]) of
    (_,  []) => []
  | ([], _ ) => []
  | (x::xs, y::ys) => (x, y) :: zip xs ys
```

まずはタプルの先頭のみに注目します

``` sml
case ([], ...) of
    (_,  ...) => []
  | ([], ...) => []
  | (x::xs, ...) => (x, y) :: zip xs ys
```

そしてタプルの先頭のパターンの種類をみて節を分割します。コンストラクタと変数（ワイルドカード）パターンの境目で分けます。

``` sml
case ([], ...) of
    (_,  ...) => []

  | ([], ...) => []
  | (x::xs, ...) => (x, y) :: zip xs ys
```

パターンマッチは上から行うのでまずは最初の1つのパターンに注目します。

``` sml
case ([], ...) of
    (_,  ...) => []

    ...
```

ワイルドカードパターンは無条件にマッチするので次に進みます。

``` sml
case ([], [true]) of
    (_,  []) => []

    ...
```

次はタプルの2つ目の要素に注目します。

``` sml
case (..., [true]) of
    (..., []) => []
  ...
```

残念ながら `[]` と `[true]` （コンストラクタでいうと `::`）なのでマッチしませんでした。

1行目は失敗したので、次のかたまりにうつります。先程は2つ目の要素で失敗したので1つ巻き戻して1つ目の要素を調べます。

``` sml
case ([], ...) of
  ...
  | ([], ...) => []
  | (x::xs, ...) => (x, y) :: zip xs ys
```

複数のコンストラクタは同時に調べられるので2行目、3行目を同時に調べます。するとこれは1つ目の節にマッチします。

``` sml
case ([], ...) of
  ...
  | ([], ...) => []
  ...
```

1つ目の要素はクリアしたので、2つ目の要素に注目します。

``` sml
case (..., [true]) of
  ...
  | (..., _)    => []
  ...
```

ワイルドカードパターンは無条件にマッチするのでこれで決定し、 2行目が採用されます。結果、`[]` が返ります。

動作イメージが掴めたでしょうか。これに相当するコードを吐くコンパイラを作っていきます。

## 原理

パターンをコンパイルするのは一見すると複雑そうですね。ですがマッチに使われているパターンを調べてパターンの種類で分岐してあげると、1つ1つのルールはシンプルなものになります。また、パターン自体がネスト可能なのでコンパイルするアルゴリズムも再帰的なものになります。

まず、対象になる式は `case` 式、つまり `case <条件> of <パターン1> => <式1> | …` の形をした式なのでした。この `<条件>` と `<パターン>` の列それぞれに着目します。

### 条件部

`条件` の部分についてです。`条件` には必ず変数がくるようにしておきます。これを **条件変数** と呼ぶことにします [^cond-var]。条件部分を変数にするのはパターンのコンパイルの過程で条件部分をコピーすることがあるからです。もし変数以外の式がきたら、事前に `let` を使って変形しておきます。

[^cond-var]: 条件変数という名前はマルチスレッドの文脈で出てくるものと同じですが全く違うものです。ここでの条件変数は「条件部分に置く変数」を縮めただけの略語であることに注意して下さい。

``` sml
case [true] of
    [] => false
  | _  => true
```

↓

``` sml
let val tmp = [true] in
case tmp of
    [] => false
  | _  => true
end
```


構文の上での `条件` は1つだけですが、パターンマッチのコンパイル中は複数あるものとして扱います。これはタプルパターンをコンパイルするときに、タプルを一旦分解してから1つ1つ順にコンパイルしていくのに必要だからです。もう少し具体的にいうと、パターンはネスト可能なので木になりますが、それを深さ優先探索で処理したいため *条件式をスタックで保持します* 。これを **条件変数スタック** と呼ぶことにします。

### パターン部

`パターン` の部分についてです。パターンの部分をどうデータで表現するか、そしてそのデータをどう処理するかを押さえましょう。

パターンも条件に連動して複数に分割されます。既に節は複数ありますから、そこからパターンを複数に分割するとなると、パターンを保持するデータ構造は2次元配列になります。やや用語の乱用の嫌いがありますが、これを **パターン行列** と呼ぶことにします。行列とはいうもののかけ算や足し算をしたりだとか行列式を求めたりだとかはしないので線形代数が苦手でも安心して下さい。節には腕もついているので、パターン行列にはこっそり腕も入れておくことにします。

おおまかには条件変数スタックとパターン行列が、パターンマッチをコンパイルする関数への入力になります。具体例をみましょう。 `zip` 関数のパターンマッチをコンパイルすることを考えます。

``` sml
case (xs, ys) of
  ([], _) => []
| (_, []) => []
| (x::xs, y::ys) => (x, y) :: zip xs ys
```

規約により、まずは条件式の部分を変数にします。

``` sml
let val tmp1 = (xs, ys) in
case tmp1 of
  ([], _) => []
| (_, []) => []
| (x::xs, y::ys) => (x, y) :: zip xs ys
end
```

そしてここから `case` 式をコンパイルするので疑似コードで書くとこのようなスタートになります。

``` sml
compile [tmp1] [
  ([([], _)]        => []),
  ([(_, [])]        => []),
  ([(x::xs, y::ys)] => (x, y) :: zip xs ys)
]
```

この `compile` の第一引数が条件変数スタック、第二引数がパターン行列です。

この記法は具体的な入力をイメージしやすいので以後、扱いたい入力を表わすのに使っていきます。

上記のパターン行列をデータでの表現とし、以下でその処理を見ていきます。

`v` から始まるものを変数パターン、 `p` と `q` からはじまるものを一般のパターンとします。 `C` ではじまるものをコンストラクタ、 `c` ではじまるものを条件変数とし、 `e` ではじまるものを一般の式とします。

それではこの記法を用いて個別のルールを見ていきましょう。

#### 空則

単純なルールからいきましょう。パターンが空ならばそれは必ずマッチします。パターンマッチは上にあるものを優先するルールから、一番上の腕に変換されます。これは基底ケースに相当します。

つまり、以下のような入力では

``` sml
compile [] [
  ([] => e1),
  ([] => e2),
  ([] => e3),
  ...
]
```

以下のように腕の式にコンパイルできます。

``` sml
e1
```


ここでは行が1つ以上あることを仮定しています。SMLでは節が1つもない `case` 式は書けませんし、他の則でも行が少なくとも1つあるパターン行列しか生成しないのでこの仮定は妥当です。

これで空の場合は拾えたので以降は条件変数が1つ以上ある場合を扱います。

#### 変数則

もしパターン行列の1列目の全てのパターンが変数パターンなら、条件変数を1つ消費して次に進みます。

つまり、以下のような入力では

``` sml
compile [c1, c2, ...] [
  ([v11, p12, ...] => e1),
  ([v21, p22, ...] => e2),
  ([v31, p32, ...] => e3),
  ...
]
```

以下のように条件変数やパターンの先頭が消えた形にコンパイルします。

``` sml
compile [c2, ...] [
  ([p12, ...] => let val v11 = c1 in e1 end),
  ([p22, ...] => let val v21 = c1 in e2 end),
  ([p32, ...] => let val v31 = c1 in e3 end),
  ...
]
```

変数パターンはその変数を値に束縛するので `let` 式を導入しています。

条件変数が1つ減りました。このまま次の再帰に進みます。

#### コンストラクタ則

もしパターン行列の1列目のパターン全てがコンストラクタなら、条件変数スタックの先頭を `case` で分岐して次に進みます。ここの処理は少し複雑です。先頭のコンストラクタをキーにしてで節をグルーピングする処理が必要だからです。

先に少し具体的な例を考えましょう。以下のようなコンストラクタ `C1` 、 `C2` を持つ代数的データ型のパターンマッチを考えます。

``` sml
compile [c1, c2, ...] [
  ([C1, p12, ...] => e1),
  ([C2, p22, ...] => e2),
  ([C1, p32, ...] => e3),
]
```

コンストラクタ `C1` が2回登場していますね。これは同じコンストラクタをひとつにまとめて以下のように変換されてほしいです。

``` sml
case c1 of
    C1 => compile [c2, ...] [
            ([p12, ...] => e1),
            ([p32, ...] => e3),
          ]
  | C2 => compile [c2, ...] [
            ([p22, ...] => e2),
          ]
```


 `C1` パターンではじまる節を全て集めて `C1` の腕に置き、 `C2` パターンではじまる節を全て集めて `C2` の腕に置いています。

もう1つ具体例を考えましょう。今度はコンストラクタが引数をとる場合を考えます。 `C1` が引数をとり、 `C2` は引数をとらない場合を考えましょう。

``` sml
compile [c1, c2, ...] [
  ([C1 p11, p12, ...] => e1),
  ([C2    , p22, ...] => e2),
  ([C1 p31, p32, ...] => e3),
]
```

これはだいたい先程とおなじように変換されてほしいですね。ただしコストラクタパターンの引数にもパターンがあるのでそれも考慮しないといけません。そうすると以下のようになるはずです。

``` sml
case c1 of
    C1 tmp => compile [tmp, c2, ...] [
            ([p11, p12, ...] => e1),
            ([p31, p32, ...] => e3),
          ]
  | C2 => compile [c2, ...] [
            ([p22, ...] => e2),
          ]
```

`C1` の腕ではコンストラクタの引数を一旦変数パターンで受け取って、それを条件変数スタックの先頭にもってきています。また、それぞれのコンストラクタパターンの引数にあったパターンもパターン行列の1列目に追加しています。

これらの具体例を元にコンパイル処理を考えましょう。まず必要なのが1列目が同じコンストラクタパターンの行を1箇所にまとめる処理です。この処理をする関数を `specialize` と名付けます。 `specialize` はコンストラクタの判別子とパターン行列を引数にとってパターン行列を返します。

`specialize` ができたとします。コンストラクタ `C1` の判別子を `C1d` 、 `C2` の判別子を `C2d` と書くことにするると、`specialize` を使ってコンパイル処理はこう書けますね。

``` sml
compile [c1, c2, ...] pattern_matrix
```

↓

``` sml
case c1 of
    C1 arg => compile [arg, c2, ...] (specialize C1d pattern_matrix)
  | C2     => compile [c2, ...]      (specialize C2d pattern_matrix)
  | ...
```

もし、パターン行列の1列目に列挙されたコンストラクタ群が `c1` に定義されたコンストラクタ群より少なければデフォルト節も必要になります。その場合はパターンマッチに失敗するので $\mathit{FAIL}$ を置きます。

``` sml
case c1 of
    C1 arg => compile [arg, c2, ...] (specialize C1 pattern_matrix)
  | C2     => compile [c2, ...] (specialize C2 pattern_matrix)
  | ...
  | _ => FAIL
```


コンストラクタによって引数があったりなかったりするので、`case` を書くときは柔軟に対応します。それぞれの節の腕はコンストラクタの判別子を使って `pattern_matrix` を `specialize` しています。

`specialize` は、指定されたコンストラクタパターンの節のみを集めます。このとき、コンストラクタに引数があることもあるので柔軟な処理が必要です。おおまかには以下のような処理をします。

``` sml
specialize constructor pattern_matrix = {for each row of pattern_matrix}
  if pi1 is constructor     => collect it as ([pi2, ...]      => ei)
  if pi1 is constructor arg => collect it as ([arg, pi2, ...] => ei)
  else ignore
{end}
```

後程、この処理を実際のプログラミング言語で実装するので概略を覚えておいて下さい。

#### タプル則

もしパターン行列の1列目のパターン全てがタプルパターンなら、タプルを平滑化して次に進みます。

``` sml
compile [c1, c2, ...] [
  ([(q11, q12, ...), p12, ...] => e1),
  ([(q21, q22, ...), p22, ...] => e2),
  ...
]
```

↓

``` sml
case c1 of
    (tmp1, tmp2, ...) => compile [tmp1, tmp2, ..., c2, ...] [
      ([q11, q12, ..., p12, ...] => e1),
      ([q21, q22, ..., p22, ...] => e2),
      ...
    ]
```

$N$ 要素のタプルなら一時変数を $N$ 個用意して束縛します。とりたてて難しいところはありません。

#### 混合則

もしパターン行列の1列目に変数パターンもコンストラクタもあるなら、それを分割しましょう。


``` sml
compile [c1, c2, ...] [
  ([C11, p12, ...] => e1),
  ([v21, p22, ...] => e2),
  ...
]
```

上記のパターン行列は1列目にコンストラクタパターンと変数パターンが交じっています。1行目と2行目の先頭パターンがそれぞれコンストラクタパターン、変数パターンと異なっているのでそこで分割します。分割した結果、 `compile condvar pattern_matrix1 || compile condvar pattern_matrix2` という見た目になります。総合すると下記のような見た目になります。

``` sml
compile [c1, c2 ...] [
  ([C11, p12, ...] => e1),
] || compile [c1, c2] [
  ([v21, p22, ...] => e2),
  ...
]
```

この例では先にコンストラクタ、後に変数が出てきていますが、逆でも同じです。

パターン行列を分割するとそれぞれがカバーできる範囲も狭くなるので、マッチしなくなることがあります。マッチしなかったら次のパターン行列を試すのですが、そのために $\mathit{FAIL}$ と $\|$ を使います。上記の例だと1つ目のパターン行列はコンストラクタが1つしかないのでマッチに失敗するかもしれません。そのときはコンストラクタ則で $\mathit{FAIL}$ が生成されます。そして $\mathit{FAIL}$ は $\|$ で拾われて、2つ目のパターン行列に処理が移るという具合です

## 具体例

ルールだけだとどういう動きをするのか分かりづらいのでコンパイルの具体例をみてみましょう。
ネストしたパターンの `case` 式からネストしていないパターンの `case` 式に変換します。

動作イメージのときと同じ `zip` のパターンマッチを例にとりましょう。

```sml
case (xs, ys) of
    (_,  []) => []
  | ([], _ ) => []
  | (x::xs, y::ys) => (x, y) :: zip xs ys
```

これをコンパイルします。まずは条件部分を変数にして、 `compile` 関数からはじめます。このとき、 `v::vs` は脱糖すると `::` がコンストラクタ、 タプルの `(v, vs)` が引数であることに注意して下さい。

```sml
let val tmp1 = (xs, ys) in
  compile [tmp1] [
    ([(_,  [])]       => []),
    ([([], _ )]       => []),
    ([(x::xs, y::ys)] => (x, y) :: zip xs ys)
  ]
end
```

まずはパターン行列の1列目が全てパターンなので *タプル則* でタプルを平滑化します。

```sml
case tmp1 of
  (tmp2, tmp3) =>
    compile [tmp2, tmp3] [
      ([_,  []],      => []),
      ([[], _ ],      => []),
      ([x::xs, y::ys] => (x, y) :: zip xs ys)
    ]
```

次のパターン行列がでてきました。1列目に変数パターンとコンストラクタパターンがありますね。 *混合則* で2つに分割します。

```sml
compile [tmp2, tmp3] [
  ([_,  []]      => [])
]
||
compile [tmp2, tmp3] [
  ([[], _ ],      => []),
  ([x::xs, y::ys] => (x, y) :: zip xs ys)
]
```

$\|$ と $\mathit{FAIL}$ の具体的な実装はアルゴリズムの実装のときに考えることにします。

さて、それぞれのパターンをコンパイルしていきます。まず上の方。

``` sml
compile [tmp2, tmp3] [
  ([_,  []]      => [])
]
```

1列目はワイルドカードパターンだけです。これは *変数則* で以下のような `case` 式にコンパイルできます。

``` sml
case tmp2 of
    _ => compile [tmp3] [
      ([[]] => [])
    ]
```

次の1列目はコンストラクタパターンですので *コンストラクタ則* を適用します。ですが、これは網羅的ではありません。書いたパターンにマッチしなかった場合に（=default節に） `FAIL` をおぎないつつ、コンストラクタで分岐するコードにコンパイルします。

``` sml
case tmp2 of
    _ => case tmp3 of
             [] => compile [] [
               ([] => [])
             ]
           | _  => FAIL
```

条件変数スタックが空になったので *空則* を適用して腕の式になります。

``` sml
case tmp2 of
  _ => case tmp3 of
           [] => []
         | _  => FAIL
```

上のカタマリが終わったので次は下のパターン行列です。

```sml
compile [tmp2, tmp3] [
  ([[], _ ],      => []),
  ([x::xs, y::ys] => (x, y) :: zip xs ys)
]
```

これの1列目はコンストラクタパターンだけがあります。 *コンストラクタ則* を適用します。

```sml
case tmp2 of
    []      => compile [tmp3] [[_] => []]
  | :: tmp4 => compile [tmp4, tmp3] [[(x, xs), y::ys] => (x, y) :: zip xs ys]
```

上の `[]` 節の方はまたワルドカードパターンなので *変数則* を適用します。そのあと *空則* を適用します。一気にやってしまいましょう。

```sml
case tmp2 of
    []      => case tmp3 of _ => []
  | :: tmp4 => compile [tmp4, tmp3] [[(x, xs), y::ys] => (x, y) :: zip xs ys]
```

下の方はタプルパターンがでてきたので *タプル則* を適用します。

```sml
case tmp2 of
    []      => case tmp3 of _ => []
  | :: tmp4 => case tmp4 of
                   (x, xs) => compile [tmp3] [
                       [y::ys] => (x, y) :: zip xs ys
                   ]
```


また `::` パターンがでてきました。先程と同様なのでコンストラクタ則とタプル則の適用を同時にやってしまいます。

```sml
case tmp2 of
    []      => case tmp3 of _ => []
  | :: tmp4 =>
      case tmp4 of
          (x, xs) =>
             case tmp3 of
                 :: tmp5 =>
                     case tmp5 of
                         (y, ys) => compile [] [
                             ([] => (x, y) :: zip xs ys)
                         ]
               | _ => FAIL
```


最後は条件変数スタックが空になったので *空則* です。

```sml
case tmp2 of
    []      => case tmp3 of _ => []
  | :: tmp4 =>
      case tmp4 of
          (x, xs) =>
             case tmp3 of
                 :: tmp5 =>
                     case tmp5 of
                         (y, ys) => (x, y) :: zip xs ys
               | _ => FAIL
```

2つのパターンがコンパイルできました。

最後に、 $\mathit{FAIL}$  が残った場合の安全弁として $\|$ で $\mathit{FAIL}$ を拾ってパターンマッチに失敗した例外 `raise Match` を上げます。
まとめると、以下のようになります。

``` sml
let val tmp1 = (xs, ys) in
(
  case tmp1 of
      (tmp2, tmp3) =>
        (
          case tmp2 of
            _ => case tmp3 of
                     [] => [],
                   | _  => FAIL
        ) || (
          case tmp2 of
              []      => case tmp3 of _ => []
            | :: tmp4 =>
                case tmp4 of
                    (x, xs) =>
                         case tmp3 of
                              :: tmp5 =>
                                  case tmp5 of
                                      (y, ys) => (x, y) :: zip xs ys
                            | _ => FAIL
        )
) || raise Match
```

コンパイル後のコードを見て、変数が複数回使われているのにお気付きでしょうか。特に前半と後半に分けたカタマリの両方で使われている変数がある点に注目です。最初の `case` 式で `tmp2` 、 `tmp3` と分岐を進めたあとに `FAIL` にでくわすと、 `||` に拾われて再度 `tmp2` 、 `tmp3` と分岐を進めていきます。進んだり戻ったりするのがバックトラック法の由来です。


`zip` の例を通して、空則、変数則、コンストラクタ則、タプル則、混合則全ての適用例をみました。アルゴリズムの動作イメージをつかんだので最後に実装していきましょう。

## 実装

頭を動かすパートが終わったので手を動かすパートに移りましょう。パターンマッチ以外の部分は終わっているので、パターンマッチのアルゴリズム部分を見ていきます。

パターンマッチのコンパイラは `CaseToSimple` とは別に実装します。 `BackTrackPatternCompiler` を定義し、そこにパターンマッチのコンパイルを実装していきましょう。まずは構造体の定義からです。

``` rust
struct BackTrackPatternCompiler {
    symbol_generator: SymbolGenerator,
    type_db: case::TypeDb,
}

impl BackTrackPatternCompiler {
    pub fn new(symbol_generator: SymbolGenerator, type_db: case::TypeDb) -> Self {
        Self {
            symbol_generator,
            type_db,
        }
    }

    // ...
}

```

途中で一時変数を使うので `SymbolGenerator` 、 型から得られる情報も使うので `TypeDb` を保持しています。

それでは準備が済んだので、 `compile` メソッドにパターンマッチのコンパイルを実装していきます。`compile` の型はこのようになります。

``` rust
impl BackTrackPatternCompiler {
    // ...

    fn compile(
        &mut self,
        cond: Stack<(Symbol, TypeId)>,
        clauses: Vec<(Stack<case::Pattern>, simple_case::Expr)>,
    ) -> simple_case::Expr {
        // assuming clauses.any(|(patterns, _)| patterns.len() == cond.len())

        // ...
    }

    // ...
}
```

`compile` の引数は概ね条件変数とパターン行列になると説明した通りですね。

1点補足説明をします。この時点で型推論は終わっている前提なので、`cond` には型IDが含まれています。型は代数的データ型のヴァリアントを列挙するのに必要になります。

それでは実装を進めましょう。まずは原理通り、空則、変数則、タプル則、コンストラクタ則、タプル則、混合則で場合分けします。

``` rust
impl BackTrackPatternCompiler {
    // ...

    fn compile(
        &mut self,
        cond: Stack<(Symbol, TypeId)>,
        clauses: Vec<(Stack<case::Pattern>, simple_case::Expr)>,
        default: Option<simple_case::Expr>,
    ) -> simple_case::Expr {
        // assuming clauses.any(|(patterns, _)| patterns.len() == cond.len())
        if cond.is_empty() {
            self.compile_empty(cond, clauses, default)
        } else if clauses
            .iter()
            .all(|(patterns, _)| patterns.last().unwrap().is_variable())
        {
            self.compile_variable(cond, clauses, default)
        } else if clauses
            .iter()
            .all(|(patterns, _)| patterns.last().unwrap().is_tuple())
        {
            self.compile_tuple(cond, clauses, default)
        } else if clauses
            .iter()
            .all(|(patterns, _)| patterns.last().unwrap().is_constructor())
        {
            self.compile_constructor(cond, clauses, default)
        } else {
            self.compile_mixture(cond, clauses, default)
        }
    }

    // ...
}
```

上から順に条件をあてはめます。

* 条件節が空ならパターンの列も空なので空則
* パターン行列の1列目が全て変数パターンなら変数則
* 同じく1列目が全てタプルパターンならタプル則
* 同じく1列目が全てコンストラクタパターンならコンストラクタ則
* それ以外の場合は混合則

というように適用しています。ここでベクトルをスタックとして代用しているので1列目は `last()` で取り出していることに注意して下さい。

それでは個々の則の実装を眺めてみましょう。

### 空則

空則のように簡単なものなら一目瞭然です。

``` sml
impl BackTrackPatternCompiler {
    // ...

    fn compile_empty(
        &mut self,
        _: Stack<(Symbol, TypeId)>,
        mut clauses: Vec<(Stack<case::Pattern>, simple_case::Expr)>,
    ) -> simple_case::Expr {
        clauses.remove(0).1
    }

    // ...
}
```

行が1つなので、その唯一の行の腕部分を取り出します。

### 変数則

次は変数則です。 `compile [c1, c2, ...] [([v11, p12...] => e1), ([v21, p22, ...] => e2), ...]` を `compile [c2, ...] [([p12...] => let val v11 = c1 in e1 end), ([p22, ...] => let val v21 = c1 in e2 end), ...]` に書き換えます。

``` rust
impl BackTrackPatternCompiler {
    // ...

    fn compile_variable(
        &mut self,
        mut cond: Stack<(Symbol, TypeId)>,
        clauses: Vec<(Stack<case::Pattern>, simple_case::Expr)>,
    ) -> simple_case::Expr {
        // 条件変数の先頭と、パターン行列の1列目を処理する
        let (sym, _) = cond.pop().unwrap();
        let clauses = clauses
            // 各行の
            .into_iter()
            .map(|(mut pat, arm)| {
                // 1列目をとりだす
                let var = pat.pop().unwrap().variable();
                // 変数をletにする
                let arm = simple_case::Expr::Let {
                    expr: Box::new(simple_case::Expr::Symbol(sym.clone())),
                    var,
                    body: Box::new(arm),
                };
                (pat, arm)
            })
            .collect();
        // 再帰処理
        self.compile(cond, clauses)
    }

    // ...
}
```

一見長いように見えますがやっていることは説明の通りです。分かりやすいように誤魔化しを入れた記法と比べて実装は泥臭くなりがちです。

ここまでは簡単なルールでした、これ以降のルールは少し複雑です。

### タプル則

タプル則に移りましょう。タプル則では `compile [c1, c2, ...] [([(q11, q12, ..), p12, ...] => e1), ([(q21, q22, ...), p22, ...] => e2), ...]` を
`case c1 of (tmp1, tmp2, ...) => compile [tmp1, tmp2, ..., c2, ...] [([q11, q12, ..., p12, ...] => e1), ([q21, q22, ..., p22, ...] => e2), ...]` にコンパイルするのでした。
見た目は簡単ですが、実装上は煩雑な処理が多いです。
タプルやその型を分解するのは泥臭い作業が必要になりますし、一時変数を生成したりの作業も入ります。


``` rust
impl BackTrackPatternCompiler {
    // ...

    fn compile_tuple(
        &mut self,
        mut cond: Stack<(Symbol, TypeId)>,
        clauses: Vec<(Stack<case::Pattern>, simple_case::Expr)>,
    ) -> simple_case::Expr {
        let (sym, ty) = cond.pop().unwrap();

        // タプル型の分解
        let param_tys: Vec<TypeId> = self.type_db.find(&ty).cloned().unwrap().tuple();
        // 一時変数の生成
        let tmp_vars = self.symbol_generator.gennsyms("v", param_tys.len());
        // タプルを分解したあとの条件部分の変数。
        let mut new_cond = cond;
        // もとの条件変数スタックと平滑化したタプルの変数をあわせる。
        // スタック構造なので逆順に値を入れていることに注意
        new_cond.extend(tmp_vars.iter().cloned().zip(param_tys).rev());

        // 各行の処理
        let clauses = clauses
            // 各行の
            .into_iter()
            .map(|(mut patterns, arm)| {
                // 1列目にあるタプルパターンをとりだす
                let vars = patterns.pop().unwrap().tuple();
                patterns.extend(vars.into_iter().rev());
                (patterns, arm)
            })
            .collect();
        // 全体は `case` 式になる
        simple_case::Expr::Case {
            cond: Box::new(simple_case::Expr::Symbol(sym)),
            clauses: vec![(
                simple_case::Pattern::Tuple(tmp_vars),
                // 再帰処理
                self.compile(new_cond, clauses),
            )],
        }
    }

    // ...
}
```

多少複雑なので順を追って説明しましょう。

<!-- 順序があるのでordered listにしたいけど1文が長い。どうするのが正解？ -->

1. まずマッチ対象の型はタプルであるはずなので、それを分解して取り出します。

このあと、 `case c1 of (tmp1, tmp2, ...) => compile [tmp1, tmp2, ..., c2, ...] [([q11, q12, ..., p12, ...] => e1), ([q21, q22, ..., p22, ...] => e2), ...]` のようにコンパイルされる訳です。条件部分に一時変数があるのが分かるかと思います。これを処理するのが以下です。

2. 上記の処理でタプルの要素数が分かるので、その数だけ一時変数を作ります。
3. 次の `compile` に渡す条件式のスタックを用意します。実際は `cond` と同じですが、わかりやすさのために新しい名前 `new_cond` をつけています。
4. `new_cond` に一時変数をpushします。
5. タプル則の条件から、全ての節の先頭パターンはタプルであるはずなので、それを処理します。
  具体的には `[(q21, q22, ...), p22, ...]` とタプルがあるパターンを `[q11, q12, ..., p12, ...]` のようにタプルを潰したパターンにします。
6. 最後に `case` 式を生成します。これは疑似コードの通りです。

### 混合則

次に混合則です。これはパターンの種類が変わったところで分割するだけなのでそれ自体は際立って説明するところはありません。ところで、ここで $\|$ と $\mathit{FAIL}$ が登場するのでその実装について触れておきましょう。

$\|$ は左辺の式が普通の値ならそのままの値を、 $\mathit{FAIL}$ なら右辺の式の評価をしてその値を返します。$\mathit{FAIL}$ が発生したら即座に $\|$ までジャンプしたいです。このような性質を見たす機能として例外とそのハンドラがあります。なのでここでは $\mathit{FAIL}$ を `raise Fail` に、 $e1 \| e2$ を `e1 handle Fail => e2` にエンコードすることにします。

それを踏まえて混合則の実装は以下になります。

``` rust
impl BackTrackPatternCompiler {
    // ...

    fn compile_mixture(
        &mut self,
        cond: Stack<(Symbol, TypeId)>,
        clauses: Vec<(Stack<case::Pattern>, simple_case::Expr)>,
    ) -> simple_case::Expr {
        // 先頭のパターンと違うものが出現した箇所でパターン行列を分割する
        let head_pattern_type = clauses[0].0.last().unwrap().pattern_type();
        let pos = clauses
            .iter()
            .position(|(pat, _)| pat.last().unwrap().pattern_type() != head_pattern_type)
            .unwrap();
        let (clauses, other) = clauses.split_at(pos);
        // 分割した残りの方をfallbackとし、コンパイルしておく。
        let fallback = self.compile(cond.clone(), other.to_vec());
        // 再帰コンパイル
        let expr = self.compile(cond, clauses.to_vec());

        // e1 handle Fail => e2
        simple_case::Expr::HandleFail {
            expr: Box::new(expr),
            handler: Box::new(fallback),
        }
    }

    // ...
}
```



### コンストラクタ則

最後は最難関のコンストラクタ則です。おおまかな処理の流れは以下になります。

1. 各節のパターンの先頭はコンストラクタパターンなはずなので、コンストラクタパターンとして取り出す
2. コンストラクタパターンから判別子を抽出する
3. 判別子ごとに特殊化したパターン行列を生成する
4. 実際の `case` 式を生成する。各節のパターンは判別子毎、腕は特殊化したパターン行列をコンパイルしたもの。


コメントも含めると少し長いですが、まずは全文を掲載します。

``` rust
impl BackTrackPatternCompiler {
    // ...

    fn compile_constructor(
        &mut self,
        mut cond: Stack<(Symbol, TypeId)>,
        clauses: Vec<(Stack<case::Pattern>, simple_case::Expr)>,
    ) -> simple_case::Expr {
        let (sym, ty) = cond.pop().unwrap();
        // 各節をパターンの先頭とそれ以外に分解。
        // 先頭は全てコンストラクタパターン。
        let clause_with_heads = clauses
            // 各行の
            .into_iter()
            .map(|mut clause| {
                // 1列目にあるコンストラクタパターンをとりだす
                let head = clause.0.pop().unwrap().constructor();
                (head, clause)
            })
            .collect::<Vec<_>>();
        // 先頭のパターンのdescriminantの集合をとる。
        let descriminants = clause_with_heads
            .iter()
            .map(|c| (c.0).0)
            .collect::<HashSet<_>>();
        // descriminantごとに特殊化する
        let mut clauses = descriminants
            .iter()
            .map(|&descriminant| {
                // パターン行列をdescriminantで特殊化したものを取得
                let clauses = self.specialize(descriminant, clause_with_heads.iter());
                // Type DBから今パターンマッチしている型の、
                // 対象にしているdescriminantをもつヴァリアントの、
                // 引数があればその型を取得する
                let param_ty = self.type_db.param_ty_of(&ty, descriminant);
                // 引数があれば一時変数を生成し、条件変数に追加する
                let tmp_var = param_ty.as_ref().map(|_| self.symbol_generator.gensym("v"));
                let mut new_cond = cond.clone();
                new_cond.extend(tmp_var.iter().cloned().zip(param_ty).rev());
                // 返り値はコンストラクタパターンと、それにマッチしたあとの腕の組
                let pat = simple_case::Pattern::Constructor {
                    descriminant,
                    data: tmp_var,
                };
                let arm = self.compile(new_cond, clauses);
                (pat, arm)
            })
            .collect();

        // ヴァリアントが網羅的かどうかで挙動を変える。
        if self.is_exhausitive(&ty, descriminants) {
            // 網羅的ならそのまま `case` 式の生成
            simple_case::Expr::Case {
                cond: Box::new(simple_case::Expr::Symbol(sym.clone())),
                clauses,
            }
        } else {
            // 網羅的でないならパターンマッチの末尾に `_ => raise Fail` を加える
            clauses.push((
                simple_case::Pattern::Variable(self.symbol_generator.gensym("_")),
                simple_case::Expr::RaiseFail,
            ));

            simple_case::Expr::Case {
                cond: Box::new(simple_case::Expr::Symbol(sym.clone())),
                clauses,
            }
        }
    }

    // ...
}
```

ほぼ説明した通りですが、4.の `case` 式の生成のところで分岐が入っています。コンストラクタが網羅的か同化でデフォルト節を挿入するかを分岐しています。網羅的でない場合は $\mathit{FAIL}$ を挟むので、実装の方でも `_ => raise Fail` を追加しています。

さて、ここで使われた `specialize` と `is_exhausitive` も掲載しておきます。

``` rust
impl BackTrackPatternCompiler {
    // ...

    fn specialize<'a, 'b>(
        &'a mut self,
        descriminant: u8,
        clause_with_heads: impl Iterator<
            Item = &'b (
                (u8, Option<case::Pattern>),
                (Stack<case::Pattern>, simple_case::Expr),
            ),
        >,
    ) -> Vec<(Stack<case::Pattern>, simple_case::Expr)> {
        // 先頭のパターンが指定された判別子に合致する節をあつめてくる
        clause_with_heads
            .filter(|(head, _)| head.0 == descriminant)
            .cloned()
            .map(|(head, (mut pat, arm))| {
                pat.extend(head.1.into_iter());
                (pat, arm)
            })
            .collect()
    }

    fn is_exhausitive(
        &self,
        type_id: &TypeId,
        descriminansts: impl IntoIterator<Item = u8>,
    ) -> bool {
        // 判別子の集合がパターンマッチしている型のヴァリアント全ての集合と合致するかで検査
        self.type_db
            .find(&type_id)
            .cloned()
            .unwrap()
            .adt()
            .into_iter()
            .map(|c| c.descriminant)
            .collect::<HashSet<_>>()
            == descriminansts.into_iter().collect::<HashSet<_>>()
    }

    // ...
}
```

これはとりたてて説明することはありません。

---

6つのルールを実装し終えたので最後に、 `BackTrackPatternCompiler` にパターンコンパイラのインタフェース `PatternCompiler` を実装します。

``` rust
impl PatternCompiler for BackTrackPatternCompiler {
    fn compile(
        &mut self,
        cond: Vec<(Symbol, TypeId)>,
        clauses: Vec<(Stack<case::Pattern>, simple_case::Expr)>,
    ) -> simple_case::Expr {
        let expr = self.compile(cond, clauses);
        // 拾いきれなかったFail(=パターンの不足)を拾ってMatch例外にする
        simple_case::Expr::HandleFail {
            expr: Box::new(expr),
            handler: Box::new(simple_case::Expr::RaiseMatch),
        }
    }
}
```

`Fail` は実装内部の例外なので捕捉してユーザ向けに `Match` として投げなおします。これでバックトラック法の実装が完了し、`CaseToSimple` コンパイラへと組込めるようになりました。

ここまでで、バックトラック法の原理と実装を紹介してきました。

