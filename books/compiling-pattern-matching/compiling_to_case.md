---
title: シンプルなパターンからC風言語へのコンパイル
---

本項では `simple_case` 言語、 `switch` 言語、 `SimpleToCase` コンパイラを実装します。

### `simple_case` 言語と `swich` 言語

`simple_case` 言語は `case` 言語とさほど変わりません。大きく違うのはパターンの表現力が異なる点です。
`case` 言語ではパターンのネストや非網羅的パターン、冗長なパターンなどが許されていましたが、
`simple_case` 言語ではそれらが許されません。パターンはネストせず、必ず網羅的で、非冗長的です。
その他に `case` 言語になかった変数束縛があります。これはパターンのコンパイルの際にあると便利だからです。

また、 `simple_case` 言語にはパターンマッチに失敗したときの例外 `Match` を上げる機能が増えています。
そしてパターンマッチ内部で使う例外 `Fail` を上げる機能と、それをハンドルする機能もあります。
これについてはバックトラック法の実装のところで触れることにします。

#### `simple_case` 言語

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

#### `swicch` 言語

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


### `case` 言語への機能追加

コンパイラを書く前に、 `case` 言語に機能を追加しておきます。
といっても構文を拡張する訳ではありません。
インタプリタを書く際に必要ない機能を飛ばしてしまったのでそれを補完します。

#### 型DB

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

#### ユーティリティ

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

### SimpleToSwitchコンパイラ

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

#### `Symbol` のコンパイル

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

#### `Let` のコンパイル

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

#### `Inject` のコンパイル

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

#### `Tuple` のコンパイル

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

#### `RaiseMatch` のコンパイル

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

#### `RaiseFail` と `HandleFail` のコンパイル

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

#### `case` のコンパイル

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

