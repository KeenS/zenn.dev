---
title: case言語からsimple_case言語へのコンパイル
---

それでは `case` 言語から `simple_case` 言語へのコンパイルをみていきます。これから、パターンマッチのコンパイルのアルゴリズムを2種類紹介するのでパターンマッチの部分だけ分離したコンパイラを先に作っておきます。 `case` 言語と `simple_case` 言語の主立った違いはパターンの部分だけですので難しいことはありません。

先にパターンマッチのコンパイルのインターフェースだけ決めてそれ以外の部分を作りましょう。パターンのコンパイルにはスタックを使うので以下のようなインターフェースになります。

```rust
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

パターンのコンパイルの都合で、条件部分には必ず変数がくるようにします。そうすることで条件部分を自由にコピーして使えます。

これを用いて、コンパイラは以下のように定義します。


```rust
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

```rust
impl<PC> CaseToSimple<PC>
where
    PC: PatternCompiler,
{
    // ...
    pub fn compile(&mut self, case: case::Expr) -> simple_case::Expr {
        match case {
            case::Expr::Tuple(vs) => self.compile_tuple(vs),
            case::Expr::Inject { discriminant, data } => {
                self.compile_inject(discriminant, data.map(|d| *d))
            }
            case::Expr::Case { cond, ty, clauses } => self.compile_case(*cond, ty, clauses),
            case::Expr::Symbol(s) => self.compile_symbol(s),
        }
    }

    fn compile_tuple(&mut self, data: Vec<case::Expr>) -> simple_case::Expr {
        simple_case::Expr::Tuple(data.into_iter().map(|d| self.compile(d)).collect())
    }

    fn compile_inject(&mut self, discriminant: u8, data: Vec<case::Expr>) -> simple_case::Expr {
        simple_case::Expr::Inject {
            discriminant,
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

`case` 式も本体はパターンコンパイラを呼ぶだけなので別に難しいことをする訳ではないです。条件部分は式ではなく変数が来るという条件があったのでその処理をします。

以下のような式を考えます。

```sml
case <cond> of <pattern> => <arm> | ...
```

これを以下のように書き換えます。

```sml
let tmp = <cond> in
case tmp of <pattern> => <arm> | ...
```

それを実装するとこうなります。

```rust
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

ここから一旦頭を実装から原理の理解に切り替えましょう。残りのパターンのコンパイルアルゴリズムを見ていきます。

