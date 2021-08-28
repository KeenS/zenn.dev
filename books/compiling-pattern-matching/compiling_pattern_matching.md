---
title: パターンマッチのコンパイル
---

ここまでで、前提となる知識や条件、そしてどんなことをするのかを確認してきました。ここから本題となるパターンマッチのコパイルアルゴリズムを見ていきます。

まずは肩慣らしも兼ねてシンプルなパターンからC風言語へのコンパイルアルゴリズムをみます。
これは先程確認したように、ほぼ直接的な変換ですので難しいところはないでしょう。
アルゴリズムというよりはむしろ、データ型やコンパイラのコードの雰囲気を掴むために見ていきます。

次に複雑なパターンからシンプルなパターンマッチのネストへのコンパイルを見ていきます。
これはアルゴリズムが2通りあるのでそれぞれ見ます。

片方のアルゴリズムは本記事では「バックトラック法」と呼ぶことにします。
これは「`switch` が使えるところでは使う」といった趣で、愚直な `if` 文のネストを高速化したような結果になります。
`if` 文のネストの場合と同じくパターンの並べ方と対象のデータに応じてパターンマッチにかかる時間が変わりますが、生成されるコード量は `if` のネストの場合と同じく書いたパターンに比例した分にしかなりません。

もう片方のアルゴリズムは本記事では「決定木法」と呼ぶことにします。
全ての場合を尽すように `switch` をネストさせるもので、パターンの並びに関わらずデータ型の複雑さに比例した時間だけでパターンマッチを行えます。
これは前回の記事で3行のコードが膨大な量のコードになる例で確認したとおり、元のコードよりも爆発的にコード量が増える可能性があります。

これら2つのアルゴリズムは時間と空間のトレードオフがあるのでそれを理解した上でどちらかを選択することになるでしょう。

さて、今まで出てきたコードと状況を整理しましょう。
(TODO)節で `case` 言語とそのインタプリタを作成しました。
我々の設計では `case` 言語からシンプルなパターンを持つ言語に変換し、さらにそこからC風の言語に変換するのでした。
それぞれを `simple_case` 言語と `switch` 言語と呼ぶことにすると、構成は以下のようになります。


作図？
```
case言語 ⟲ インタプリタ
 ↓ CaseToSimpleコンパイラ
 ↓    * バックトラック法
 ↓    * 決定木法
simple_case言語
 ↓ SimpleToSwitchコンパイラ
switch言語
```

このうち、`case` 言語とそとインタプリタのみ作成済みです。
これから、 `simple_case` 言語、 `switch` 言語、そして `CaseToSwitch` コンパイラと `SimpleToSwitch` コンパイラを実装していきます。
`CaseToSwitch` コンパイラは先程説明した通り、パターンマッチのコンパイルのアルゴリズムが複数あるのでそれらを切り替えられるようにします。

#### シンプルなバターンからC風言語へのコンパイル

本項では `simple_case` 言語、 `switch` 言語、 `SimpleToCase` コンパイラを実装します。

##### `simple_case` 言語と `swich` 言語

`simple_case` 言語は `case` 言語とさほど変わりません。大きく違うのはパターンの表現力が異なる点です。
`case` 言語ではパターンのネストや非網羅的パターン、冗長なパターンなどが許されていましたが、
`simple_case` 言語ではそれらが許されません。パターンはネストせず、必ず網羅的で、非冗長的です。
その他に `case` 言語になかった変数束縛があります。これはパターンのコンパイルの際にあると便利だからです。

また、 `simple_case` 言語にはパターンマッチに失敗したときの例外 `Match` を上げる機能が増えています。
そしてパターンマッチ内部で使う例外 `Fail` を上げる機能と、それをハンドルする機能もあります。
これについてはバックトラック法の実装のところで触れることにします。

###### `simple_case` 言語

`simple_case` 言語では以下のような式が書けます。


``` sml
(* データ型の値を構築する *)
inj<1>()

(* タプル *)
(inj<1>(), inj<1>())

(* 変数束縛とパターンマッチ *)
let val tuple = (inj<1>(), inj<1>())
in case tuple of
     (* タプルパターン *)
     (x, y) =>
       (* 変数xを参照できる *)
       case x of
       (* コンストラクタパターン*)
         c<1>() => ()
       | no_match => raise Fail (* Fail例外 *)
     (* 変数パターン *)
   | other            => ()
```

`case` 言語で使えたデータ型の値の構築、変数束縛とパターンマッチが使えます。
各種パターンはネストができず、必ず網羅的でないとならない制約がついています。
そして `Fail` 例外を挙げる機能などの新規機能があります。

この言語をデータ型にすると以下のようになります。

``` rust
mod simple_case {
    use super::*;

    /// 式
    #[derive(Debug, Clone)]
    pub enum Expr {
        /// (expr1, expr2, ...)
        Tuple(Vec<Expr>),
        /// let val var = expr in body end
        Let {
            var: Symbol,
            expr: Box<Expr>,
            body: Box<Expr>,
        },
        /// inj<descriminant>(data)
        Inject {
            descriminant: u8,
            data: Option<Box<Expr>>,
        },
        /// case cond of pattern1 => expr1 | pattern2 => expr2 ...
        Case {
            cond: Box<Expr>,
            clauses: Vec<(Pattern, Expr)>,
        },
        /// raise Match;
        RaiseMatch,
        /// raise Fail;
        RaiseFail,
        /// expr handle Fail => handler
        HandleFail { expr: Box<Expr>, handler: Box<Expr> },
        /// x
        Symbol(Symbol),
    }

    /// パターン
    #[derive(Debug, Clone)]
    pub enum Pattern {
        /// (pat1, pat2, ...)
        Tuple(Vec<Symbol>),
        /// c<descriminant>(data)
        Constructor {
            descriminant: u8,
            data: Option<Symbol>,
        },
        /// x
        Variable(Symbol),
    }
}

```

`Expr` のそれぞれのヴァリアントがそれぞれの構文に対応しています。
`RaiseMatch` 、 `RaiseFail` 、`HandleFail` のように、いくつか構文とその引数が融合したヴァリアントがあります。
本来は構文とその引数で別々に定義するものですが、例外をちゃんと実装しようとすると不必要に複雑になるのでこのようにしています。

`simple_case` 言語はパターンの部分は `case` 言語より簡単になっていますが言語全体としては機能が増えています。
`let` 式や `raise Fail` 式など `case` 言語にはなかった構文がありますね。
難しい機能を減らす代わりに簡単な機能を加えて低級な言語に近付けるのはコンパイラではよくあることです。

###### `swicch` 言語

`switch` 言語はC風の言語です。大きな特徴はC言語のような `switch` 文があることです。
また、メモリ操作など手続的な操作もあります。
代わりに `simple_case` 言語にあった `Inject` がなくなり、メモリの確保やデータの保存を逐一行なうようになります。

``` C
/* 変数 */
x := 1;

/* メモリ確保 */
tuple = alloc(2);

/* メモリ書き込み */
store(tuple+0, 1);
store(tuple+1, 1);

/* メモリ読み出し */
x := load(tuple+0);
y := load(tuple+1);

/* switch文 */
switch(x) {
  case 0: {
    /* goto文 */
    goto NEXT;
  }
  case 1: {
    result := 3;
  }
  /* default節 */
  default: {
    /* UNREACHABLE文 */
    UNREACHABLE;
  }
};
y := result;
/* ラベル */
NEXT:
  /* return文 */
  return result;

/* 例外 */
sml_raise(SML_EXN_MATCH);
```

変数定義、メモリ確保にメモリ操作、 `switch` 文や `goto` 文とラベルなどがあります。

`simple_case` 言語と同様に `switch` 言語をデータ型で表わすと以下のようになります。

``` rust
mod switch {
    use super::*;

    /// ブロック（文のかたまり）
    pub struct Block(pub Vec<Stmt>);

    /// 文
    pub enum Stmt {
        /// x := op;
        Assign(Symbol, Op),
        /// store(base+offset, data);
        Store {
            base: Symbol,
            offset: u8,
            data: Symbol,
        },
        /// switch(cond) {
        ///   target: {
        ///     block
        ///   }
        ///   ...
        ///   defalut: {
        ///     default
        ///   }
        /// };
        Switch {
            cond: Symbol,
            targets: Vec<(i32, Block)>,
            default: Block,
        },
        /// label:
        Label(Symbol),
        /// goto label;
        Goto(Symbol),
        /// UNREACHABLE;
        Unreachable,
        /// return x;
        Ret(Symbol),
    }

    /// 操作。x := の右側に書けるもの。
    pub enum Op {
        /// alloc(size);
        Alloc { size: u8 },
        /// load(base+offset);
        Load { base: Symbol, offset: u8 },
        /// x
        Symbol(Symbol),
        /// 1
        Const(i32),
    }
}
```

`switch` 言語へは多少説明が必要でしょうか。

Case言語は全体が1つの式でしたが、Switch言語は複数の文の列になります。
なのでトップレベルを表わすのは `Block(pub Vec<Stmt>)` です。
トップレベル以外でも `Switch` の腕や、 `goto` のラベルの後なども文の列がくるので `Block` を保持しています。

1行を表わすのが `Stmt` で、最初に紹介した構文が定義されています。
`Switch` は必ずデフォルト節を取るようになっていて、もしデフォルト節を使わないなら `Unreachable` を使ってそこに到達しないことを表現することにします。

さて、ソースとターゲットの言語が出揃ったのでこれらの間の変換を書きます。
おおまかに、以下のような `simple_case` 言語 → `switch` 言語の対応付けをします

編：テーブルとかの方がいい？

* `let` （`Let`）→ 代入（`Assign`）
* コンストラクタの適用（`Inject`） → メモリ確保（`Alloc`）とそこへの値の書き込み（`Store`）
* `case`（`Case`）→ `switch` （`Switch`）とメモリからの読み出し（`Load`）
* シンボル → シンボル

`switch` 言語の代入の右側に表われる式（右辺値）はメモリ確保命令、ロード、シンボル、定数と限られた式しか使えません。
なので `simple_case` 言語のように式がいくらでも大きくなることはなく、1つ1つのシンプルな式が並べられます。すると自然と生成されるコードは縦に長くなります。


##### `case` 言語への機能追加

コンパイラを書く前に、 `case` 言語に機能を追加しておきます。
といっても構文を拡張する訳ではありません。
インタプリタを書く際に必要ない機能を飛ばしてしまったのでそれを補完します。

###### 型DB

`case` 言語には代数的データ型の定義を表わす式がありません。
例えば `datatype bool = True | False` に相当する「`bool` 型にはコンストラクタ `True` と `False` がある」という情報を表わせません。

データ型の情報を登録するために **型DB** を定義します。
この型DBはまさしく「`bool` 型にはコンストラクタ `True` と `False` がある」のような情報を登録します。
もうちょっというとコンストラクタは引数をとることとがあるので、その引数の情報も必要です。

それでは `case` 言語に型DBを追加しましょう。
モージュール定義の下部に追記する形で実装します。
必要なのはコンストラクタ、型、型DBです。

``` rust
mod case {
    // ...

    #[derive(Debug, Clone)]
    pub struct Constructor {
        pub descriminant: u8,
        pub param: Option<TypeId>,
    }

    #[derive(Debug, Clone)]
    pub enum Type {
        Tuple(Vec<TypeId>),
        Adt(Vec<Constructor>),
    }

    #[derive(Debug, Clone)]
    pub struct TypeDb(HashMap<TypeId, Type>);

}
```


型DBにはコンストラクタ、登録、参照の機能もつけておきましょう。

``` rust
mod case {
    // ...

    impl TypeDb {
        /// コンストラクタ
        pub fn new() -> Self {
            Self(HashMap::new())
        }

        /// タプル型の登録
        pub fn register_tuple(&mut self, type_id: TypeId, tys: impl IntoIterator<Item = TypeId>) {
            self.0
                .insert(type_id, Type::Tuple(tys.into_iter().collect()));
        }

        /// 代数的データ型の登録
        pub fn register_adt(
            &mut self,
            type_id: TypeId,
            constructors: impl IntoIterator<Item = Constructor>,
        ) {
            self.0
                .insert(type_id, Type::Adt(constructors.into_iter().collect()));
        }

        /// 参照
        pub fn find(&self, type_id: &TypeId) -> Option<&Type> {
            self.0.get(type_id)
        }
}
```

`register_tuple` について補足します。
タプル型は本来 `(bool, bool)` や `(hoge, fuga, piyo)` のように要素数、個々の要素ともに柔軟な組み込み型です。
一方でコンパイラ内部の都合でそれぞれの型1つ1つにIDがほしいです。
なので `(bool, bool)` や `(hoge, fuga, piyo)` などそれぞれのユースケースごとに別の型とみなしてIDを割り振ってしまいます。そのための機能が `register_tuple` です。

型DBに登録するときは以下のように地道に1つ1つデータ型の表現を登録していきます。


``` rust
let mut type_db = case::TypeDb::new();
// datatype bool = True | False
type_db.register_adt(
    TypeId::new("bool"),
    vec![
        // True
        case::Constructor {
            descriminant: 0,
            param: None,
        },
        // False
        case::Constructor {
            descriminant: 1,
            param: None,
        },
    ],
);

// (bool, bool)
type_db.register_tuple(
    TypeId::new("tuple2"),
    vec![TypeId::new("bool"), TypeId::new("bool")],
);
```

###### ユーティリティ

型DBで `case` 言語は完成しましたが、コンパイラを書くために便利メソッドをいくつか追加しておきます。

まずはパターンが3種類あるのでそのどれであるかを返すメソッドを定義します。

``` rust
mod case {
    // ...

    #[derive(Debug, Clone, PartialEq, Eq)]
    pub enum PatternType {
        Tuple,
        Constructor,
        Variable,
    }

    impl Pattern {
        pub fn pattern_type(&self) -> PatternType {
            use PatternType::*;
            match self {
                Pattern::Tuple(_) => Tuple,
                Pattern::Constructor { .. } => Constructor,
                Pattern::Variable(_) => Variable,
            }
        }
    }
}
```

`PatternType` は `Pattern` からコンストラクタの引数を抜いたものです。
ほぼ同じものなので改めて定義するのはちょっと歯痒いですが丁寧に処理していきます[^derive-kind-crate]。

[^derive-kind-crate]: Rustのパッケージレジストリcrates.ioにはこれを自動でやってくれるライブラリもあります。しかし今回はCargoの説明に紙面を割けないので手で定義しています。

次にパターンに名指しでタプルであるかなどを返す `is_xxx` と、パターンがタプルであると決め付けてその引数を取り出す `xxx` メソッド群を定義します。

``` rust
mod case {
    // ...

    impl Pattern {
        // ...

        pub fn is_tuple(&self) -> bool {
            use Pattern::*;
            match self {
                Tuple(_) => true,
                _ => false,
            }
        }

        pub fn tuple(self) -> Vec<Pattern> {
            use Pattern::*;
            match self {
                Tuple(patterns) => patterns,
                _ => panic!("pattern is not a tuple"),
            }
        }

        pub fn is_constructor(&self) -> bool {
            use Pattern::*;
            match self {
                Constructor { .. } => true,
                _ => false,
            }
        }

        pub fn constructor(self) -> (u8, Option<Pattern>) {
            match self {
                Pattern::Constructor {
                    descriminant,
                    pattern,
                } => (descriminant, pattern.map(|p| *p)),
                _ => panic!("pattern is not a constructor"),
            }
        }
        pub fn is_variable(&self) -> bool {
            use Pattern::*;
            match self {
                Variable { .. } => true,
                _ => false,
            }
        }

        pub fn variable(self) -> Symbol {
            match self {
                Pattern::Variable(s) => s,
                _ => panic!("pattern is not a variable"),
            }
        }
    }
}
```

`Type` 型にも同様のメソッドを定義しましょう。

``` rust
mod case {
    // ...

    impl Type {
        pub fn tuple(self) -> Vec<TypeId> {
            use Type::*;
            match self {
                Tuple(ts) => ts,
                _ => panic!("type is not a tuple"),
            }
        }
        pub fn adt(self) -> Vec<Constructor> {
            use Type::*;
            match self {
                Adt(cs) => cs,
                _ => panic!("type is not an ADT"),
            }
        }
    }
}
```

これで下準備が整いました。

##### SimpleToSwitchコンパイラ

コンパイラを書きはじめましょう。
まずはユーティリティとしてシンボル生成器を定義しておきます。

``` rust
use std::rc::Rc;
use std::cell::Cell;

#[derive(Debug, Clone)]
pub struct SymbolGenerator(Rc<Cell<u32>>);
impl SymbolGenerator {
    pub fn new() -> Self {
        Self(Rc::new(Cell::new(0)))
    }

    /// 名前のヒントを受け取って既存のシンボルと衝突しないシンボルを生成する
    pub fn gensym(&mut self, hint: impl Into<String>) -> Symbol {
        let id = self.0.get();
        self.0.set(id + 1);
        Symbol(hint.into(), id)
    }

    /// n個のシンボルを生成する
    pub fn gennsyms(&mut self, hint: impl Into<String>, n: usize) -> Vec<Symbol> {
        let hint = hint.into();
        std::iter::repeat_with(|| self.gensym(hint.clone()))
            .take(n)
            .collect::<Vec<_>>()
    }
}
```

`simple_case` 言語から `switch` 言語に変換するにあたって一時変数を使うことがあります。
ところで名前解決のところ（TODO: 章番号参照する）で説明した通り、 `case` 言語以下の中間言語では名前が衝突しないことになっていました。
すると一時変数もスコープにある名前と衝突しないものを用意しないといけません。
そのためにシンボルジェネレータを使ってどの変数とも衝突しないシンボルを作ります。それが `gensym` です。
このシンボルジェネレータはコンパイラを通して共通の通し番号を使う設計で、 `Clone` しても番号が共有されるように `Rc<Cell<>>` を使っています。

さて、コンパイラ自身のデータ型は以下のように定義します。

``` rust
struct SimpleToSwitch {
    symbol_generator: SymbolGenerator,
    local_handler_labels: Vec<Symbol>,
}

impl SimpleToSwitch {
    pub fn new(symbol_generator: SymbolGenerator) -> Self {
        Self {
            symbol_generator,
            local_handler_labels: Vec::new(),
        }
    }
}
```

シンボルジェネレータとハンドラのラベルを保持します。ハンドラのラベルについては `RaiseFail` と `HandleFail` のところで解説します。

コンパイル処理を書き始めます。エントリーポイントは `compile` です。
内部的にはほぼ `compile_expr` を呼ぶだけです。
そして `compile_expr` は構文でパターンマッチしてそれぞれの構文ごとの処理に振り分けるだけです。


``` rust
impl SimpleToSwitch {
    // ...

    pub fn compile(&mut self, case: simple_case::Expr) -> switch::Block {
        let (mut stmts, ret) = self.compile_expr(case);
        stmts.push(switch::Stmt::Ret(ret));
        switch::Block(stmts)
    }

    fn compile_expr(&mut self, case: simple_case::Expr) -> (Vec<switch::Stmt>, Symbol) {
        match case {
            simple_case::Expr::Tuple(t) => self.compile_tuple(t),
            simple_case::Expr::Let { var, expr, body } => self.compile_let(var, *expr, *body),
            simple_case::Expr::Inject { descriminant, data } => {
                self.compile_inject(descriminant, data.map(|d| *d))
            }
            simple_case::Expr::Case { cond, clauses } => self.compile_case(*cond, clauses),
            simple_case::Expr::RaiseMatch => self.compile_raise_match(),
            simple_case::Expr::RaiseFail => self.compile_raise_fail(),
            simple_case::Expr::HandleFail { expr, handler } => {
                self.compile_handle_fail(*expr, *handler)
            }
            simple_case::Expr::Symbol(s) => self.compile_symbol(s),
        }
    }

}
```

`simple_case` 言語では式指向だったので式を書くと暗黙にその値が返りましたが `Switch` 言語では文指向なので明示的な `return` が必要です。
かといって全ての式に `return` をつけると `1 + 1` が `return (return 1) + (return 1);` になってしまいます。
どうにかして計算の部分と値を返す部分を区別できるようにしないといけません。
そこでコンパイル処理の返り値を「式の計算文」と「計算結果を格納したシンボル名」の2つにし、値を受け取る側でどうするのかを決めることにします。
`compile_expr` の返り値型は `(Vec<switch::Stmt>, Symbol)` になてちますが、 `Vec<...>` の部分が式の計算文、 `Symobl` が計算結果を格納したシンボル名です。
`compile` では最後には `return` したいので最終的に受け取ったシンボルを `Ret` にしてブロックの最後に加えています。

`compile_expr` では構文ごとのメソッドに処理を振りわけています。
それぞれの構文ごとのコンパイル処理を見てみましょう。
簡単なものからとりかかります。

###### `Symbol` のコンパイル

まずは `Symbol` です。

``` rust
impl SimpleToSwitch {
    // ...
    fn compile_symbol(&mut self, s: Symbol) -> (Vec<switch::Stmt>, Symbol) {
        (vec![], s)
    }
}
```

シンボルはSwitch言語でもそのままシンボルとして扱えるのでそのままシンボルを返すだけです。
「式の計算文」に相当するものは不要なので空のベクトルを返しています。

###### `Let` のコンパイル

`simple_case` 言語での `let val var = expr in body end` のコンパイルです。
おおざっぱには以下のような見た目になります。

``` c
/* exprの部分 */
...
sym := ...;
/* bodyの部分 */
var := sym;
....
ret := ...;
```

これを実装したのが以下です。

``` rust
impl SimpleToSwitch {
    // ...
    fn compile_let(
        &mut self,
        var: Symbol,
        expr: simple_case::Expr,
        body: simple_case::Expr,
    ) -> (Vec<switch::Stmt>, Symbol) {
        use switch::{Op, Stmt};
        let (mut stmts, sym) = self.compile_expr(expr);
        stmts.push(Stmt::Assign(var, Op::Symbol(sym)));
        let (stmts2, ret) = self.compile_expr(body);
        stmts.extend(stmts2);
        (stmts, ret)
    }
}
```

`expr` と `body` を順にコンパイルし、それぞれの文を1つにまとめます。
`expr` の返りシンボルは `let` で束縛していたシンボルに代入（エイリアス）します。

###### `Inject` のコンパイル

つぎは `Inject` 、 `simple_case` 言語での `inject(desc, data)` のコンパイルです。  `Inject` のコンパイルに入る前にシンボルジェネレータの `gensym` を短く呼べる手を用意しておきましょう。

``` rust
impl SimpleToSwitch {
    // ...
    fn gensym(&mut self, hint: impl Into<String>) -> Symbol {
        self.symbol_generator.gensym(hint)
    }
}
```

`.symbol_generator` の部分を省けるようになるだけですが、以後何度か呼ぶので短かく呼べるようにする価値がありす。

さて `inject(desc, data)` はおおまかには以下のようにコンパイルされます。

``` c
v := alloc(2);
store(v+0, <desc>);
store(v+1, <data>);
```

ヒープ領域は `data` があればその分と判別子のサイズ分使います。
今回のデータ表現では値は全てヒープに確保することにして、それらを扱うときはポインタを経由することにします[^data-expression]。
すると引数をとるコンストラクタでは `data` がどんなサイズの値になっても `inject` ではちょうど2ワード分使います。

[^data-expression]: 現実の処理系ではポインタが多いと遅くなるのでポインタが減る工夫がされています。しかしここではシンプルさを重視してひとまずポインタを噛ませる方式にしました。

ただし `data` はさらにコンパイルされる点、`store` の右辺にシンボルしか許してない点から、もう少し詳しく書くと以下のようになります。

``` c
/* descriminantの部分 */
desc := <desc>;

/* dataの部分 */
...
sym := ...;
data := ...;

/* injectの構成 */
v := alloc(2);
store(v+0, desc);
store(v+1, data);
```

先に `desc` 、 `data` を計算してからそれぞれの値をメモリに書き込みます。

以上のことに加え、 `data` はオプショナルなことに注意しつつコードにするとこうなります。

``` rust
impl SimpleToSwitch {
    // ...
    fn compile_inject(
        &mut self,
        descriminant: u8,
        data: Vec<simple_case::Expr>,
    ) -> (Vec<switch::Stmt>, Symbol) {
        use switch::{Op, Stmt};
        let mut ret = Vec::new();
        let size = (data.is_some() as u8) + 1;

        // descriminantの部分
        let des = self.gensym("des");
        ret.push(Stmt::Assign(des.clone(), Op::Const(descriminant as i32)));

        // dataの部分
        let mut data_sym = None;
        if let Some(d) = data {
            let (stmts, d) = self.compile_expr(d);
            ret.extend(stmts);
            data_sym = Some(d)
        }

        // injectの構成
        let v = self.gensym("v");
        ret.push(Stmt::Assign(v.clone(), Op::Alloc { size }));
        ret.push(Stmt::Store {
            base: v.clone(),
            offset: 0,
            data: des.clone(),
        });
        if let Some(sym) = data_sym {
            ret.push(Stmt::Store {
                base: v.clone(),
                offset: 1,
                data: sym,
            })
        }
        (ret, v.clone())
    }
}
```


少し長くなりますが、複雑なことをしている訳ではなくて作業量が多いだけです。

###### `Tuple` のコンパイル

`simple_case` 言語での `(e0, ..., en)` のコンパイルです。
タプルは要素を1つ1つコンパイルしたあと、メモリを確保してそこに書き込むだけです。

``` c
/* 0つめのデータ */
...
e0 := ...
/* 1つめのデータ */
...
e1 := ...

...

/* タプルの構成 */
tuple := alloc(n);
store(tuple+0, e0);
store(tuple+1, e1);
...
```

これを実装するとこうなります。

``` rust
impl SimpleToSwitch {
    // ...
    fn compile_tuple(&mut self, es: Vec<simple_case::Expr>) -> (Vec<switch::Stmt>, Symbol) {
        use switch::{Op, Stmt};
        let mut stmts = vec![];
        let mut symbols = vec![];

        // 各データのコンパイル
        for e in es {
            let (s, sym) = self.compile_expr(e);
            stmts.extend(s);
            symbols.push(sym);
        }

        // タプルの構成
        let t = self.gensym("tuple");
        stmts.push(Stmt::Assign(
            t.clone(),
            Op::Alloc {
                size: symbols.len() as u8,
            },
        ));
        for (i, s) in symbols.into_iter().enumerate() {
            stmts.push(Stmt::Store {
                base: t.clone(),
                offset: i as u8,
                data: s,
            })
        }
        (stmts, t)
    }
}
```

比較的素直な実装ですね。

###### `RaiseMatch` のコンパイル

`simple_case` 言語での `raise Match` のコンパイルです。
`RaiseMatch` は例外の送出なので本来は複雑な処理です。
しかしここでは例外を上げる関数 `sml_raise` を呼ぶことにしてお茶を濁します。

``` c
sml_raise(SML_EXN_MATCH);
```

実装も至ってシンプルです。

``` rust
impl SimpleToSwitch {
    // ...
    fn compile_raise_match(&mut self) -> (Vec<switch::Stmt>, Symbol) {
        use switch::Stmt;
        (
            vec![Stmt::RaiseMatch],
            self.symbol_generator.gensym("undefined"),
        )
    }
}
```

例外の送出は値を返さない（その場で脱出してしまうので制御が元の式に戻らない）ので返り値を表わすシンボルの扱いに困ります。
ここでは `undefined` という名前のシンボルを用意してごまかすことにします[^bottom-type]。

[^bottom-type]: 現実のコンパイラだと、例外の送出のあとは制御が戻らないことを利用して後続のコードを削除したりするので困ることは少ないです。

###### `RaiseFail` と `HandleFail` のコンパイル

`simple_case` 言語での `raise Fail` と `handle Fail => exrp` のコンパイルです。
`RaiseFail` と `HandleFail` はコンパイラが生成する式なので、無茶な使われ方をしないという前提でコンパイルできます。
具体的には「`RaiseFail` に対応する `HandleFail` が必ず同一のスコープにある」という仮定を置いてコンパイルします。
本来、例外の送出とそのハンドリングは複雑な処理ですが、この仮定を置くことでただの `goto` とラベルに置き換えられます。
すなわち、以下のような `simple_case` 言語の式を

``` sml
(... raise Fail ...) handle Fail => expr
```

以下のような `switch` 言語の式に変換できます。

``` c
  ...
  goto Catch;
  ...

 Catch:
  expr
```


さて、ここで `handle` 式は以下のようにネストすることがあることに気をつけましょう。

``` sml
((raise Fail) handle Fail => ...) handle Fail => ...
```

このとき `raise Fail` に対応するのは内側の `handle Fail` です。

`raise` とそれに対応する `handle` を探してあげる操作が必要になります。
`handle` は `raise` に最も近いものを選べばいいのでスタックを使って管理すればよさそうです。
Rustの標準ライブラリにStackはありませんが、 `Vec` で代用できます。
`SimpleToSwitch` に `local_handler_labels: Vec<Symbol>` があることを思い出しながら2つの構文のコンパイルを眺めましょう。

``` rust
impl SimpleToSwitch {
    // ...
    fn compile_raise_fail(&mut self) -> (Vec<switch::Stmt>, Symbol) {
        use switch::Stmt;
        let label = self
            .local_handler_labels
            // スタックの先頭を取り出す
            .last()
            .expect("internal error: Fail will not be handled")
            .clone();

        (
            vec![Stmt::Goto(label)],
            self.symbol_generator.gensym("undefined"),
        )
    }
}
```

`RaiseFail` のコンパイルでは、スタックの先頭にあるラベルに向かって `goto` すればよいです。

`HandleFail` のコンパイルではスタックにラベルをpushしてハンドルする式をコンパイルしたあとラベルをpopします。
ここで少し面倒なのが `handle` したあとも式なので返り値の管理をしないといけない点です。
以下のような式を考えます。

``` rust
val result = (...) handle Fail => 0;
case result of
  ...
```

これは `handle` が値を返す点と例外が起きなかった場合は例外をハンドルする式が呼ばれない点に注意して、以下のようにコンパイルできます。


``` c
  ...
  /* raiseは goto Catch に変換されているものとする */
  result := ...
  goto Finally

 Catch:
  /* 例外をハンドルしたときも値を返す */
  result := 0;

 Finally:
  /* case result of ... の変換結果 */
  ...
```

これを実装しましょう。

``` rust
impl SimpleToSwitch {
   // ...
    fn compile_handle_fail(
        &mut self,
        expr: simple_case::Expr,
        handler: simple_case::Expr,
    ) -> (Vec<switch::Stmt>, Symbol) {
        use switch::{Op, Stmt};

        let catch = self.symbol_generator.gensym("catch");
        let finally = self.symbol_generator.gensym("finally");
        let result_sym = self.symbol_generator.gensym("handle_result");
        // 一旦ラベルをpushしてコンパイルしたあと、popする
        self.local_handler_labels.push(catch.clone());
        let (mut expr_bb, expr_sym) = self.compile_expr(expr);
        self.local_handler_labels.pop();

        let (mut handler_bb, handler_sym) = self.compile_expr(handler);

        //   expr
        //   ...
        //   var := ...
        //   handle_result := var
        //   goto finally
        // catch:
        //   handler
        //   ...
        //   var' := ...
        //   handle_result := var'
        // finally:
        //   ...
        expr_bb.push(Stmt::Assign(result_sym.clone(), Op::Symbol(expr_sym)));
        expr_bb.push(Stmt::Goto(finally.clone()));
        handler_bb.insert(0, Stmt::Label(catch));
        handler_bb.push(Stmt::Assign(result_sym.clone(), Op::Symbol(handler_sym)));
        handler_bb.push(Stmt::Label(finally));

        expr_bb.extend(handler_bb);

        (expr_bb, result_sym)
    }
}
```

見た目はごちゃごちゃしていますが、やっていることは上で説明した通りです。

###### `case` のコンパイル

ここまできたら残りは `Case` です。`simple_case` 言語での `case .. of ... | ...` です。
`case` は条件式やそれぞれの腕の式と、式が多いので見た目が長くなりますがそこまで難しいことはしません。
例えば以下の `simple_case` 言語の式を考えます。

``` sml
case <cond> of
  Foo => <arm1>
  Bar (x, y) => <arm2>
  other => <arm3>
```

このとき、 `Foo` の判別子は `0` 、 `Bar` の判別子は `1` とします。
するとこれは以下のような `switch` 言語の文にコンパイルされます。

``` c
/* condの計算 */
...
cond := ...;

cond_desc = load(cond+0);
switch(cond_desc) {
  case 0: {
    <arm1>
  }
  case 1: {
    cond_data = load(cond+1);
    x := load(cond_data+0);
    y := load(cond_data+0);
    <arm2>
  }
  default: {
    other := tmp;
    <arm3>
  }
}

```

`Inject` のときと同じく式のコンパイルを端折っていますがおおむね雰囲気は掴めるかと思います。

ところで、分岐が出てくるので `case` 式の返り値の扱いに注意が必要です。
`switch`言語にコンパイルしたあとに `switch` の外で使える変数に返り値を保存しないといけません。

``` sml
switch {
  case 0: {
    /* ここの返り値 */
  };
  case 1: {
    /* ここの返り値 */
  }
  default: {
    /* ここの返り値 */
  }
};

/* ここでswitchで計算した値を使いたい */
```

LLVM IRなどのSSAではφノードと呼ばれるノードを配置しますが、ここではややこしくなるので使いません。
`switch` の各腕で同じ名前の変数に代入することでどの節を経由しても1つの変数で返り値を受け取れるようにしましょう。
これらを含めると先程のコードは詳細には以下のようにコンパイルされます。

``` c
/* condの計算 */
...
cond := ...;

cond_desc = load(cond+0);
switch(cond_desc) {
  case 0: {
    <arm1>
    switch_result := ...;
  }
  case 1: {
    cond_data = load(cond+1);
    x := load(cond_data+0);
    y := load(cond_data+0);
    <arm2>
    switch_result := ...;
  }
  default: {
    other := tmp;
    <arm3>
    switch_result := ...;
  }
}

/* ここで switch_resultでswitch文の返り値を参照できる*/
```

今回は独自に定義した `switch` 言語でのことですが、他の言語にコンパイルするときも同じようなことができます。
C言語にコンパイルするなら `switch` の手前で変数宣言しておけば問題ありません。
LLVM IRにコンパイルするなら今回のコードを多少変更するだけでφノードの挿入に変更することは容易でしょう。
あるいは一旦スタックに領域を確保してやりとりし、後の最適化で消すようにすればφノードを意識しなくてもよくなります。

`case` 式のコンパイルにはもう1つ注意点があります。
`case` 式は事実上2種類に分類できるのです。
分岐をする `case` 式（代数的データ型へのパターンマッチ）とデータを分解する `case` 式（タプルへのパターンマッチ）です。
データを分解する方では `switch` 文が不要になります。
これらは `case` 言語から `simple_case` 言語にコンパイルする時点で別々の構文にしてしまう手もありますが、今回は `case` 言語と `simple_case` 言語の差分を小さくしたかったのでそのまま同じ構文にしています。

これらの点に気をつけながら実装しましょう。

まずは `compile_case` は分岐する `case` と分岐しない `case` で場合分けして処理します。

``` rust
impl SimpleToSwitch {
   // ...
    fn compile_case(
        &mut self,
        cond: simple_case::Expr,
        clauses: Vec<(simple_case::Pattern, simple_case::Expr)>,
    ) -> (Vec<switch::Stmt>, Symbol) {
        // まずcondをコンパイルする
        let (cond_stmts, cond_symbol) = self.compile_expr(cond);

        // caseには分岐するcase（代数的データ型に対するパターンマッチ）と分岐しないcase（タプルに対するパターンマッチ）がある。
        // それをここで判別する。
        if let simple_case::Pattern::Tuple(_) = clauses[0].0 {
            // タプルパターンのコンパイル
            self.compile_case_tuple(cond_stmts, cond_symbol, clauses)
        } else {
            // 代数的データ型パターンのコンパイル
            self.compile_case_adt(cond_stmts, cond_symbol, clauses)
        }
    }

}
```

ここはそこまで難しいことはないですね。
タプルパターンの処理もそこまで難しいことはないです。

``` rust
impl SimpleToSwitch {
    // ...
    fn compile_case_tuple(
        &mut self,
        mut stmts: Vec<switch::Stmt>,
        cond_symbol: Symbol,
        mut clauses: Vec<(simple_case::Pattern, simple_case::Expr)>,
    ) -> (Vec<switch::Stmt>, Symbol) {
        use switch::{Op, Stmt};

        assert_eq!(clauses.len(), 1);
        let (pat, arm) = clauses.pop().unwrap();

        let tuple = match pat {
            simple_case::Pattern::Tuple(t) => t,
            // 代数的データ型は別で処理するのでここにはこない
            _ => unreachable!(),
        };
        let load_elements = tuple.into_iter().enumerate().map(|(i, s)| {
            Stmt::Assign(
                s,
                Op::Load {
                    base: cond_symbol.clone(),
                    offset: i as u8,
                },
            )
        });
        stmts.extend(load_elements);

        let (arm_stmts, arm_symbol) = self.compile_expr(arm);
        stmts.extend(arm_stmts);

        (stmts, arm_symbol)
    }
}
```

個々の要素をロードして変数に代入したあと、腕を実行するだけです。

代数的データ型のコンパイルは少しだけ複雑になります。

``` rust
impl SimpleToSwitch {
    // ...
    fn compile_case_adt(
        &mut self,
        mut stmts: Vec<switch::Stmt>,
        cond_symbol: Symbol,
        clauses: Vec<(simple_case::Pattern, simple_case::Expr)>,
    ) -> (Vec<switch::Stmt>, Symbol) {
        use switch::{Block, Op, Stmt};

        // それぞれの行を、判別子とそれに対応するブロックの組として集める
        let mut switch_clauses = Vec::<(i32, Block)>::new();
        // デフォルト節があれば記録する
        let mut default = None;
        // 返り値のシンボル
        let result_sym = self.gensym("switch_result");

        for (pat, arm) in clauses {
            let (arm_stmts, arm_symbol) = self.compile_expr(arm);
            // コンストラクタ、変数へのパターンマッチそれぞれをここで捌く
            match pat {
                simple_case::Pattern::Constructor { descriminant, data } => {
                    let mut block = vec![];
                    if let Some(sym) = data {
                        let load = Op::Load {
                            base: cond_symbol.clone(),
                            offset: 1,
                        };
                        block.push(Stmt::Assign(sym, load));
                    }
                    block.extend(arm_stmts);
                    block.push(Stmt::Assign(result_sym.clone(), Op::Symbol(arm_symbol)));
                    switch_clauses.push((descriminant as i32, Block(block)))
                }
                simple_case::Pattern::Variable(s) => {
                    let mut block = vec![Stmt::Assign(s.clone(), Op::Symbol(cond_symbol.clone()))];
                    block.extend(arm_stmts);
                    block.push(Stmt::Assign(result_sym.clone(), Op::Symbol(arm_symbol)));
                    // variableは必ずdefault節になる
                    default = Some(Block(block))
                }
                // タプルは別で処理するのでここにはこない
                _ => unreachable!(),
            }
        }

        stmts.push(Stmt::Switch {
            cond: cond_symbol,
            targets: switch_clauses,
            // default節がなければ（=変数パターンがなければ）default節をUNREACHABLEで埋める
            default: default.unwrap_or_else(|| Block(vec![Stmt::Unreachable])),
        });
        (stmts, result_sym)
    }
}
```

コンストラクタと変数2種類あることや返り値を表わすシンボルの扱いなどでコードが膨らみます。
とはいえ、原理通りに実装するだけなので落ち着いて追えば意味を理解できるはずです。

これで `simple_case` 言語から `switch` 言語へのコンパイルが完成しました。
コンパイルというものに馴染がない方にとっては思ったよりも簡単だったのではないでしょうか。
ブラックボックスに見えるコンパイラもソースとターゲットの関係を見極めて地道に変換していくだけです。

#### `case` 言語から `simple_case` 言語へのコンパイル

それでは `case` 言語から `simple_case` 言語へのコンパイルをみていきます。
これから、パターンマッチのコンパイルのアルゴリズムを2種類紹介するのでパターンマッチの部分だけ分離したコンパイラを先に作っておきます。
`case` 言語と `simple_case` 言語の主立った違いはパターンの部分だけですので難しいことはありません。

先にパターンマッチのコンパイルのインターフェースだけ決めてそれ以外の部分を作りましょう。
パターンのコンパイルにはスタックを使うので以下のようなインターフェースになります。

``` rust
// Rustの標準ライブラリにスタック型がないので `Vec<T>` で代用する
type Stack<T> = Vec<T>;

trait PatternCompiler {
    fn compile(
        &mut self,
        cond: Stack<(Symbol, TypeId)>,
        clauses: Vec<(Stack<case::Pattern>, simple_case::Expr)>,
    ) -> simple_case::Expr;
}
```

パターンのコンパイルの都合で、条件部分には必ず変数がくるようにします。
そうすることで条件部分を自由にコピーして使えます。

これを用いて、コンパイラは以下のように定義します。


``` rust
struct CaseToSimple<PC> {
    symbol_generator: SymbolGenerator,
    pattern_compiler: PC,
}

impl<PC> CaseToSimple<PC>
where
    PC: PatternCompiler,
{
    pub fn new(symbol_generator: SymbolGenerator, pattern_compiler: PC) -> Self {
        Self {
            symbol_generator,
            pattern_compiler,
        }
    }
    // ...
}
```

コンパイラにはパターンのコンパイラの他、 `SimpleToSwitch` と同様にシンボルジェネレータを保持します。

コンパイルは、 `case` 式以外は何も難しいことはしません。ただ内容を詰め替えているだけです。

``` rust
impl<PC> CaseToSimple<PC>
where
    PC: PatternCompiler,
{
    // ...
    pub fn compile(&mut self, case: case::Expr) -> simple_case::Expr {
        match case {
            case::Expr::Tuple(vs) => self.compile_tuple(vs),
            case::Expr::Inject { descriminant, data } => {
                self.compile_inject(descriminant, data.map(|d| *d))
            }
            case::Expr::Case { cond, ty, clauses } => self.compile_case(*cond, ty, clauses),
            case::Expr::Symbol(s) => self.compile_symbol(s),
        }
    }

    fn compile_tuple(&mut self, data: Vec<case::Expr>) -> simple_case::Expr {
        simple_case::Expr::Tuple(data.into_iter().map(|d| self.compile(d)).collect())
    }

    fn compile_inject(&mut self, descriminant: u8, data: Vec<case::Expr>) -> simple_case::Expr {
        simple_case::Expr::Inject {
            descriminant,
            data: data.into_iter().map(|d| self.compile(d)).collect(),
        }
    }
    fn compile_symbol(&mut self, s: Symbol) -> simple_case::Expr {
        simple_case::Expr::Symbol(s)
    }

    fn gensym(&mut self, hint: impl Into<String>) -> Symbol {
        self.symbol_generator.gensym(hint)
    }
}
```

`case` 式も本体はパターンコンパイラを呼ぶだけなので別に難しいことをする訳ではないです。
条件部分は式ではなく変数が来るという条件があったのでその処理をします。

以下のような式を考えます。

``` sml
case <cond> of <pattern> => <arm> | ...
```

これを以下のように書き換えます。

``` sml
let tmp = <cond> in
case tmp of <pattern> => <arm> | ...
```

それを実装するとこうなります。

``` rust
impl<PC> CaseToSimple<PC>
where
    PC: PatternCompiler,
{
    // ...
    fn compile_case(
        &mut self,
        cond: case::Expr,
        ty: TypeId,
        clauses: Vec<(case::Pattern, case::Expr)>,
    ) -> simple_case::Expr {
        let cond = self.compile(cond);
        let clauses = clauses
            .into_iter()
            .map(|(pat, arm)| (vec![pat], self.compile(arm)))
            .collect();
        let v = self.gensym("v");

        simple_case::Expr::Let {
            expr: Box::new(cond),
            var: v.clone(),
            body: Box::new(self.pattern_compiler.compile(vec![(v, ty)], clauses)),
        }
    }
}
```

これで `CaseToSimple` の枠組みはできました。

ここから一旦頭を実装から原理の理解に切り替えましょう。
残りのパターンのコンパイルアルゴリズムを見ていきます。

#### バックトラック法

まず紹介するのはバックトラック法です。
ここで紹介するアルゴリズムは[@LeFessant:2001:OPM:507635.507641]に基くものですが、SMLのパターンマッチに合わせて多少の改変を加えています。
具体的にはタプルを追加したのと代数的データ型の各コンストラクタの引数を高々1つに制限しています。

バックトラック法でコンパイルされたパターンマッチは、マッチ判定中にパターンを先に進んだり後に戻ったりするのが特徴です。
そのため最悪のケースで `case` 式に書かれた全てのパターンを辿ることもあります。
一方でコードはそれ以上複雑にならないので全てのパターンの量に比例したサイズのコードにしかなりません。

実装コードだけみても分かりづらいと思うのでまずはアルゴリズムの基本的な部分を押さえ、そして実装に移りたいと思います。

##### 動作イメージ

まずはコンパイルする前に、どういう動作をするコードを出力したいのかのイメージを掴みましょう。

以下のコードを例にとって動作を考えます。

``` sml
case ([], [true]) of
    (_,  []) => []
  | ([], _ ) => []
  | (x::xs, y::ys) => (x, y) :: zip xs ys
```

まずはタプルの先頭のみに注目します

編: ハイライト入れた画像とかに差し替えたい

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

1行目は失敗したので、次のかたまりにうつります。
先程は2つ目の要素で失敗したので1つ巻き戻して1つ目の要素を調べます。

``` sml
case ([], ...) of
  ...
  | ([], ...) => []
  | (x::xs, ...) => (x, y) :: zip xs ys
```

複数のコンストラクタは同時に調べられるので2行目、3行目を同時に調べます。
するとこれは1つ目の節にマッチします。

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

ワイルドカードパターンは無条件にマッチするのでこれで決定し、 2行目が採用されます。
結果、`[]` が返ります。

動作イメージが掴めたでしょうか。
これに相当するコードを吐くコンパイラを作っていきます。

##### 原理

パターンをコンパイルするのは一見すると複雑そうですね。
ですがマッチに使われているパターンを調べてパターンの種類で分岐してあげると、1つ1つのルールはシンプルなものになります。
また、パターン自体がネスト可能なのでコンパイルするアルゴリズムも再帰的なものになります。

TBD: 準備自体はバックトラック法と決定木法で共通なのでくくりだした方がいい？
いくつか準備しておきます。

まず、対象になる式は `case` 式、つまり `case <条件> of <パターン1> => <式1> | ` の形をした式なのでした。
この `<条件>` と `<パターン>` の列それぞれに着目します。

`条件` の部分についてです。`条件` には必ず変数がくるようにしておきます。
これを **条件変数** と呼ぶことにします [^cond-var]。
条件部分を変数にするのはパターンのコンパイルの過程で条件部分をコピーすることがあるからです。
もし変数以外の式がきたら、事前に `let` を使って変形しておきます。

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


構文の上での `条件` は1つだけですが、パターンマッチのコンパイル中は複数あるものとして扱います。
これはタプルパターンをコンパイルするときに、タプルを一旦分解してから1つ1つ順にコンパイルしていくのに必要だからです。
もう少し具体的にいうと、パターンはネスト可能なので木になりますが、それを深さ優先探索で処理したいため *条件式をスタックで保持します* 。
これを **条件変数スタック** と呼ぶことにします。

`パターン` の部分についてです。パターンも条件に連動して複数に分割されます。
節は複数ありますから、さらにパターンを複数に分割するとなると、パターンを保持するデータ構造は2次元配列になります。
やや用語の乱用の嫌いがありますが、これを **パターン行列** と呼ぶことにします。
行列とはいうもののかけ算や足し算をしたりだとか行列式を求めたりだとかはしないので線形代数が苦手でも安心して下さい。
節には腕もついているので、パターン行列にはこっそり腕も入れておくことにします。

おおまかには条件変数スタックとパターン行列が、パターンマッチをコンパイルする関数への入力になります。
具体例をみましょう。 `zip` 関数のパターンマッチをコンパイルすることを考えます。

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

また、 `v` から始まるものを変数パターン、 `p` からはじまるものを一般のパターンとします。
`C` ではじまるものをコンストラクタ、 `c` ではじまるものを条件変数とし、 `e` ではじまるものを一般の式とします。

それではこの記法を用いて個別のルールを見ていきましょう。

###### 空則

単純なルールからいきましょう。
パターンが空ならばそれは必ずマッチします。
パターンマッチは上にあるものを優先するルールから、一番上の腕に変換されます。
これは基底ケースに相当します。

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


ここでは行が1つ以上あることを仮定しています。
SMLでは節が1つもない `case` 式は書けませんし、他の則でも行が少くとも1つあるパターン行列しか生成しないのでこの仮定は妥当です。

これで空の場合は拾えたので以降は条件変数が1つ以上ある場合を扱います。

###### 変数則

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

###### コンストラクタ則

もしパターン行列の1列目のパターン全てがコンストラクタなら、条件変数スタックの先頭を `case` で分岐して次に進みます。
ここの処理は少し複雑です。先頭のコンストラクタをキーにしてで節をグルーピングする処理が必要だからです。

先に少し具体的な例を考えましょう。以下のようなコンストラクタ `C1` 、 `C2` を持つ代数的データ型のパターンマッチを考えます。

``` sml
compile [c1, c2, ...] [
  ([C1, p12, ...] => e1),
  ([C2, p22, ...] => e2),
  ([C1, p32, ...] => e3),
]
```

コンストラクタ `C1` が2回登場していますね。
これは同じコンストラクタをひとつにまとめて以下のように変換されてほしいです。

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

もう1つ具体例を考えましょう。
今度はコンストラクタが引数をとる場合を考えます。
`C1` が引数をとり、 `C2` は引数をとらない場合を考えましょう。

``` sml
compile [c1, c2, ...] [
  ([C1 p11, p12, ...] => e1),
  ([C2    , p22, ...] => e2),
  ([C1 p31, p32, ...] => e3),
]
```

これはだいたい先程とおなじように変換されてほしいですね。ただしコストラクタパターンの引数にもパターンがあるのでそれも考慮しないといけません。
そうすると以下のようになるはずです。

編: p11、p31にハイライト入れたい

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

`C1` の腕ではコンストラクタの引数を一旦変数パターンで受け取って、それを条件変数スタックの先頭にもってきています。
また、それぞれのコンストラクタパターンの引数にあったパターンもパターン行列の1列目に追加しています。

これらの具体例を元にコンパイル処理を考えましょう。
まず必要なのが1列目が同じコンストラクタパターンの行を1箇所にまとめる処理です。
この処理をする関数を `specialize` と名付けます。
`specialize` はコンストラクタの判別子とパターン行列を引数にとってパターン行列を返します。

`specialize` ができたとします。
コンストラクタ `C1` の判別子を `C1d` 、 `C2` の判別子を `C2d` と書くことにするると、`specialize` を使ってコンパイル処理はこう書けますね。

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

もし、パターン行列の1列目に列挙されたコンストラクタ群が `c1` に定義されたコンストラクタ群より少なければデフォルト節も必要になります。
その場合はパターンマッチに失敗するので $\mathlt{FAIL}$ を置きます。

``` sml
case c1 of
    C1 arg => compile [arg, c2, ...] (specialize C1 pattern_matrix)
  | C2     => compile [c2, ...] (specialize C2 pattern_matrix)
  | ...
  | _ => FAIL
```


コンストラクタによって引数があったりなかったりするので、`case` を書くときは柔軟に対応します。
それぞれの節の腕はコンストラクタの判別子を使って `pattern_matrix` を `specialize` しています。

`specialize` は、指定されたコンストラクタパターンの節のみを集めます。
このとき、コンストラクタに引数があることもあるので柔軟な処理が必要です。
おおまかには以下のような処理をします。

``` sml
specialize constructor pattern_matrix = {for each row of pattern_matrix}
  if pi1 is constructor     => collect it as ([pi2, ...]      => ei)
  if pi1 is constructor arg => collect it as ([arg, pi2, ...] => ei)
  else ignore
{end}
```

後程、この処理を実際のプログラミング言語で実装するので概略を覚えておいて下さい。

###### タプル則

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

$N$ 要素のタプルなら一時変数を $N$ 個用意して束縛します。
とりたてて難しいところはありません。

###### 混合則

もしパターン行列の1列目に変数パターンもコンストラクタもあるなら、それを分割しましょう。


``` sml
compile [c1, c2, ...] [
  ([C11, p12, ...] => e1),
  ([v21, p22, ...] => e2),
  ...
]
```

上記のパターン行列は1列目にコンストラクタパターンと変数パターンが交じっています。
1行目と2行目の先頭パターンがそれぞれコンストラクタパターン、変数パターンと異なっているのでそこで分割します。
分割した結果、 `compile condvar pattern_matrix1 || compile condvar pattern_matrix2` という見た目になります。
総合すると下記のような見た目になります。

``` sml
compile [c1, c2 ...] [
  ([C11, p12, ...] => e1),
] || compile [c1, c2] [
  ([v21, p22, ...] => e2),
  ...
]
```

この例では先にコンストラクタ、後に変数が出てきていますが、逆でも同じです。

パターン行列を分割するとそれぞれがカバーできる範囲も狭くなるので、マッチしなくなることがあります。
マッチしなかったら次のパターン行列を試すのですが、そのために $\mathlt{FAIL}$ と $\|$ を使います。
上記の例だと1つ目のパターン行列はコンストラクタが1つしかないのでマッチに失敗するかもしれません。
そのときはコンストラクタ則で $\mathlt{FAIL}$ が生成されます。
そして $\mathlt{FAIL}$ は $\|$ で拾われて、2つ目のパターン行列に処理が移るという具合です

##### 具体例

ルールだけだとどういう動きをするのか分かりづらいのでコンパイルの具体例をみてみましょう。
ネストしたパターンの `case` 式からネストしていないパターンの `case` 式に変換します。

動作イメージのときと同じ `zip` のパターンマッチを例にとりましょう。

```sml
case (xs, ys) of
    (_,  []) => []
  | ([], _ ) => []
  | (x::xs, y::ys) => (x, y) :: zip xs ys
```

これをコンパイルします。
まずは条件部分を変数にして、 `compile` 関数からはじめます。
このとき、 `v::vs` は脱糖すると `::` がコンストラクタ、 タプルの `(v, vs)` が引数であることに注意して下さい。

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

次のパターン行列がでてきました。
1列目に変数パターンとコンストラクタパターンがありますね。
*混合則* で2つに分割します。

```sml
compile [tmp2, tmp3] [
  ([_,  []]      => [])
]
|| (* 編注： ←例の記号 *)
compile [tmp2, tmp3] [
  ([[], _ ],      => []),
  ([x::xs, y::ys] => (x, y) :: zip xs ys)
]
```

$\|$ と $\mathit{FAIL}$ の具体的な実装はアルゴリズムの実装のときに考えることにします。

さて、それぞれのパターンをコンパイルしていきます。
まず上の方。

``` sml
compile [tmp2, tmp3] [
  ([_,  []]      => [])
]
```

1列目はワイルドカードパターンだけです。
これは *変数則* で以下のような `case` 式にコンパイルできます。

``` sml
case tmp2 of
    _ => compile [tmp3] [
      ([[]] => [])
    ]
```

次の1列目はコンストラクタパターンですので *コンストラクタ則* を適用します。
ですが、これは網羅的ではありません。書いたパターンにマッチしなかった場合に（=default節に） `FAIL` をおぎないつつ、コンストラクタで分岐するコードにコンパイルします。

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

これの1列目はコンストラクタパターンだけがあります。
*コンストラクタ則* を適用します。

```sml
case tmp2 of
    []      => compile [tmp3] [[_] => []]
  | :: tmp4 => compile [tmp4, tmp3] [[(x, xs), y::ys] => (x, y) :: zip xs ys]
```

上の `[]` 節の方はまたワルドカードパターンなので *変数則* を適用します。
そのあと *空則* を適用します。一気にやってしまいましょう。

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


また `::` パターンがでてきました。
先程と同様なのでコンストラクタ則とタプル則の適用を同時にやってしまいます。

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

最後に、 $\mathlt{FAIL}$  が残った場合の安全弁として $\|$ で $\mathlt{FAIL}$ を拾ってパターンマッチに失敗した例外 `raise Match` を上げます。
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

コンパイル後のコードを見て、変数が複数回使われているのにお気付きでしょうか。
最初の `case` 式で `tmp2` 、 `tmp3` と分岐を進めたあとに `FAIL` にでくわすと、 `||` に拾われて再度 ``tmp2` 、 `tmp3` と分岐を進めていきます。
進んだり戻ったりするのがバックトラック法の由来です。


`zip` の例を通して、空則、変数則、コンストラクタ則、タプル則、混合則全ての適用例をみました。
アルゴリズムの動作イメージをつかんだので最後に実装していきましょう。

##### 実装

頭を動かすパートが終わったので手を動かすパートに移りましょう。
パターンマッチ以外の部分は終わっているので、パターンマッチのアルゴリズム部分を見ていきます。

パターンマッチのコンパイラは `CaseToSimple` とは別に実装します。
`BackTrackPatternCompiler` を定義し、そこにパターンマッチのコンパイルを実装していきましょう。
まずは構造体の定義からです。

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

それでは準備が済んだので、 `compile` メソッドにパターンマッチのコンパイルを実装していきます。
`compile` の型はこのようになります。

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

1点補足説明をします。
この時点で型推論は終わっている前提なので、`cond` には型IDが含まれています。
型は代数的データ型のヴァリアントを列挙するのに必要になります。

それでは実装を進めましょう。
まずは原理通り、空則、変数則、タプル則、コンストラクタ則、タプル則、混合則で場合分けします。

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

というように適用しています。
ここでベクトルをスタックとして代用しているので1列目は `last()` で取り出していることに注意して下さい。

それでは個々の則の実装を眺めてみましょう。

###### 空則

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

###### 変数則

次は変数則です。
`compile [c1, c2, ...] [([v11, p12...] => e1), ([v21, p22, ...] => e2), ...]` を `compile [c2, ...] [([p12...] => let val v11 = c1 in e1 end), ([p22, ...] => let val v21 = c1 in e2 end), ...]` に書き換えます。

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

一見長いように見えますがやっていることは説明の通りです。
分かりやすいように誤魔化しを入れた記法と比べて実装は泥臭くなりがちです。

ここまでは簡単なルールでした、これ以降のルールは少し複雑です。

###### タプル則

タプル則に移りましょう。
タプル則では `compile [c1, c2, ...] [([(q11, q12, ..), p12, ...] => e1), ([(q21, q22, ...), p22, ...] => e2), ...]` を
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

###### 混合則

次に混合則です。
これはパターンの種類が変わったところで分割するだけなのでそれ自体は際立って説明するところはありません。
ところで、ここで $\|$ と $\mathlt{FAIL}$ が登場するのでその実装について触れておきましょう。

$\|$ は左辺の式が普通の値ならそのままの値を、 $\mathlt{FAIL}$ なら右辺の式の評価をしてその値を返します。
$\mathlt{FAIL}$ が発生したら即座に $\|$ までジャンプしたいです。
このような性質を見たす機能として例外とそのハンドラがあります。
なのでここでは $\mathlt{FAIL}$ を `raise Fail` に、 $e1 \| e2$ を `e1 handle Fail => e2` にエンコードすることにします。

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



###### コンストラクタ則

最後は最難関のコンストラクタ則です。
おおまかな処理の流れは以下になります。

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

ほぼ説明した通りですが、4.の `case` 式の生成のところで分岐が入っています。
コンストラクタが網羅的か同化でデフォルト節を挿入するかを分岐しています。
網羅的でない場合は $\mathlt{FAIL}$ を挟むので、実装の方でも `_ => raise Fail` を追加しています。

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

最後に、 `BackTrackPatternCompiler` にパターンコンパイラのインタフェース `PatternCompiler` を実装します。

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

`Fail` は実装内部の例外なので捕捉してユーザ向けに `Match` として投げなおします。

ここまでで、バックトラック法の原理と実装を紹介してきました。

#### 決定木法

バックトラック法に続いて決定木法に移ります。
ここで紹介するアルゴリズムは[@Maranget:2008:CPM:1411304.1411311]に基くものですが、バックトラック方と同じくSMLのパターンマッチに合わせて改変を加えています。

決定木法はバックトラック法と違ってパターンを先に進めることしかしません。そのため、マッチ判定が高速に終わります。
また、決定木法には副産物として嬉しい性質があります。
全ての可能性を列挙することから自然と網羅性判定もついてくるのです。

一方でマッチ判定の全ての可能性をコードに落としてしまうのでパターンの書き方によっては生成されるコードサイズが爆発的に大きくなってしまいます。
コードサイズが増えるということは、生成されるバイナリが大きくなるというだけでなくてコンパイル時間の増大も意味します。

バックトラック法と同じく動作イメージからみていきましょう。

##### 動作イメージ

バックトラック法と比較しやすいように同じコード例を使って動作イメージを確認しましょう。

以下のコードを考えます。

``` sml
case ([], [true]) of
    (_,  []) => []
  | ([], _ ) => []
  | (x::xs, y::ys) => (x, y) :: zip xs ys
```


まずはパーターンの先頭行に着目します。
先頭行には2要素目にコンストラクタパターンがあります。
ここを起点に分岐したいので、パターンマッチ全体で2要素目に注目します。

編: ハイライト入れた画像とかに差し替えたい

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

残りは1要素になりました。
また1行目に着目します。
最初と同じく1コンストラクタパターンがあるので、ここを起点に分岐します。

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

動作イメージが掴めたでしょうか。
これに相当するコードを吐くコンパイラを作っていきます。


##### 原理

決定木法もバックトラック法と同様に、いくつかのルールから成ります。
ルール名や一部の補助関数はバックトラック法と同じ名前をしていますが、中身は別物ということに注意して下さい。

###### 空則

パターン行列の行数が0の場合、マッチは失敗します。

つまり、以下のようなコンパイルは

``` sml
compile [c1, ...] []
```

$\mathlt{FAIL$} へとコンパイルされます

``` sml
FAIL
```

ここでの $\mathlt{FAIL$} の扱いですが、バックトラック法と違って $\mathlt{FAIL}$ が出たら即座にパターンが非網羅的であることが言えます。
なのでこの時点でコンパイルエラーにしたり、警告を出したりしても構いません。

###### 変数則

もし、先頭の行が全て変数であれば、その行にマッチします。
特に、列数が0の場合もこの規則があてはまります。

ここでは空則で行数が0の場合を除いたのでここでは行が1つ以上あることを仮定しています。

変数則が決定木法でのコンパイルアルゴリズムの基底になります。

つまり、以下のようなコンパイルは

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


###### 混合則


空則も変数則もあてはまらない場合、先頭行のどこかの列に変数ではないパターンが存在します。
混合則ではこの変数ではないパターンを処理します。

まず、先頭行の変数ではないパターンが $i$ 列目にあるとすると、それを1列目と交換します。
このとき、他の行のパターンや条件変数も列を交換します。

編: 1列目とi列目を入れ替えたことを表わす矢印とか入れたい

``` sml
compile [c1, c2, ..., ci, ..., cn] [
  ([v11, v12, ..., p1i, ..., v1n] => E1),
  ([v21, v22, ..., v2i, ..., v2n] => E2),
  ...
]
```

↓

``` sml
compile [ci, c2, ..., c1, ..., cn] [
  ([p1i, v12, ..., v11, ..., v1n] => E1),
  ([v2i, v22, ..., v21, ..., v2n] => E2),
  ...
]
```


これでパターンの先頭列に対して操作を行えます。

参照元の論文ではコンストラクタしか想定していませんが、今回のパターンにはタプルもあるのでコンストラクタかパターンかで分岐します。
それぞれの場合をみていきましょう。
まずは比較的簡単なタプルの場合から

####### タプルの場合

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

パターン行列の1列目のパターンはタプルの形をしているか変数の形をしているかです。
それぞれの処理を考えます。
タプルの場合はパターンを分解します。
変数の場合はタプルを分解したあとの各要素は `_`  で無視し、 `ci` の方を使います。


まとめると、1行1列のパターンをタプル、2行1列のパターンを変数とすると、 `compile` は以下のように変換します。

``` sml
case ci of
  (d1, ..., dn) => compile [d1, ..., dn, ..., c1, ..., cn] [
    ([q11, ..., q1m, ..., v11, ..., v1n] => E1),
    ([_  , ..., _  , ..., v21, ..., v2n] => let val v2i = ci in E2 end),
    ...
  ]
```

####### コンストラクタの場合

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

まず、先頭パターンのコンストラクタの判別子を集めてきます。
各行の1列目はコンストラクタパターン `Ca pki` か、変数パターンのはずです。
このうちコンストラクタパターンの `Ca` の部分を集めてきます。
コンストラクタの引数は無視して判別子の部分だけ取り出すのです。
ここで取り出した判別子の集合を $\Sigma$ としておきましょう。

次に、判別子ごと、つまり $\Sigma$ の要素ごとの特殊化行列を作ります。
特殊化行列を作る関数を `specialize` とします。
`specialize` は引数に条件変数、判別子、パターン行列をとってパターン行列を返します。
バックラック法でも `specialize` はでてきましたが、それとは異なったものです。


パターン行列の特殊化は以下の疑似コード `specialize` で表現されます。
引数があるコンストラクタとないコンストラクタがあるのでその場合分け、コンストラクタパターンと変数パターンがあるのでその場合分けで都合4通りの処理をします。

``` sml
specialize condvar descriminant pattern_matrix = {for each row of pattern_matrix}
  if pi1 is constructor and
     descriminant matches   => collect it as ([pi2, ...]      => ei)
  if pi1 is constructor arg and
     descriminant matches   => collect it as ([arg, pi2, ...] => ei)
  if pi1 is variable and
     constructor has no arg => collect it as ([pi2, ...]      => let pi1 = condvar in ei end)
  if pi1 is variable and
     constructor has an arg => collect it as ([_, pi2, ...]   => let pi1 = condvar in ei end)
  else ignore
{end}
```

バックトラック法の `specialize` との違いは、引数に条件変数も増えている点と、変数パターンも含めるか否かです。
バックトラック法ではコンストラクタが正確に一致する行を集めるのに対して、決定木法ではコンストラクタにマッチするパターンを1列目に持つ行全てを集めています。

$\Sigma$ に `C1` 〜 `Ca` までのコンストラクタの判別子が集まったとしましょう。
このとき、 元の `compile` 関数の引数にあったパターン行列を `pattern_matrix` とすると `compile` は1段階進んで以下のようになります。

``` sml
case ci of
   C1 [arg] => compile [arg, c2, ...] (specialize ci C1 pattern_matrix)
|  C2 [arg] => compile [arg, c2, ...] (specialize ci C2 pattern_matrix)
   ...
```

ここで、 $\Sigma$ に集まったコンストラクタが網羅的でなければ追加の処理があります。
`_ => ...` の形のデフォルト節を追加します。
このデフォルト節には変数パターンの行が含まれます。

追加するためのデフォルト節の腕を構築する `default_patterns` も定義しましょう。
疑似コードで以下のような処理をします。

``` sml
default_patterns condvar pattern_matrix = {for each row of pattern_matrix}
  if pi1 is variable  => collect it as ([pi2, ...] => let pi1 = condvar in ei end)
  else ignore
{end}
```

これを使って、最終的に `case` は以下のようになります。

``` sml
case ci of
   C1 [arg] => compile [arg, c2, ...] (specialize ci C1 pattern_matrix)
|  C2 [arg] => compile [arg, c2, ...] (specialize ci C2 pattern_matrix)
   ...
|  _        => compile [c2, ...] (default_patterns ci, pattern_matrix)
```


少し複雑ですが、これが決定木法の原理です。



決定木法について少し考察してみましょう。
もし、パターンが網羅的でなかったら何が起こるでしょう。
最後の `default_patterns` が空の行列（行数が0の行列）を返します。
これは次のステップで空則にあたるので $\mathlt{FAIL}$ になります。
逆に、行数が0の行列を作りうるのは `default_patterns` のみです。
パターンが網羅的でない場合でしか $\mathlt{FAIL}$ はでてきません。
このことからも決定木法は網羅性判定を内包することがよく分かるでしょう。


注意深く見ると、決定木法ではそれぞれの条件変数が高々1度しか `case` に使われていないのが分かるかと思います。このおかげで実行が速くなっています。
一方で `specialize` や `default_patterns` で変数で始まる列をコピーしています。このため、コンパイルしたあとのコードは大きくなりがちです。

##### 具体例

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

<!-- 「全てが変数」ではないの意。全てが「変数でない」と紛らわしくない言いまわしがある？ -->
パターン行列の先頭行のパターン全てが変数ではありません。
*混合則* を適用します。
タプルパターンがあるのでそれを先頭にもってきて、 *タプルの場合* に則って分解します。

```sml
case (xs, ys) of
  (tmp1, tmp2) => compile [tmp1, tmp2] [
    ([_,  []]       => []),
    ([[], _ ]       => []),
    ([x::xs, y::ys] => (x, y) :: zip xs ys)
]
```

次に出てきたパターン行列も、先頭行のパターンは全てが変数ではありません。
*混合則* を適用します。
先頭行のパターンでは、2列目がコンストラクタパターン `[]` を持つます。
これを先頭にもってきて処理を進めます。

```sml
case (xs, ys) of
  (tmp1, tmp2) => compile [tmp2, tmp1] [
    ([[],  _]       => []),
    ([_ , []]       => []),
    ([y::ys, x::xs] => (x, y) :: zip xs ys)
]
```

先頭にもってきたあとの1列目はリストへのパターンマッチです。
なので *コンストラクタの場合* の処理に進みます。
リストのコンストラクタは `[]` と `::` の2つあります。
それぞれにマッチする行を `specialize` で集めます。

```sml
case (xs, ys) of
  (tmp1, tmp2) => case tmp2 of
      [] => compile [tmp1] (specialize tmp2 [] [
        ([[],  _]       => []),
        ([_ , []]       => []),
        ([y::ys, x::xs] => (x, y) :: zip xs ys)
      ])
    | :: tmp3 => compile [tmp3, tmp2] (specialize tmp2 :: [
        ([[],  _]       => []),
        ([_ , []]       => []),
        ([y::ys, x::xs] => (x, y) :: zip xs ys)
    ])
```

1列目が `[]` の条件式にマッチするのは1行目と2行目です。
同じく1列目が `::` にマッチするのは2行目と3行目です。
つまり、それぞれの `specialize` を処理すると以下のようになります。

```sml
case (xs, ys) of
  (tmp1, tmp2) => case tmp2 of
      [] => compile [tmp1] [
        ([_]        => let val _ = tmp2 in [] end),
        ([[]]       => let _ = tmp2 in [] end),
      ]
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
      [] => let val _ = tmp1 in [] end
    | :: tmp3 => compile [tmp3, tmp2] [
        ([_      , []   ] => let _ = tmp2 in [] end),
        ([(y, ys), x::xs] => (x, y) :: zip xs ys)
    ]
```

次に、 `::` の腕です。これは先頭行が全てパターンではありません。
そこで *混合則* を適用します。
コンストラクタパターン `[]` がある2列目を先頭にもってきます。

```sml
compile [tmp2, tmp3] [
    ([[]   , _      ] => let _ = tmp2 in [] end),
    ([x::xs, (y, ys)] => (x, y) :: zip xs ys)
]
```

いれかえたあとの1列目はリストへのパターンマッチですので *コンストラクタの場合* に進みます。
リストのコンストラクタは `[]` と `::` があります。
それぞれのコンストラクタで場合分けして、それぞれの腕に `specialize` を置きます。

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

結果、以下のようになります。

```
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

ルールの適用が長いですし、コンパイルされるコードも大きくなりました。
しかし分岐の数は少なくなっています。
分岐があるコード（`case` に `|` がついているコード）は2箇所だけです。
分岐のネストも最短1、最長2と、とても小さくなっています。

コンパイル後のコードを見て、木になっていることにお気付きでしょうか。
これが決定木法の由来です。

`zip` の例を通して、変数則、混合則のタプルの場合、コンストラクタの場合の適用例をみました。
今回のパターンは網羅的だったので空則はでてきていません。
アルゴリズムの動作イメージをつかんだので最後に実装していきましょう。

##### 実装

それでは実装に移りましょう。
バックトラック法と同じくパターンマッチのアルゴリズム部分のみを見ていきます。

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


バックトラック法と同じですね。
`compile` もほぼ同様です。

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

###### 空則

バックトラック法と同じく、空則と変数則は実装が簡単です。
空則の実装は以下になりす。


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

空則は $\mathlt{FAIL}$ するのでした。
そしてこの $\mathlt{FAIL}$ はパターンマッチが非網羅的な場合にのみ出現するのでした。
そのままマッチに失敗したときの例外、 `Match` を投げることにしましょう。
本物のコンパイラであればコンパイラエラー、あるいは警告を出す手段が用意してあるはずなので、その仕組みを使ってユーザに非網羅的であることを伝えることになります。

###### 変数則

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

変数則も原理をそのまま適用するだけです。
変数をそのまま条件変数の別名となるように処理して、腕の式 `expr` を評価します。

###### 混合則

次は混合則です。
混合則はタプルの場合とコンストラクタの場合でそれぞれ場合分けがありました。
タプル、コンストラクタそれぞれ別メソッドに切り出すことにします。
すると混合則の実装はこうなります。

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

変数でないパターンをみつけたらそれを先頭と交換したあと、タプル、またはコンストラクタで分岐します。
ここで、 `find_nonvar` の実装はこうなっています。

``` rust
impl DecisionTreePatternCompiler {
    fn find_nonvar(&mut self, clauses: &[(Stack<case::Pattern>, simple_case::Expr)]) -> usize {
        clauses[0].0.iter().rposition(|p| !p.is_variable()).unwrap()
    }

    // ...
}
```

変数でないパターンは1行に複数あることがあります。
そのうちどれを選んでもいいのですが[^nonvar]、ここではシンプルに先頭のものを選んでいます。

[^nonvar]: どれを選んでも正しく動作するのですが、コンパイルしたコードのパフォーマンスは変わります。これについては後程軽く触れます。

それではタプルの場合、コンストラクタの場合をそれぞれ見ていきましょう。

####### タプルの場合

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

次にパターン行列のそれぞれの行の先頭パターンに対して操作をしていきます。
原理のところで説明したとおり、タプルパターンならその場に展開し、
変数パターンなら展開された要素は `_` で無視して `let val v2i = ci in e2 end` を生成します。

原理自体も多少複雑ですが実装も行数が嵩みます。
これはタプルの要素数が複数ありますが、疑似コードでは `(e1, ..., en)` とごまかしていた部分を実際のコードでは愚直に繰り返し処理を書かないといけないからです。
こういった面でも理論と実装のギャップがあります。

####### コンストラクタの場合

次は混合則のコンストラクタの場合です。
これは原理のところで説明が長かったことからも分かるように、実装も長くなります。
まずはメソッド全体を掲載します。

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
        let descriminants = clause_with_heads
            .iter()
            .filter_map(|(head, _)| match head {
                case::Pattern::Constructor { descriminant, .. } => Some(*descriminant),
                _ => None,
            })
            .collect::<HashSet<_>>();
        // 2. コンストラクタ毎に特殊化行列を作る
        let mut clauses = descriminants
            .iter()
            .map(|&descriminant| {
                let clauses = self.specialized_patterns(
                    sym.clone(),
                    &ty,
                    descriminant,
                    clause_with_heads.iter(),
                );
                // 本来ならコンストラクタごとに、引数の有無で処理が微妙に変わる。
                // そこを `Option` 型のメソッドで違いを吸収している。
                let param_ty = self.type_db.param_ty_of(&ty, descriminant);
                let tmp_var = param_ty.clone().map(|_| self.symbol_generator.gensym("v"));
                let mut new_cond = cond.clone();
                new_cond.extend(tmp_var.iter().cloned().zip(param_ty).rev());
                let pat = simple_case::Pattern::Constructor {
                    descriminant,
                    data: tmp_var,
                };
                let arm = self.compile(new_cond, clauses);
                (pat, arm)
            })
            .collect();

        // 3. コンストラクタが網羅的でなければデフォルト行列を作る
        if self.is_exhausitive(&ty, descriminants) {
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


コメントだけ拾い読みすれば分かりますが、
大枠は最初に実装の都合上の処理をしたあとは、原理どおりに3ステップを処理を進めているだけです。

要点は原理のところで説明してあるので、実装上の注意点を紹介します。

まず、コンストラクタの場合の処理では先頭パターンをよく使います。
今回の実装ではスタックをベクタで代用しているので先頭パターンはベクタの末尾にあります。
これを取り出すのは少し手間がかかるので最初に取り出してしまっています。

2のコンストラクタ毎に特殊化行列を作るのところは見た目は長いですが、落ち着いてよく見ると原理通りの実装です。
`Self::specialized_patterns` で特殊化行列を作ったあと、必要ならばコンストラクタの引数を受け取る変数パターンを用意してパターン行列の行を作っています。

3のコンストラクタが網羅的でなければデフォルト行列を作るの箇所では `is_exhausitive` を使って場合分けしています。
網羅的な場合は素直な実装を、網羅的でない場合は原理通りにデフォルト行列を作っています。

それでは `compile_constructor` で使われた3つのメソッド、 `specialized_patterns` 、 `default_patterns` 、 `is_exhausitive` を掲載します。

まずは `specialized_patterns` です。

``` rust
impl DecisionTreePatternCompiler {
    fn specialized_patterns<'a, 'b>(
        &'a mut self,
        cond: Symbol,
        type_id: &TypeId,
        descriminant: u8,
        clause_with_heads: impl Iterator<
            Item = &'b (case::Pattern, (Stack<case::Pattern>, simple_case::Expr)),
        >,
    ) -> Vec<(Stack<case::Pattern>, simple_case::Expr)> {
        let param_ty = self.type_db.param_ty_of(&type_id, descriminant);
        clause_with_heads
            .filter_map(|(head, clause)| match head {
                // 判別子が一致するコンストラクタパターンはそのままあつめる
                case::Pattern::Constructor {
                    descriminant: d,
                    pattern,
                } if *d == descriminant => Some((pattern.clone().map(|p| *p), clause.clone())),
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
                // descriminantが一致しないコンストラクタは無視
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

`specialized_pattern` はパターン行列から指定されたコンストラクタにマッチする行全てを集めてくる処理でした。
コードでも、（先頭列を取り出した）パターン行列の各行に対して以下の処理をしています

1. 先頭列のパターンがコンストラクタでかつ判別子が一致するならそれを集める
2. 先頭列のパターンが変数パターンならば適切に加工した上でそれを集める

ここで、1. の処理でパターンガードを使っています。

``` rust
match head {
  Constructor { ... } if *d == descriminant => { ... },
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

`default_patterns` は先頭列が変数パターンな行を集めてくるメソッドなのでした。
実際、 `.filter(|(head, _)| head.is_variable())` を使ってそのように実装されています。

最後に `compile_constructor` で使われていたメソッドの3つめ、 `is_exhausitive` に移りましょう。

``` rust
impl DecisionTreePatternCompiler {
    fn is_exhausitive(
        &self,
        type_id: &TypeId,
        descriminansts: impl IntoIterator<Item = u8>,
    ) -> bool {
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

パターン内で使われている `descriminant` の集合が、データ型に定義された全てのコンストラクタの判別子の集合と一致しているかで判定しています。
Rustでは集合を実装した型 `HashSet` を `==` で比較すると集合同士の比較の意味になります。

以上でパターンマッチのコンパイルアルゴリズム2種類を含めたコンパイラが完成しました。
いかがだったでしょうか、原理と比べて思ったより分量が多いと感じたのではないでしょうか。
枝葉末節を省いた原理は理解しやすくはあるのですが、実際のコードに落としたときとのギャップが大きくなりがちです。
そのギャップを埋めた本記事は読者の皆様に有用であると信じています。
