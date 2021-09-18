---
title: パターンマッチのコンパイル：決定木法
---

バックトラック法に続いて決定木法に移ります。 ここで紹介するアルゴリズムは参考文献1（[後半まとめの章](conclusion_of_second_half)参照）に基くものですが、バックトラック方と同じくSMLのパターンマッチに合わせて改変を加えています。

決定木法はバックトラック法と違ってパターンを先に進めることしかしません。そのため、マッチ判定が高速に終わります。また、決定木法には副産物として嬉しい性質があります。全ての可能性を列挙することから自然と網羅性判定もついてくるのです。

一方でマッチ判定の全ての可能性をコードに落としてしまうのでパターンの書き方によっては生成されるコードサイズが爆発的に大きくなってしまいます。コードサイズが増えるということは、生成されるバイナリが大きくなるというだけでなくてコンパイル時間の増大も意味します。

バックトラック法と同じく動作イメージからみていきましょう。

## 動作イメージ

バックトラック法と比較しやすいように同じコード例を使って動作イメージを確認しましょう。

以下のコードを考えます。

``` sml
case ([], [true]) of
    (_,  []) => []
  | ([], _ ) => []
  | (x::xs, y::ys) => (x, y) :: zip xs ys
```


まずはパーターンの先頭行に着目します。先頭行には2要素目にコンストラクタパターンがあります。ここを起点に分岐したいので、パターンマッチ全体で2要素目に注目します。

``` sml
case (..., [true]) of
    (...,  []) => []
  | (...,   _) => []
  | (..., y::ys) => (x, y) :: zip xs ys
```

このうち、条件式の `[true]` にマッチしうるのは2節目と3節だけですのでそれを残して他は選択肢から外します。

``` sml
case (..., [true]) of
  | (...,   _) => []
  | (..., y::ys) => (x, y) :: zip xs ys
```

2要素目にマッチする可能性は全て列挙できたので2要素目のことは忘れます。

``` sml
case ([]) of
  | ([]) => []
  | (x::xs) => (x, y) :: zip xs ys
```

残りは1要素になりました。また1行目に着目します。最初と同じく1コンストラクタパターンがあるので、ここを起点に分岐します。

条件式 `[]` にマッチするのは1節目だけですのでそれを残して他は選択肢から外します。

``` sml
case ([]) of
  | ([]) => []
```

これで1要素目のことは忘れます。

``` sml
case () of
  | () => []
```

これでタプルの最後まできたのでパターンマッチは終了で、一番上にある節の腕、 `[]` が返ります。

動作イメージが掴めたでしょうか。これに相当するコードを吐くコンパイラを作っていきます。


## 原理

決定木法もバックトラック法と同様に、いくつかのルールから成ります。ルール名や一部の補助関数はバックトラック法と同じ名前をしていますが、中身は別物ということに注意して下さい。

### 空則

パターン行列の行数が0の場合、マッチは失敗します。

つまり、以下のようなコンパイルは

``` sml
compile [c1, ...] []
```

$\mathit{FAIL}$ へとコンパイルされます

``` sml
FAIL
```

ここでの $\mathit{FAIL}$ の扱いですが、バックトラック法と違って $\mathit{FAIL}$ が出たら即座にパターンが非網羅的であることが言えます。なのでこの時点でコンパイルエラーにしたり、警告を出したりしても構いません。

### 変数則

もし、先頭の行が全て変数であれば、その行にマッチします。特に、列数が0の場合もこの規則があてはまります。ここでは空則で行数が0の場合を除いたのでここでは行が1つ以上あることを仮定しています。

変数則が決定木法でのコンパイルアルゴリズムの基底になります。つまり、以下のようなコンパイルは

``` sml
compile [c1, ..., cn] [
  ([v1, v2, ..., vn]   => e1),
  ([p21, p22, ..., p2n]   => e2),
  ...
  ([pm1, pm2, ..., pmn]   => em),
]
```

先頭行が採用されるので `E1` になります。
ただし、変数の束縛が必要なのでそこだけ導入しておきます。

``` sml
let
  val v1 = c1
  val v2 = c2
  ...
  val vn = cn
in
  E1
end
```


### 混合則

空則も変数則もあてはまらない場合、先頭行のどこかの列に変数ではないパターンが存在します。混合則ではこの変数ではないパターンを処理します。

まず、先頭行の変数ではないパターンが $i$ 列目にあるとすると、それを1列目と交換します。このとき、他の行のパターンや条件変数も列を交換します。


``` sml
         ↓             ↓
compile [c1, c2, ..., ci, ..., cn] [
  ([v11, v12, ..., p1i, ..., v1n] => E1),
  ([v21, v22, ..., v2i, ..., v2n] => E2),
  ...
]
```

↓

``` sml
         ↓             ↓
compile [ci, c2, ..., c1, ..., cn] [
  ([p1i, v12, ..., v11, ..., v1n] => E1),
  ([v2i, v22, ..., v21, ..., v2n] => E2),
  ...
]
```


これでパターンの先頭列に対して操作を行えます。

参照元の論文ではコンストラクタしか想定していませんが、今回のパターンにはタプルもあるのでコンストラクタかパターンかで分岐します。それぞれの場合をみていきましょう。まずは比較的簡単なタプルの場合から

#### タプルの場合

1行1列がタプルの場合、 `compile` は以下のような見た目をしているはずです。

``` sml
compile [ci, c2, ..., c1, ..., cn] [
  ([(q11, ..., q1m), v12, ..., v11, ..., v1n] => E1),
  ([v2i,           , v22, ..., v21, ..., v2n] => E2),
  ...
]
```

条件変数とパターン行列に分けて考えましょう。

条件変数の方はタプルはバックトラック法のときと同じく `case` を使って分解します。

``` sml
case ci of
  (d1, ..., dn) => compile [d1, ..., dn, ..., c1, ..., cn] [
    ...
  ]
```


条件変数の処理が終わったのでパターン行列を処理します。

パターン行列の1列目のパターンはタプルの形をしているか変数の形をしているかです。それぞれの処理を考えます。タプルの場合はパターンを分解します。変数の場合はタプルを分解したあとの各要素は `_`  で無視し、 `ci` の方を使います。


まとめると、1行1列のパターンをタプル、2行1列のパターンを変数とすると、 `compile` は以下のように変換します。

``` sml
case ci of
  (d1, ..., dn) => compile [d1, ..., dn, ..., c1, ..., cn] [
    ([q11, ..., q1m, ..., v11, ..., v1n] => E1),
    ([_  , ..., _  , ..., v21, ..., v2n] => let val v2i = ci in E2 end),
    ...
  ]
```

#### コンストラクタの場合

1行1列がコンストラクタの場合は以下のような見た目をしているはずです。

``` sml
compile [ci, c2,..., c1, ..., cn] [
  ([C1 p1i, v12, ..., v11, ..., v1n] => E1),
  ([v2i,    v22, ..., v21, ..., v2n] => E2),
  ...
]
```

これを以下の3ステップで変換します。

1. 先頭パターンのコンストラクタの判別子を集めてくる
2. 判別子ごとに特殊化行列を作る
3. コンストラクタが網羅的でなければデフォルト行列を作る

まず、先頭パターンのコンストラクタの判別子を集めてきます。各行の1列目はコンストラクタパターン `Ca pki` か、変数パターンのはずです。このうちコンストラクタパターンの `Ca` の部分を集めてきます。コンストラクタの引数は無視して判別子の部分だけ取り出すのです。ここで取り出した判別子の集合を $\Sigma$ としておきましょう。

次に、判別子ごと、つまり $\Sigma$ の要素ごとの特殊化行列を作ります。特殊化行列を作る関数を `specialize` とします。`specialize` は引数に条件変数、判別子、パターン行列をとってパターン行列を返します。バックラック法でも `specialize` はでてきましたが、それとは異なったものです。


パターン行列の特殊化は以下の疑似コード `specialize` で表現されます。引数があるコンストラクタとないコンストラクタがあるのでその場合分け、コンストラクタパターンと変数パターンがあるのでその場合分けで都合4通りの処理をします。

``` sml
specialize condvar discriminant pattern_matrix = for each row of pattern_matrix
  if pi1 is constructor and
     discriminant matches   => collect it as ([pi2, ...]      => ei)
  if pi1 is constructor arg and
     discriminant matches   => collect it as ([arg, pi2, ...] => ei)
  if pi1 is variable and
     constructor has no arg => collect it as ([pi2, ...]      => let pi1 = condvar in ei end)
  if pi1 is variable and
     constructor has an arg => collect it as ([_, pi2, ...]   => let pi1 = condvar in ei end)
  else ignore
end
```

バックトラック法の `specialize` との違いは、引数に条件変数も増えている点と、変数パターンも含めるか否かです。バックトラック法ではコンストラクタが正確に一致する行を集めるのに対して、決定木法ではコンストラクタにマッチするパターンを1列目に持つ行全てを集めています。

$\Sigma$ に `C1` 〜 `Ca` までのコンストラクタの判別子が集まったとしましょう。このとき、 元の `compile` 関数の引数にあったパターン行列を `pattern_matrix` とすると `compile` は1段階進んで以下のようになります。

``` sml
case ci of
   C1 [arg] => compile [arg, c2, ...] (specialize ci C1 pattern_matrix)
|  C2 [arg] => compile [arg, c2, ...] (specialize ci C2 pattern_matrix)
   ...
```

ここで、 $\Sigma$ に集まったコンストラクタが網羅的でなければ追加の処理があります。どこかの行に先頭列が変数パターンのものがあるはずなので、それらにマッチする処理です。 `_ => ...` の形のデフォルト節を追加します。

追加するためのデフォルト節の腕を構築する `default_patterns` も定義しましょう。疑似コードで以下のような処理をします。

``` sml
default_patterns condvar pattern_matrix = for each row of pattern_matrix
  if pi1 is variable  => collect it as ([pi2, ...] => let pi1 = condvar in ei end)
  else ignore
end
```

これを使って、最終的に `case` は以下のようになります。

``` sml
case ci of
   C1 [arg] => compile [arg, c2, ...] (specialize ci C1 pattern_matrix)
|  C2 [arg] => compile [arg, c2, ...] (specialize ci C2 pattern_matrix)
   ...
|  _        => compile [c2, ...] (default_patterns ci, pattern_matrix)
```

---

少し複雑ですが、これが決定木法の原理です。

決定木法について少し考察してみましょう。もし、パターンが網羅的でなかったら何が起こるでしょう。最後の `default_patterns` が空の行列（行数が0の行列）を返します。これは次のステップで空則にあたるので $\mathit{FAIL}$ になります。逆に、行数が0の行列を作りうるのは `default_patterns` のみです。パターンが網羅的でない場合でしか $\mathit{FAIL}$ はでてきません。このことからも決定木法は網羅性判定を内包することがよく分かるでしょう。


注意深く見ると、決定木法ではそれぞれの条件変数が高々1度しか `case` に使われていないのが分かるかと思います。このおかげで実行が速くなっています。一方で `specialize` や `default_patterns` で変数で始まる列をコピーしています。このため、コンパイルしたあとのコードは大きくなりがちです。

## 具体例

ここでも具体的なコードに上記の原理をあてはめてみましょう。
何度もでてきている `zip` のコード例です。

``` sml
case (xs, ys) of
    (_,  []) => []
  | ([], _ ) => []
  | (x::xs, y::ys) => (x, y) :: zip xs ys
```

このパターン部分をコンパイルするコードだけに注目します。

```sml
compile [(xs, ys)] [
  ([(_,  [])]       => []),
  ([([], _ )]       => []),
  ([(x::xs, y::ys)] => (x, y) :: zip xs ys)
]
```

これを手順を追ってコンパイルしていきましょう。

パターン行列の先頭行のパターン全てが変数ではありません。*混合則* を適用します。タプルパターンがあるのでそれを先頭にもってきて、 *タプルの場合* に則って分解します。

```sml
case (xs, ys) of
  (tmp1, tmp2) => compile [tmp1, tmp2] [
    ([_,  []]       => []),
    ([[], _ ]       => []),
    ([x::xs, y::ys] => (x, y) :: zip xs ys)
]
```

次に出てきたパターン行列も、先頭行のパターンは全てが変数ではありません。 *混合則* を適用します。先頭行のパターンでは、2列目がコンストラクタパターン `[]` を持つます。これを先頭にもってきて処理を進めます。

```sml
case (xs, ys) of
  (tmp1, tmp2) => compile [tmp2, tmp1] [
    ([[],  _]       => []),
    ([_ , []]       => []),
    ([y::ys, x::xs] => (x, y) :: zip xs ys)
]
```

先頭にもってきたあとの1列目はリストへのパターンマッチです。なので *コンストラクタの場合* の処理に進みます。リストのコンストラクタは `[]` と `::` の2つあります。それぞれにマッチする行を `specialize` で集めます。

```sml
case (xs, ys) of
  (tmp1, tmp2) => case tmp2 of
      (* specializeしたものをcompileする *)
      [] => compile [tmp1] (specialize tmp2 [] [
        ([[],  _]       => []),
        ([_ , []]       => []),
        ([y::ys, x::xs] => (x, y) :: zip xs ys)
      ])
      (* specializeしたものをcompileする *)
    | :: tmp3 => compile [tmp3, tmp2] (specialize tmp2 :: [
        ([[],  _]       => []),
        ([_ , []]       => []),
        ([y::ys, x::xs] => (x, y) :: zip xs ys)
    ])
```

1列目が `[]` の条件式にマッチするのは1行目と2行目です。同じく1列目が `::` にマッチするのは2行目と3行目です。つまり、それぞれの `specialize` を処理すると以下のようになります。

```sml
case (xs, ys) of
  (tmp1, tmp2) => case tmp2 of
      (* specializeを解決 *)
      [] => compile [tmp1] [
        ([_]        => let val _ = tmp2 in [] end),
        ([[]]       => let _ = tmp2 in [] end),
      ]
      (* specializeを解決 *)
    | :: tmp3 => compile [tmp3, tmp2] [
        ([_      ,    []] => let val _ = tmp2 in [] end),
        ([(y, ys), x::xs] => (x, y) :: zip xs ys)
    ]
```


`[]` と `::` に分解できました。それぞれの腕にまだ `compile` が残っているのでそれぞれ処理していきます。

まずは `[]` の腕から。先頭行が全て変数パターンなので *変数則* を適用し、この行を採用します。
結果、こうなります。

```sml
case (xs, ys) of
  (tmp1, tmp2) => case tmp2 of
      (* compileを解決 *)
      [] => let val _ = tmp1 in [] end
    | :: tmp3 => compile [tmp3, tmp2] [
        ([_      , []   ] => let _ = tmp2 in [] end),
        ([(y, ys), x::xs] => (x, y) :: zip xs ys)
    ]
```

次に、 `::` の腕です。これは先頭行が全てパターンではありません。そこで *混合則* を適用します。コンストラクタパターン `[]` がある2列目を先頭にもってきます。少し複雑になってきたので `compile` をするコード部分だけ抜き出して処理を進めましょう。

```sml
compile [tmp2, tmp3] [
    ([[]   , _      ] => let _ = tmp2 in [] end),
    ([x::xs, (y, ys)] => (x, y) :: zip xs ys)
]
```

いれかえたあとの1列目はリストへのパターンマッチですので *コンストラクタの場合* に進みます。リストのコンストラクタは `[]` と `::` があります。それぞれのコンストラクタで場合分けして、それぞれの腕に `specialize` を置きます。

```sml
case tmp2 of
    []      => compile [tmp3] (specialize tmp2 [] [
        ([[]   , _      ] => let _ = tmp2 in [] end),
        ([x::xs, (y, ys)] => (x, y) :: zip xs ys)
    ])
  | :: tmp4 => compile [tmp4, tmp3] (specialize tmp2 :: [
        ([[]   , _      ] => let _ = tmp2 in [] end),
        ([x::xs, (y, ys)] => (x, y) :: zip xs ys)
    ])
```

これは網羅的なのでデフォルト節は不要です。

それぞれの `specialize` を処理すると以下のようになります。

```sml
case tmp2 of
    []      => compile [tmp3] [
        ([_] => let _ = tmp2 in [] end),
    ]
  | :: tmp4 => compile [tmp4, tmp3] [
        ([(x, xs), (y, ys)] => (x, y) :: zip xs ys)
    ]
```


上の `compile` は *変数則* を適用してコンパイル終了ですので下の `compile` をみていきます。

``` sml
compile [tmp4, tmp3] [
    ([(x, xs), (y, ys)] => (x, y) :: zip xs ys)
]
```

1行目は全てが変数パターンではありませんので *混合則* を適用します。
そのまま *タプルの場合* に進み、処理します。

``` sml
case tmp4 of
    (tmp5, tmp6) => compile [tmp5, tmp6, tmp3] [
    ([x, xs, (y, ys)] => (x, y) :: zip xs ys)
]
```

まだタプルパターンが残っているので同じく *混合則* の *タプルの場合* に進みます。

``` sml
case tmp4 of
    (tmp5, tmp6) => case tmp3 of
        (tmp7, tmp8) => compile [tmp7, tmp8, tmp5, tmp6] [
            ([y, ys, x, xs] => (x, y) :: zip xs ys)
        ]
```


これは先頭行が全て変数パターンなので変数則を適用して以下のようになります。

``` sml
case tmp4 of
    (tmp5, tmp6) => case tmp3 of
        (tmp7, tmp8) => let
          val y  = tmp7
          val ys = tmp8
          val x  = tmp5
          val xs = tmp6
        in
          (x, y) :: zip xs ys
        end
```

結果、全体としては以下のようになります。

```sml
case (xs, ys) of
  (tmp1, tmp2) =>  case tmp2 of
      [] => let val _ = tmp1 in [] end
    | :: tmp3 => case tmp2 of
          []      => let val _ = tmp3 in
                     let val _ = tmp2 in
                       []
                     end end
        | :: tmp4 => case tmp4 of
              (tmp5, tmp6) => case tmp3 of
                  (tmp7, tmp8) => let
                    val y  = tmp7
                    val ys = tmp8
                    val x  = tmp5
                    val xs = tmp6
                  in
                    (x, y) :: zip xs ys
                  end
```

ルールの適用が長いですし、コンパイルされるコードも大きくなりました。しかし分岐の数は少なくなっています。分岐があるコード（`case` に `|` がついているコード）は2箇所だけです。分岐のネストも最短1、最長2と、とても小さくなっています。

コンパイル後のコードを見て、木になっていることにお気付きでしょうか。これが決定木法の由来です。

`zip` の例を通して、変数則、混合則のタプルの場合、コンストラクタの場合の適用例をみました。今回のパターンは網羅的だったので空則はでてきていません。アルゴリズムの動作イメージをつかんだので最後に実装していきましょう。

## 実装

それでは実装に移りましょう。バックトラック法と同じくパターンマッチのアルゴリズム部分のみを見ていきます。

まずは構造体。

``` rust
struct DecisionTreePatternCompiler {
    type_db: case::TypeDb,
    symbol_generator: SymbolGenerator,
}

impl DecisionTreePatternCompiler {
    pub fn new(symbol_generator: SymbolGenerator, type_db: case::TypeDb) -> Self {
        Self {
            type_db,
            symbol_generator,
        }
    }

    // ...
}
```


バックトラック法と同じですね。`compile` もほぼ同様です。

``` rust
impl DecisionTreePatternCompiler {
    fn compile(
        &mut self,
        cond: Stack<(Symbol, TypeId)>,
        clauses: Vec<(Stack<case::Pattern>, simple_case::Expr)>,
    ) -> simple_case::Expr {
        // assuming clauses.any(|(patterns, _)| patterns.len() == cond.len())
        if clauses.is_empty() {
            self.compile_empty(cond, clauses)
        } else if clauses[0].0.iter().all(|p| p.is_variable()) {
            self.compile_variable(cond, clauses)
        } else {
            self.compile_mixture(cond, clauses)
        }
    }

    // ...
}
```

バックトラック法と構造は同じですが、各メソッドが呼び出される条件が違います。

それでは個別の則の実装をみていきましょう。

### 空則

バックトラック法と同じく、空則と変数則は実装が簡単です。空則の実装は以下になりす。


``` rust
impl DecisionTreePatternCompiler {
    fn compile_empty(
        &mut self,
        _: Stack<(Symbol, TypeId)>,
        _: Vec<(Stack<case::Pattern>, simple_case::Expr)>,
    ) -> simple_case::Expr {
        simple_case::Expr::RaiseMatch
    }

    // ...
}
```

空則は $\mathit{FAIL}$ するのでした。そしてこの $\mathit{FAIL}$ はパターンマッチが非網羅的な場合にのみ出現するのでした。ここではそのままマッチに失敗したときの例外、 `Match` を投げることにしましょう。本物のコンパイラであればコンパイラエラー、あるいは警告を出す手段が用意してあるはずなので、その仕組みを使ってユーザに非網羅的であることを伝えることになります。

### 変数則

空則に続いて変数則もそこまで難しくはありません。

``` rust
impl DecisionTreePatternCompiler {
    fn compile_variable(
        &mut self,
        cond: Stack<(Symbol, TypeId)>,
        mut clauses: Vec<(Stack<case::Pattern>, simple_case::Expr)>,
    ) -> simple_case::Expr {
        // 先頭行をとりだす
        let (patterns, expr) = clauses.remove(0);
        patterns
            // 全ての列に対して
            .into_iter()
            // 変数をしてとりだして
            .map(|p| p.variable())
            // let val p = c in ... end を作る
            .zip(cond.iter().cloned())
            .fold(expr, |acc, (p, (sym, _))| simple_case::Expr::Let {
                expr: Box::new(simple_case::Expr::Symbol(sym)),
                var: p,
                body: Box::new(acc),
            })
    }

    // ...
}
```

変数則も原理をそのまま適用するだけです。変数をそのまま条件変数の別名となるように処理して、腕の式 `expr` を評価します。

### 混合則

次は混合則です。混合則はタプルの場合とコンストラクタの場合でそれぞれ場合分けがありました。タプル、コンストラクタそれぞれ別メソッドに切り出すことにします。すると混合則の実装はこうなります。

``` rust
impl DecisionTreePatternCompiler {
    fn compile_mixture(
        &mut self,
        mut cond: Stack<(Symbol, TypeId)>,
        mut clauses: Vec<(Stack<case::Pattern>, simple_case::Expr)>,
    ) -> simple_case::Expr {
        let pos = self.find_nonvar(&clauses);
        let top = cond.len() - 1;

        // 非変数のパターンをさがして先頭にもってくる
        cond.swap(top, pos);
        for clause in &mut clauses {
            clause.0.swap(top, pos);
        }

        // 先頭にもってきたパターンで場合分け
        if clauses[0].0[top].is_tuple() {
            self.compile_tuple(cond, clauses)
        } else {
            self.compile_constructor(cond, clauses)
        }
    }

    // ...
}
```

変数でないパターンをみつけたらそれを先頭と交換したあと、タプル、またはコンストラクタで分岐します。ここで、 `find_nonvar` の実装はこうなっています。

``` rust
impl DecisionTreePatternCompiler {
    fn find_nonvar(&mut self, clauses: &[(Stack<case::Pattern>, simple_case::Expr)]) -> usize {
        clauses[0].0.iter().rposition(|p| !p.is_variable()).unwrap()
    }

    // ...
}
```

変数でないパターンは1行に複数あることがあります。そのうちどれを選んでもいいのですが[^nonvar]、ここではシンプルに先頭のものを選んでいます。

[^nonvar]: どれを選んでも正しく動作するのですが、コンパイルしたコードのパフォーマンスは変わります。これについては後程軽く触れます。

それではタプルの場合、コンストラクタの場合をそれぞれ見ていきましょう。

#### タプルの場合

少し長くなりますが、タプルの場合は以下のような実装です。

``` rust
impl DecisionTreePatternCompiler {
    fn compile_tuple(
        &mut self,
        mut cond: Stack<(Symbol, TypeId)>,
        clauses: Vec<(Stack<case::Pattern>, simple_case::Expr)>,
    ) -> simple_case::Expr {
        // 条件変数の先頭を取り出す
        let (sym, ty) = cond.pop().unwrap();

        // タプルのそれぞれの型を取得
        let param_tys: Vec<TypeId> = self.type_db.find(&ty).cloned().unwrap().tuple();
        let clauses = clauses
            // それぞれの行の
            .into_iter()
            .map(|(mut patterns, mut arm)| {
                // 先頭のパターンに対して、
                let tuple = match patterns.pop().unwrap() {
                    // タプルパターンならその場に展開し、
                    case::Pattern::Tuple(t) => t,
                    // 変数パターンならタプルの要素数と同じ数の `_` を生成する。
                    case::Pattern::Variable(var) => {
                        let pattern = self
                            .symbol_generator
                            .gennsyms("_", param_tys.len())
                            .into_iter()
                            .map(case::Pattern::Variable)
                            .collect();
                        arm = simple_case::Expr::Let {
                            expr: Box::new(simple_case::Expr::Symbol(sym.clone())),
                            var: var.clone(),
                            body: Box::new(arm),
                        };
                        pattern
                    }
                    _ => unreachable!(),
                };
                patterns.extend(tuple.into_iter().rev());
                (patterns, arm)
            })
            .collect();

        // パターン行列の処理が終わったら条件変数の方にもn個の変数を入れる
        let tmp_vars = self.symbol_generator.gennsyms("v", param_tys.len());
        cond.extend(tmp_vars.iter().cloned().zip(param_tys.clone()).rev());
        // 条件変数に追加したものと同じ変数で `case cond of (v1, v2, ...) => ...` を作る
        simple_case::Expr::Case {
            cond: Box::new(simple_case::Expr::Symbol(sym)),
            clauses: vec![(
                simple_case::Pattern::Tuple(tmp_vars),
                self.compile(cond, clauses),
            )],
        }
    }

    // ...
}
```

まず条件の先頭の変数を取り出したら、そこから型情報が引けるのでタプルの要素などの情報を得ます。

次にパターン行列のそれぞれの行の先頭パターンに対して操作をしていきます。原理のところで説明したとおり、タプルパターンならその場に展開し、変数パターンなら展開された要素は `_` で無視して `let val v2i = ci in e2 end` を生成します。

原理自体も多少複雑ですが実装も行数が嵩みます。これはタプルの要素数が複数ありますが、疑似コードでは `(e1, ..., en)` とごまかしていた部分を実際のコードでは愚直に繰り返し処理を書かないといけないからです。こういった面でも理論と実装のギャップがあります。

#### コンストラクタの場合

次は混合則のコンストラクタの場合です。これは原理のところで説明が長かったことからも分かるように、実装も長くなります。まずはメソッド全体を掲載します。

``` rust
impl DecisionTreePatternCompiler {
    fn compile_constructor(
        &mut self,
        mut cond: Stack<(Symbol, TypeId)>,
        clauses: Vec<(Stack<case::Pattern>, simple_case::Expr)>,
    ) -> simple_case::Expr {
        // 条件変数の先頭を取り出す
        let (sym, ty) = cond.pop().unwrap();

        // 各行の先頭パターンだけ使うことがよくあるので最初に取り出しておく
        let clause_with_heads = clauses
            .into_iter()
            .map(|mut clause| {
                let head = clause.0.pop().unwrap();
                (head, clause)
            })
            .collect::<Vec<_>>();
        // 1. 先頭パターンのコンストラクタを集めてくる
        let discriminants = clause_with_heads
            .iter()
            .filter_map(|(head, _)| match head {
                case::Pattern::Constructor { discriminant, .. } => Some(*discriminant),
                _ => None,
            })
            .collect::<HashSet<_>>();
        // 2. コンストラクタ毎に特殊化行列を作る
        let mut clauses = discriminants
            .iter()
            .map(|&discriminant| {
                let clauses = self.specialized_patterns(
                    sym.clone(),
                    &ty,
                    discriminant,
                    clause_with_heads.iter(),
                );
                // 本来ならコンストラクタごとに、引数の有無で処理が微妙に変わる。
                // そこを `Option` 型のメソッドで違いを吸収している。
                let param_ty = self.type_db.param_ty_of(&ty, discriminant);
                let tmp_var = param_ty.clone().map(|_| self.symbol_generator.gensym("v"));
                let mut new_cond = cond.clone();
                new_cond.extend(tmp_var.iter().cloned().zip(param_ty).rev());
                let pat = simple_case::Pattern::Constructor {
                    discriminant,
                    data: tmp_var,
                };
                let arm = self.compile(new_cond, clauses);
                (pat, arm)
            })
            .collect();

        // 3. コンストラクタが網羅的でなければデフォルト行列を作る
        if self.is_exhausitive(&ty, discriminants) {
            simple_case::Expr::Case {
                cond: Box::new(simple_case::Expr::Symbol(sym.clone())),
                clauses,
            }
        } else {
            let default = self.default_patterns(sym.clone(), clause_with_heads.iter());
            clauses.push((
                simple_case::Pattern::Variable(self.symbol_generator.gensym("_")),
                self.compile(cond, default),
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


コメントだけ拾い読みすれば分かりますが、大枠は最初に実装の都合上の処理をしたあとは、原理どおりに3ステップを処理を進めているだけです。

要点は原理のところで説明してあるので、実装上の注意点を紹介します。

まず、コンストラクタの場合の処理では先頭パターンをよく使います。今回の実装ではスタックをベクタで代用しているので先頭パターンはベクタの末尾にあります。これを取り出すのは少し手間がかかるので最初に取り出してしまっています。

2のコンストラクタ毎に特殊化行列を作るのところは見た目は長いですが、落ち着いてよく見ると原理通りの実装です。 `Self::specialized_patterns` で特殊化行列を作ったあと、必要ならばコンストラクタの引数を受け取る変数パターンを用意してパターン行列の行を作っています。

3のコンストラクタが網羅的でなければデフォルト行列を作るの箇所では `is_exhausitive` を使って場合分けしています。網羅的な場合は素直な実装を、網羅的でない場合は原理通りにデフォルト行列を作っています。

それでは `compile_constructor` で使われた3つのメソッド、 `specialized_patterns` 、 `default_patterns` 、 `is_exhausitive` を掲載します。

まずは `specialized_patterns` です。

``` rust
impl DecisionTreePatternCompiler {
    fn specialized_patterns<'a, 'b>(
        &'a mut self,
        cond: Symbol,
        type_id: &TypeId,
        discriminant: u8,
        clause_with_heads: impl Iterator<
            Item = &'b (case::Pattern, (Stack<case::Pattern>, simple_case::Expr)),
        >,
    ) -> Vec<(Stack<case::Pattern>, simple_case::Expr)> {
        let param_ty = self.type_db.param_ty_of(&type_id, discriminant);
        clause_with_heads
            .filter_map(|(head, clause)| match head {
                // 判別子が一致するコンストラクタパターンはそのままあつめる
                case::Pattern::Constructor {
                    discriminant: d,
                    pattern,
                } if *d == discriminant => Some((pattern.clone().map(|p| *p), clause.clone())),
                // 変数パターンは無条件で集めるが、
                // 腕やパターンに小細工が必要
                case::Pattern::Variable(var) => {
                    let extra_pattern = if param_ty.is_some() {
                        let var = self.symbol_generator.gensym("_");
                        Some(case::Pattern::Variable(var))
                    } else {
                        None
                    };

                    // pattern => arm を pattern => let val var = c in armに変換
                    let (pat, arm) = clause.clone();
                    let arm = simple_case::Expr::Let {
                        expr: Box::new(simple_case::Expr::Symbol(cond.clone())),
                        var: var.clone(),
                        body: Box::new(arm),
                    };
                    Some((extra_pattern, (pat, arm)))
                }
                // discriminantが一致しないコンストラクタは無視
                _ => None,
            })
            .map(|(extra_pattern, (mut pat, arm))| {
                if let Some(p) = extra_pattern {
                    pat.push(p)
                }
                (pat, arm)
            })
            .collect()
    }

    // ...
}
```

`specialized_pattern` はパターン行列から指定されたコンストラクタにマッチする行全てを集めてくる処理でした。コードでも、（先頭列を取り出した）パターン行列の各行に対して以下の処理をしています

1. 先頭列のパターンがコンストラクタでかつ判別子が一致するならそれを集める
2. 先頭列のパターンが変数パターンならば適切に加工した上でそれを集める

ここで、1. の処理でパターンガードを使っています。

``` rust
match head {
  Constructor { ... } if *d == discriminant => { ... },
  Variable( ...)                            => { ... },
  _                                         => { ... }
}
```

このようにパターンガードは条件に合致した場合にのみマッチしたい処理に有用です。

それでは `compile_constructor` で使われていたメソッドの2つめ、 `default_patterns` に移ります。

``` rust
impl DecisionTreePatternCompiler {
    fn default_patterns<'a, 'b>(
        &'a mut self,
        sym: Symbol,
        clause_with_heads: impl Iterator<
            Item = &'b (case::Pattern, (Stack<case::Pattern>, simple_case::Expr)),
        >,
    ) -> Vec<(Stack<case::Pattern>, simple_case::Expr)> {
        clause_with_heads
            .filter(|(head, _)| head.is_variable())
            .cloned()
            .map(|(p, (pat, arm))| {
                let var = p.variable();
                let arm = simple_case::Expr::Let {
                    expr: Box::new(simple_case::Expr::Symbol(sym.clone())),
                    var: var.clone(),
                    body: Box::new(arm),
                };
                (pat, arm)
            })
            .collect()
    }

    // ...
}
```

`default_patterns` は先頭列が変数パターンな行を集めてくるメソッドなのでした。実際、 `.filter(|(head, _)| head.is_variable())` を使ってそのように実装されています。

最後に `compile_constructor` で使われていたメソッドの3つめ、 `is_exhausitive` に移りましょう。

``` rust
impl DecisionTreePatternCompiler {
    fn is_exhausitive(
        &self,
        type_id: &TypeId,
        discriminansts: impl IntoIterator<Item = u8>,
    ) -> bool {
        self.type_db
            .find(&type_id)
            .cloned()
            .unwrap()
            .adt()
            .into_iter()
            .map(|c| c.discriminant)
            .collect::<HashSet<_>>()
            == discriminansts.into_iter().collect::<HashSet<_>>()
    }

    // ...
}
```

パターン内で使われている `discriminant` の集合が、データ型に定義された全てのコンストラクタの判別子の集合と一致しているかで判定しています。Rustでは集合を実装した型 `HashSet` を `==` で比較すると集合同士の比較の意味になります。

---

以上でパターンマッチのコンパイルアルゴリズム2種類を含めたコンパイラが完成しました。いかがだったでしょうか、原理と比べて思ったより分量が多いと感じたのではないでしょうか。枝葉末節を省いた原理は理解しやすくはあるのですが、実際のコードに落としたときとのギャップが大きくなりがちです。そのギャップを埋めた本記事は読者の皆様に有用であると信じています。
