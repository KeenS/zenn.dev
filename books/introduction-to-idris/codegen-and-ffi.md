---
title: "CodegenとFFI"
---

本章ではあまり触れてこなかったIdrisコンパイラの裏側と、FFIについて学習します。

<!--more-->

# C Codegen

いままであまり触れてきませんでしたが、Idrisは裏でCのコードを生成し、Cコンパイラがバイナリを作っています。コンパイラオプションをいじることでそれを垣間見ることができます。

簡単なコードを書いて実験してみましょう。以下のコードを `codegen.idr` という名前で保存します。

```idris:codegen.idr
main : IO ()
main = putStrLn "Hello"
```

コンパイラに `-S` （`--codegenonly`）オプションを渡すことで、コンパイラが生成したCコードがみれます。

```shell-session
$ idris -S codegen.idr -o codegen.c
$ ls codegen.c
codegen.c
```

中身を覗いてみましょう。長いので一部省略します。

```c:codegen.c
#include "math.h"
#include "idris_rts.h"
#include "idris_bitstring.h"
#include "idris_stdfgn.h"
void* _idris_assert_95_unreachable(VM*, VAL*);
void* _idris_call_95__95_IO(VM*, VAL*);
void* _idris_idris_95_crash(VM*, VAL*);
void* _idris_io_95_bind(VM*, VAL*);
void* _idris_io_95_pure(VM*, VAL*);
void* _idris_Main_46_main(VM*, VAL*);
void* _idris_mkForeignPrim(VM*, VAL*);
void* _idris_prim_95__95_asPtr(VM*, VAL*);
void* _idris_prim_95__95_eqManagedPtr(VM*, VAL*);
void* _idris_prim_95__95_eqPtr(VM*, VAL*);
void* _idris_prim_95__95_managedNull(VM*, VAL*);
void* _idris_prim_95__95_null(VM*, VAL*);
void* _idris_prim_95__95_peek16(VM*, VAL*);
void* _idris_prim_95__95_peek32(VM*, VAL*);
void* _idris_prim_95__95_peek64(VM*, VAL*);
void* _idris_prim_95__95_peek8(VM*, VAL*);
void* _idris_prim_95__95_peekDouble(VM*, VAL*);
void* _idris_prim_95__95_peekPtr(VM*, VAL*);
void* _idris_prim_95__95_peekSingle(VM*, VAL*);
void* _idris_prim_95__95_poke16(VM*, VAL*);
void* _idris_prim_95__95_poke32(VM*, VAL*);
void* _idris_prim_95__95_poke64(VM*, VAL*);
void* _idris_prim_95__95_poke8(VM*, VAL*);
void* _idris_prim_95__95_pokeDouble(VM*, VAL*);
void* _idris_prim_95__95_pokePtr(VM*, VAL*);
void* _idris_prim_95__95_pokeSingle(VM*, VAL*);
void* _idris_prim_95__95_ptrOffset(VM*, VAL*);
void* _idris_prim_95__95_readChars(VM*, VAL*);
void* _idris_prim_95__95_readFile(VM*, VAL*);
void* _idris_prim_95__95_registerPtr(VM*, VAL*);
void* _idris_prim_95__95_sizeofPtr(VM*, VAL*);
void* _idris_prim_95__95_stderr(VM*, VAL*);
void* _idris_prim_95__95_stdin(VM*, VAL*);
void* _idris_prim_95__95_stdout(VM*, VAL*);
void* _idris_prim_95__95_vm(VM*, VAL*);
void* _idris_prim_95__95_writeFile(VM*, VAL*);
void* _idris_prim_95__95_writeString(VM*, VAL*);
void* _idris_prim_95_io_95_bind(VM*, VAL*);
void* _idris_run_95__95_IO(VM*, VAL*);
void* _idris_unsafePerformPrimIO(VM*, VAL*);
void* _idris_world(VM*, VAL*);
void* _idris__123_APPLY_95_0_125_(VM*, VAL*);
void* _idris__123_APPLY2_95_0_125_(VM*, VAL*);
void* _idris__123_EVAL_95_0_125_(VM*, VAL*);
void* _idris__123_runMain_95_0_125_(VM*, VAL*);
void* _idris_io_95_bind_95_IO_95__95_idr_95_108_95_34_95_108_95_36_95_case(VM*, VAL*);

// ....

void* _idris_Main_46_main(VM* vm, VAL* oldbase) {
    INITFRAME;
loop:
    RESERVE(1);
    ADDTOP(1);
    LOC(1) = MKSTR(vm, "Hello""\x0a""");
    LOC(1) = MKINT((i_int)(idris_writeStr(stdout,GETSTR(LOC(1)))));
    RVAL = NULL_CON(0);
    TOPBASE(0);
    REBASE;
}


// ...
```

典型的なCコードジェネレータの吐いたコードです。`RESERVE` や `LOC` などはCのマクロになっています。恐らくですがIdrisのコンパイラが内部で `RESERVE` や `LOC` のような命令を持っており、それに対応するコードを生成するようなCのマクロが定義されているのです。例えば `ADDTOP` であれば以下のように定義されます。

```c
#define ADDTOP(x) vm->valstack_top += (x)
```


因みにここで使われている `idris_writeStr` は `idris_stdfgn.c` で定義されていて、以下のような実装になっています。

```c
int idris_writeStr(void* h, char* str) {
    FILE* f = (FILE*)h;
    if (fputs(str, f) >= 0) {
        return 0;
    } else {
        return -1;
    }
}
```

意外と素直な実装です。


# JavaScript Codegen

Cのコードを吐くのがデフォルトの挙動ですが、別のコードジェネレータを使うこともできます。コードジェネレータ自体はプラグインとして作れるのでサードパーティも含めれば色々あるのですが、コンパイラに同梱されているものにJavaScriptがあります。

こちらもコード生成してみましょう。 `--codegen javascript` をつけるとJavaScriptのコードを吐いてくれます。

```shell-session
$ idris --codegen javascript -S codegen.idr -o codegen.js
```

JavaScriptのバックエンドの方は比較的小さいのでそのまま貼ってみます。

```javascript:codegen.js
"use strict";

(function(){

const $JSRTS = {
    throw: function (x) {
        throw x;
    },
    Lazy: function (e) {
        this.js_idris_lazy_calc = e;
        this.js_idris_lazy_val = void 0;
    },
    force: function (x) {
        if (x === undefined || x.js_idris_lazy_calc === undefined) {
            return x
        } else {
            if (x.js_idris_lazy_val === undefined) {
                x.js_idris_lazy_val = x.js_idris_lazy_calc()
            }
            return x.js_idris_lazy_val
        }
    },
    prim_strSubstr: function (offset, len, str) {
        return str.substr(Math.max(0, offset), Math.max(0, len))
    }
};
$JSRTS.prim_systemInfo = function (index) {
    switch (index) {
        case 0:
            return "javascript";
        case 1:
            return navigator.platform;
    }
    return "";
};

$JSRTS.prim_writeStr = function (x) { return console.log(x) };

$JSRTS.prim_readStr = function () { return prompt('Prelude.getLine') };

$JSRTS.die = function (message) { throw new Error(message) };




const $HC_0_0$MkUnit = ({type: 0});
// Main.main

function Main__main($_0_in){
    const $_1_in = $JSRTS.prim_writeStr("Hello\n");
    return $HC_0_0$MkUnit;
}

// {runMain_0}
function $_0_runMain(){
    return $JSRTS.force(Main__main(null));
}


$_0_runMain();
}.call(this))
```

生成したコードはそのままNodeで動かせます。

```text
$ node codegen.js
Hello
```

試してないですがブラウザでも動くはずです。

因みにNode専用ならNodeバックエンドもあるので `--codegen node` という書き方もできます。未確認ですがこっちの方が標準出入力の扱いが上手そうです。

```javascript
// Node codegenのランタイムのコード抜粋
$JSRTS.prim_writeStr = function (x) { return process.stdout.write(x) };

$JSRTS.prim_readStr = function () {
    var ret = '';
    var b = Buffer.alloc(1024);
    var i = 0;
    while (true) {
        $JSRTS.fs.readSync(0, b, i, 1)
        if (b[i] == 10) {
            ret = b.toString('utf8', 0, i);
            break;
        }
        i++;
        if (i == b.length) {
            var nb = Buffer.alloc(b.length * 2);
            b.copy(nb)
            b = nb;
        }
    }
    return ret;
};
```

# C FFI

IdrisにはFFIの仕組みがあります。特にデフォルトのcodegenバックエンドであるCのFFIは重要です。例えばプレリュードの `File` は以下のようにFFIをベースに組み立てられています。


```idris
data File : Type where
  FHandle : (p : Ptr) -> File

export
fflush : File -> IO ()
fflush (FHandle h) = foreign FFI_C "fflush" (Ptr -> IO ()) h
```


公式でバックエンドを複数提供しつつプレリュードでCに依存するのはどうなんだというツッコみはありますが、それはおいておいてファイルIOなどの基本的操作が言語組み込みではなくてユーザランドで実装されているのは楽しいですね。

さて、もう少しFFIについて解説しておくと、キーになるのは `foreign` 関数です。

```text
Idris> :type foreign
foreign : (f : FFI) -> ffi_fn f -> (ty : Type) -> {auto fty : FTy f [] ty} -> ty
Idris> :doc FTy
Data type FTy : FFI -> List Type -> Type -> Type
    
    
    The function is: public export
Constructors:
    FRet : ffi_types f t -> FTy f xs (IO' f t)
        
        
        The function is: public export
    FFun : ffi_types f s -> FTy f (s :: xs) t -> FTy f xs (s -> t)
        
        
        The function is: public export
```

C FFIに限っていえば `foreign C_FFI "関数名" (対応するIdrisの型)` という使い方になるでしょうか。

C FFIを使うにあたって、Cのオブジェクトファイルをリンクする必要がありますよね？それ専用のIdrisのディレクティブもあります。実例で確かめてみましょう。まずはリンクするCのオブジェクトファイルを用意しておきましょう。一番シンプルな内容でいきます。以下の内容を `ffi.h` に保存しておきます。

```c:ffi.h
int
add(int, int);
```


そして `ffi.c` に以下を記述します。

```c:ffi.c
#include "ffi.h"

int
add(int x, int y) {
  return x + y;
}
```

これは一旦コンパイルしておきましょう。

```shell-session
$ gcc -c -o ffi.o ffi.c
```


これを使うIdrisのコードを書きます。

Cのオブジェクトファイルを使う手段として、2つのディレクティブがあります。 `%include` と `%link` です。それぞれ `%include バックエンド "インクルードするファイル名"` 、 `%link バックエンド "リンクするファイル名"` の構文です。ドキュメントにはちゃんと載ってないのですが、CにおいてはそれぞれCPPの `#include`  、コンパイラの引数へと変換されると思っていてよさそうです。

これをふまえて、まず `ffi.idr` の先頭に以下を書きます。

```idris:ffi.idr
%include C "ffi.h"
%link C "ffi.o"
```

`int add(int, int);` を呼び出すコードを書きましょう。FFIをした返り値は `IO` でないといけないようなので以下のコードを書きます。

```idris:ffi.idr
ffiAdd : Int -> Int -> IO Int
ffiAdd = foreign FFI_C "add" (Int -> Int -> IO Int)
```

これを使う `main` はこう書きましょう。

```idris:ffi.idr
main : IO ()
main = do
  ret <- ffiAdd 1 2
  printLn ret
```


これをコンパイル・実行します。

```shell-session
$ idris ffi.idr -o ffi
$ ./ffi
3
```

簡単にではありますがC FFIを実行できました。

因みに `%include` は本当に `#include "..."` しているだけのようです。試しに `%include` してcodegenしてみたらファイルの先頭に `#include` が足されてました。

```c
// ↓これ
#include "ffi.h"
#include "math.h"
#include "idris_rts.h"
#include "idris_bitstring.h"
#include "idris_stdfgn.h"
void* _idris_Prelude_46_Bool_46__38__38_(VM*, VAL*);
// ...
```

# JavaScript FFI

Cと同様にJavaScrptバックエンドでもFFIができます。Cと同じように `add` 関数を定義した `ffi.js` を用意します。

```javascript:ffi.js
function add(x, y) {
    return x + y;
}
```

`%include` はCと同様です。`%link` はどうも意味を成さないようです。

```idris:ffi_js.idr
%include JavaScript "ffi.js"
```

C FFIと同じようなコードを書くと、このような書き方になります。

```idris:ffi_js.idr
ffiAdd : Int -> Int -> JS_IO Int
ffiAdd = foreign FFI_JS "add(%0, %1)" (Int -> Int -> JS_IO Int)
```

* `IO` は `JS_IO` になる
* 関数名のところは関数名ではなく、テンプレートを書く
  + `%0.method(%1, %2)` のような書き方もできるよう

これを使う `main`も以下のようになります。

```idris:ffi_js.idr
main : JS_IO ()
main = do
  ret <- ffiAdd 1 2
  printLn' ret
```

`main` も `IO` ではなく `JS_IO` になります。

このコードをコンパイル・実行してみましょう。

```shell-session
$ idris --codegen javascript ffi_js.idr -o ffi_gen.js
$ node ffi_gen.js
3
```

動いているようです。

因みに `Node` バックエンドを使うときは `%include` の第一引数が `JavaScript` ではなく `Node` になります。それ以外は共通です。両方とも同じファイルをインクルードするなら2つともまとめて書いてしまえばよいでしょう。

```idris
%include JavaScript "ffi.js"
%include Node       "ffi.js"
```

JavaScriptバックエンドに関しては興味のある方が多いかと思いますが、申し訳ないことに私がそこまで詳しくないのであんまり詳細な内容が書けません。参考リンクを貼っておくので各自で試行錯誤してみて下さい。

* [IdrisでWebアプリを書く](https://www.slideshare.net/tanakh/idrisweb)
* [New Foreign Function Interface — Idris 1.3.3 documentation](http://docs.idris-lang.org/en/latest/reference/ffi.html#javascript-ffi-descriptor)

因みにCと同様、 `%include` は本当にファイルの中身を展開しているだけのようです。

```javascript:ffi_gen.js
// ...

$JSRTS.prim_writeStr = function (x) { return console.log(x) };

$JSRTS.prim_readStr = function () { return prompt('Prelude.getLine') };

$JSRTS.die = function (message) { throw new Error(message) };
$JSRTS.jsbn = (function () {
  // ...
}).call(this);

// ↓ ここ
function add(x, y) {
    return x + y;
}

// ...
```

# まとめ

Idrisがバックエンドを複数持つこと、それぞれのバックエンドにFFIがあることを学びました。バックエンドの切り替えはコンパイラオプションででき、FFIは専用のディレクティブや関数を使うことで実現できました。

本章もそれなりにドキュメントの薄い部分でしたが次章はさらにドキュメントがほとんどなく手探りでしか使えない機能、Elaboratorリフレクションについて学びます。
