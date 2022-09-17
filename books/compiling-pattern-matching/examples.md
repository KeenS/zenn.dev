---
title: コンパイルの動作例
---

せっかく動くコンパイラを構築したのでいくつか入力をあたえて動かしてみましょう。全体としては、以下のような流れで、型情報や式を用意したあと `case` -> `simple_case` 、 `simple_case` -> `case` のコンパイルと進みます。

```rust
let mut type_db = case::TypeDb::new();
// 型情報の登録

let mut sg = SymbolGenerator::new();

// コンパイルしたい式
let expr = ...;

// case -> simple_caseへのコンパイル
let mut compiler = CaseToSimple::new(
    sg.clone(),
    // ここではバックトラック法を選択
    BackTrackPatternCompiler::new(sg.clone(), type_db.clone()),
);
let sc = compiler.compile(expr.clone());

// simple_case -> switch へのコンパイル
let mut compiler = SimpleToSwitch::new(sg.clone());
let s = compiler.compile(sc);

```

ユーザが与えるパラメータは型情報、コンパイルしたい式、 `case` から `simple_case` へのコンパイル時のパターンのコンパイルのアルゴリズムです。

例として、 インタプリタのところで用意したブール値のXORをコンパイルして出力してみましょう。
少し長くなります。

```
{
    dis@19 := 1;
    v@20 := alloc(1);
    store(v@20 + 0, dis@19);
    dis@21 := 1;
    v@22 := alloc(1);
    store(v@22 + 0, dis@21);
    tuple@23 := alloc(2);
    store(tuple@23 + 0, v@20);
    store(tuple@23 + 1, v@22);
    v@16 := tuple@23;
    v@17 := load(v@16, 0);
    v@18 := load(v@16, 1);
    switch(v@17) {
        case 0: {
            switch(v@18) {
                case 0: {
                    dis@30 := 1;
                    v@31 := alloc(1);
                    store(v@31 + 0, dis@30);
                    switch_result@29 := v@31;
                }

                case 1: {
                    dis@32 := 0;
                    v@33 := alloc(1);
                    store(v@33 + 0, dis@32);
                    switch_result@29 := v@33;
                }

                default: {
                    UNREACHABLE;
                }

            };
            switch_result@28 := switch_result@29;
        }

        case 1: {
            switch(v@18) {
                case 0: {
                    dis@35 := 0;
                    v@36 := alloc(1);
                    store(v@36 + 0, dis@35);
                    switch_result@34 := v@36;
                }

                case 1: {
                    dis@37 := 1;
                    v@38 := alloc(1);
                    store(v@38 + 0, dis@37);
                    switch_result@34 := v@38;
                }

                default: {
                    UNREACHABLE;
                }

            };
            switch_result@28 := switch_result@34;
        }

        default: {
            UNREACHABLE;
        }

    };
    switch_result@27 := switch_result@28;
    handle_result@26 := switch_result@27;
    goto finally@25;
  catch@24:
    sml_raise(SML_EXN_MATCH);
    handle_result@26 := undefined@39;
  finally@25:
    return handle_result@26;
}
```


一見分かりづらいですが、ちゃんとコンパイルできていますね。コンパイル結果が長いのは途中の最適化をしていないので比較的無駄の多いコードが生成されたままになっているからです。XORについてはバックトラック法と決定木法で違いはないので決定木法の結果は省略します。

次に、差分の出る `zip` の（ような）例を見てみましょう。元のSMLの式は以下のようなものです。

```sml
datatype list = Nil | Cons () * List
case (Nil, Nil) of
    (Nil, _) => ()
  | (_, Nil) => ()
  | (Cons (_, _), Cons(_, _)) => ()
```

このデータ型を表現するのは少し労力が必要です。タプルにも名前が必要な設計なので、ここで登場する3種類のタプルもそれぞれ定義してあげないといけないからです。

```rust
// ()
type_db.register_tuple(TypeId::new("unit"), vec![]);
// Consの引数
type_db.register_tuple(
    TypeId::new("cons"),
    vec![TypeId::new("unit"), TypeId::new("list")],
);
// caseで使われるタプル
type_db.register_tuple(
    TypeId::new("2list"),
    vec![TypeId::new("list"), TypeId::new("list")],
);
// listの定義
type_db.register_adt(
    TypeId::new("list"),
    vec![
        // Nil
        case::Constructor {
            discriminant: 0,
            param: None,
        },
        // Cons () * List
        case::Constructor {
            discriminant: 1,
            param: Some(TypeId::new("cons")),
        },
    ],
);
```


これを元に `zip` （のようなもの）を記述します。

```rust
let expr = {
    use case::*;
    use Expr::*;

    // `Nil` （値）のつもり
    let nilv = Expr::Inject {
        discriminant: 0,
        data: None,
    };

    // `Cons` （コンストラクタ）のつもり
    fn consv(arg: Expr) -> Expr {
        Expr::Inject {
            discriminant: 1,
            data: Some(Box::new(arg)),
        }
    }

    // `()` （値）のつもり
    let unitv = Expr::Tuple(vec![]);

    // 2要素タプルパターンの便利メソッド
    fn tuple2p(p1: Pattern, p2: Pattern) -> Pattern {
        Pattern::Tuple(vec![p1, p2])
    }

    // `Nil` （パターン）のつもり
    let nilp = Pattern::Constructor {
        discriminant: 0,
        pattern: None,
    };

    // `Cons` （パターン）のつもり
    fn consp(car: Pattern, cdr: Pattern) -> Pattern {
        Pattern::Constructor {
            discriminant: 1,
            pattern: Some(Box::new(tuple2p(car, cdr))),
        }
    }

    // _ パターン。
    // 衝突しないために毎度シンボルジェネレータを使って生成する
    fn wild(sg: &mut SymbolGenerator) -> Pattern {
        Pattern::Variable(sg.gensym("_"))
    }

    // case (Nil, Nil) of
    //     (Nil, _) => ()
    //   | (_, Nil) => ()
    //   | (Cons(_, _), Cons(_, _)) => ()
    Case {
        cond: Box::new(Tuple(vec![nilv.clone(), nilv.clone()])),
        ty: TypeId::new("2list"),
        clauses: vec![
            (tuple2p(nilp.clone(), wild(&mut sg)), unitv.clone()),
            (tuple2p(wild(&mut sg), nilp.clone()), unitv.clone()),
            (
                tuple2p(
                    consp(wild(&mut sg), wild(&mut sg)),
                    consp(wild(&mut sg), wild(&mut sg)),
                ),
                unitv.clone(),
            ),
        ],
    }
}
```

これをバックトラック法、決定木法それぞれでコンパイルしてみましょう。

バックトラック法

```
{
    dis@29 := 0;
    v@30 := alloc(1);
    store(v@30 + 0, dis@29);
    dis@31 := 0;
    v@32 := alloc(1);
    store(v@32 + 0, dis@31);
    tuple@33 := alloc(2);
    store(tuple@33 + 0, v@30);
    store(tuple@33 + 1, v@32);
    v@16 := tuple@33;
    v@17 := load(v@16, 0);
    v@18 := load(v@16, 1);
    switch(v@17) {
        case 0: {
            _@10 := v@18;
            tuple@42 := alloc(0);
            switch_result@41 := tuple@42;
        }

        default: {
            _@28 := v@17;
            goto catch@38;
            switch_result@41 := undefined@43;
        }

    };
    handle_result@40 := switch_result@41;
    goto finally@39;
  catch@38:
    switch(v@18) {
        case 0: {
            _@11 := v@17;
            tuple@48 := alloc(0);
            switch_result@47 := tuple@48;
        }

        default: {
            _@27 := v@18;
            goto catch@44;
            switch_result@47 := undefined@49;
        }

    };
    handle_result@46 := switch_result@47;
    goto finally@45;
  catch@44:
    switch(v@17) {
        case 1: {
            v@19 := load(v@17, 1);
            v@20 := load(v@19, 0);
            v@21 := load(v@19, 1);
            switch(v@18) {
                case 1: {
                    v@22 := load(v@18, 1);
                    v@23 := load(v@22, 0);
                    v@24 := load(v@22, 1);
                    _@15 := v@24;
                    _@14 := v@23;
                    _@13 := v@21;
                    _@12 := v@20;
                    tuple@54 := alloc(0);
                    switch_result@53 := tuple@54;
                    switch_result@52 := switch_result@53;
                }

                default: {
                    _@25 := v@18;
                    goto catch@34;
                    switch_result@52 := undefined@55;
                }

            };
            switch_result@51 := switch_result@52;
            switch_result@50 := switch_result@51;
        }

        default: {
            _@26 := v@17;
            goto catch@34;
            switch_result@50 := undefined@56;
        }

    };
    handle_result@46 := switch_result@50;
  finally@45:
    handle_result@40 := handle_result@46;
  finally@39:
    switch_result@37 := handle_result@40;
    handle_result@36 := switch_result@37;
    goto finally@35;
  catch@34:
    sml_raise(SML_EXN_MATCH);
    handle_result@36 := undefined@57;
  finally@35:
    return handle_result@36;
}
```

決定木法

```
{
    dis@68 := 0;
    v@69 := alloc(1);
    store(v@69 + 0, dis@68);
    dis@70 := 0;
    v@71 := alloc(1);
    store(v@71 + 0, dis@70);
    tuple@72 := alloc(2);
    store(tuple@72 + 0, v@69);
    store(tuple@72 + 1, v@71);
    v@58 := tuple@72;
    v@59 := load(v@58, 0);
    v@60 := load(v@58, 1);
    switch(v@59) {
        case 0: {
            _@10 := v@60;
            tuple@75 := alloc(0);
            switch_result@74 := tuple@75;
        }

        case 1: {
            v@62 := load(v@59, 1);
            switch(v@60) {
                case 0: {
                    _@61 := v@62;
                    _@11 := v@59;
                    tuple@77 := alloc(0);
                    switch_result@76 := tuple@77;
                }

                case 1: {
                    v@63 := load(v@60, 1);
                    v@64 := load(v@63, 0);
                    v@65 := load(v@63, 1);
                    v@66 := load(v@62, 0);
                    v@67 := load(v@62, 1);
                    _@12 := v@66;
                    _@13 := v@67;
                    _@15 := v@65;
                    _@14 := v@64;
                    tuple@80 := alloc(0);
                    switch_result@79 := tuple@80;
                    switch_result@78 := switch_result@79;
                    switch_result@76 := switch_result@78;
                }

                default: {
                    UNREACHABLE;
                }

            };
            switch_result@74 := switch_result@76;
        }

        default: {
            UNREACHABLE;
        }

    };
    switch_result@73 := switch_result@74;
    return switch_result@73;
}
```

先頭列がコンストラクタ、変数、コンストラクタと並んでいるのでバックトラック法では3分割されています。それに呼応してトップレベルに `switch` 文が3度でてきています。

一方決定木法ではタプルで2つのデータ型に対してマッチしているので `switch` 文も2重にネストしています。そして決定木法では必ずトップレベルの `switch` 文は1つになります。

今回の例では差はつきませんでしたが、全体のswitch文の数、要するに生成されるコード量ではバックトラック法の方が小さくなりがちです。ここまで読んできた読者の方々なら自力でswitch文の数に違いがでるコード例を構築できるはずです。興味のある方は試してみて下さい。

