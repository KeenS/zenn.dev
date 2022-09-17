---
title: パターンマッチのC言語へのコンパイル
---

それではいよいよ、パターンマッチを実行可能なコードへとコンパイルする方法を説明しましょう。SMLのパターンマッチをC言語へと変換する方法について、変換前と変換後のコードを見比べることで説明していきます。手続的でコンピュータにおける処理がわかりやすいC言語のコードへと変換された状態を見ることで、動作をイメージしやすくなるでしょう。

パターンマッチのような複雑な機能は、多くのコンパイラでは段階を分けてコンパイルします。具体的には、まずネストのあるパターンからネストのないパターンに変換します。そして、ネストのないパターンを、C言語にあるような `switch` 文へと変換します。本稿ではコンパイルの話には深く立ち入りませんが、このように複数の段階を経てコンパイルされるということは覚えておいてください（図6.1上）。

![図6.1 SMLからCに変換する際のコードとデータのフロー図](/images/compiling_pattern_matching/sml_to_c.png)
*図6.1 SMLからCに変換する際のコードとデータのフロー図*

以降では、パターンマッチのコンパイルを見る前に、代数的データ型のコンパイルについて説明します。なお、代数的データ型については、段階を分けることなくそのままC言語のコードに対応付けられます（図6.1下）。

## 代数的データ型のコンパイル

まずはSMLの代数的データ型をC言語のどのようなデータ型に変換すべきかを考えます。単純なものから考えていきましょう。最初の方に出てきた `week` はこのように定義されていました。

```sml:リスト6.2
datatype week = Sunday | Monday | Tuesday | Wednesday | Thursday | Friday | Saturday
```
*データ型weekの定義（再掲）*

これは以下のようなC言語の列挙型へと変換すればいいでしょう。

```c:リスト6.3
enum week {
  Sunday,
  Monday,
  Tuesday,
  Wednesday,
  Thursday,
  Friday,
  Saturday
};
```
*week型の変換後*

次は`person` 型です。

```sml:リスト6.4
datatype person = MkPerson of string * int
```
*データ型personの定義（再掲）*

この代数的データ型は、単純には以下のようなC言語の構造体へと変換できます。

```sml:リスト6.5
enum person_constructor {
  MkPerson
};

struct tuple_string_int {
  sml_string _1;
  sml_int _2
};

struct person {
  enum person_constructor tag;
  struct tuple_string_int *data;
};
```
*person型の変換後*

`person`型は、`week`型と違ってコンストラクタが1つなので、11行めの `tag` には用がありません。この`tag`は、実際にはコンパイラの最適化で消えることになります。しかし、まずは基本的な形を押さえるために、ここでは残した形を示しておきます。

次は `entry` 型です。`entry` 型は以下のように定義されていました。

```sml:リスト6.6
datatype entry = File of string | Directory of string * entry list
```
*entry型の定義（再掲）*


`entry`型には、ピタリと対応するデータ型がC言語にはありません。`string` または `string * entry list` を保持するので構造体ではないですし、列挙型だとデータを保持できません。

しかし、タグ付き `union` と呼ばれる表現を使えば、C言語にエンコードできます。列挙型で `File` または `Directory` を、共用体で `string` または `string * entry list` を表現すればエンコードできるのです。

```c:リスト6.7
enum entry_constructor {
  File,
  Directory
};

struct tuple_string_entry_list {
  sml_string _1;
  /* リストの定義はふわっとしておく */
  sml_list   _2;
}

struct entry {
  enum entry_constructor tag;
  union {
    sml_string file;
    struct tuple_string_entry_list *directory;
  } data;
};
```
*entry型の変換後*


キーワードが多いので、初見だと少し身構えるかもしれません。しかし、よく見れば難しいことはしていないので、少しずつ解読していきます。

まず、 `struct entry` の構造を把握します。一見すると複雑ですが、フィールドは `tag` と `data` の2つだけです（図6.8）。

![図6.8 構造体`entry`のフィールド](/images/compiling_pattern_matching/structentry.png)
*図6.8 構造体`entry`のフィールド*


`tag` は `enum` で定義されているので、 `File` または `Directory` の2値しか取りません。

`data` は `union` で定義されています。その `union` のフィールドはというと、 `file` か `directory` の2つです。どちらが入っているかによってメモリの使われ方が変わります。`file` の場合は `sml_string` 型の値が格納されますが、これは実装次第です。ひとまず3ワード分を使うものとしておきましょう。`directory` の場合は `struct tuple_string_entry_list` へのポインタです。メモリは1ワード分を使います（図6.9）。


![図6.9 共用体`data`のフィールド](/images/compiling_pattern_matching/union1.png)
*図6.9 共用体`data`のフィールド*


これで2通りのデータ型を同じ型、同じメモリ領域で表現できるようになりました。

一見すると、この共用体 `data` だけで、SMLの `entry` 型を表現できているように見えるかもしれません。しかし、まだ大きな違いがあります。SMLの `datatype` では、 `File` か `Directory` かをプログラム上もパターンマッチで区別できます。しかし、C言語の `union` の場合にはプログラム上で両者を区別する手段がなく、どちらが入っているかはプログラマーが知っているものと仮定して使われます。どちらかが入るかをプログラマーが知っていれば、そもそも `union` を使う必要がありません。そのため、現実的には追加でデータを持たせる必要があります。それが `tag` です。`tag` が `File` のときには `file` のほうにデータが入っていると約束し、 `tag` が `Directory` のときは `directory` のほうにデータが入っていると約束して使うことで、両者を区別できます。

まとめると、 `struct entry` のメモリは以下のような使われ方をします（図6.10）。

![図6.10 構造体`entry`の使われ方](/images/compiling_pattern_matching/union2.png)
*図6.10 構造体`entry`の使われ方*

以上で、SMLの `datatype`に対応するデータ型を作れるようになりました。
この表現を見ると、代数的データ型が時に**タグ付きユニオン**と呼ばれることが理解できると思います。

## 単純なパターンマッチのコンパイル

前項では、SMLの代数的データ型がどうC言語へとコンパイルされるかを見ました。
前項のとおりに代数的データ型が翻訳されるという前提で、次はパターンマッチをC言語へとコンパイルした結果について考えていきます。
まずは、ネストのないシンプルなパターンマッチの例として、「 `entry` 型の値を受け取って、そこに含まれているファイル名を表示する `printPathname` 」を見てみましょう。

`printPathname` は以下のように定義されているのでした。

```sml:リスト6.11
fun printPathname prefix e =
    case e of
        File name => print (prefix ^ name ^ "\n")
      | Directory (name, entries) => let val prefix = prefix ^ name ^ "/"
                                     in app (printPathname prefix) entries end
```
*printPathname関数の定義（再掲）*

これをC言語へとコンパイルすることを考えます。まず、代数的データ型である`entry` 型は以下のように翻訳されるのでした。

```c:リスト6.12
enum entry_constructor {
  File,
  Directory
};

struct tuple_string_entry_list {
  sml_string _1;
  sml_list   _2;
}

struct entry {
  enum entry_tag tag;
  union {
    sml_string file;
    struct tuple_string_entry_list *directory;
  } data;
};
```
*entry型の変換後（再掲）*

`printPathname` は、おおまかにはリスト6.13のように変換されます。本当はパターンマッチ以外の処理も必要なので、コンパイラの内部の処理で型が付いたり名前が変わったりしますが、ここでは無視しています。

```c:リスト6.13
void
printPathname(sml_string prefix, struct entry *e)
{
  swtich(e->tag) {
    case File: {
      sml_string name = e->data.file;
      /* 処理 */
      break;
    }
    case Directory: {
      string name = e->data.directory->_1;
      sml_list entries = e->data.directory->_2;
      /* 処理 */
      break;
    }
  }
}
```
*printPathnameの変換後*

タグで分岐したあとに（4行め）、入っているはずのデータを取り出して（5行めと10行め）、変数に束縛してます（6行めと11～12行め）。実際のコンパイラはほかにもいろいろな変換をはさむので、もう少し奇怪な出力になるでしょうが、概ねこのような素直な変換が考えられます。

さて、この変換が正しいかどうかを考えてみましょう。C言語の`switch` 文は、該当するラベルに飛ぶような動作をします。一方、SMLのパターンマッチは、マッチした節の中で一番上にある節を選びます。そのようなSMLのパターンマッチの挙動をC言語の `switch` 文へと素直に変換しても問題ないのでしょうか。

このことは次の2つの事実から確かめられます。

まず、 $\mathit{match}$ と $\|$ の挙動を思い出しましょう。「対象の値と複数並んだ節のすべてのパターンを比較し、マッチすれば腕の式に、マッチしなければ$\mathit{FAIL}$に置き換える」というのが、パターンマッチの意味なのでした。

次に、「データ型の値には同時にAでありかつBであるような値は存在しない」という事実を思い出してください。このことから、マッチする節は1つだけで、それ以外の節は$\mathit{FAIL}$に変換されていることがわかります。この、唯一マッチする節の腕を選んであげればよいので、 `switch` で複数のラベルのうち該当するただ1つのラベルに飛べばそれで問題ない、といえます。パターンマッチの意味を抽象化したことが、ここで役に立っています。


## 複雑なパターンマッチのコンパイル

シンプルなパターンマッチの場合は、素直な`switch`に変換できることがわかりました。次はネストのあるパターンマッチの変換を見てみましょう。
以前 `isSequence` 関数とその関連データ型を以下のように定義しました。

```sml:リスト6.14
datatype stone = Black | White

datatype cell = Empty | Full of stone

fun isSequence s = case s of
                       (Full Black, Full Black, Full Black) => true
                    |  (Full White, Full White, Full White) => true
                    | _ => false
```
*リスト4.12の再掲*


このネストしたパターンを、まずはシンプルなパターンマッチのネストに変換します。その結果は、次のようなネストした `case` 式です。

```sml:リスト6.15
fun isSequence s = case s of
    (tmp_1, tmp_2, tmp_3) => case tmp_1 of
      Full tmp_4 => (case tmp_4 of
          Black => (case tmp_2 of
              Full tmp_5 => (case tmp_5 of
                  Black => (case tmp_3 of
                      Full tmp_6 => (case tmp_6 of
                          Black => true
                        | Whice => false
                      )
                      Empty => false
                  )
                | White => false
              )
            | Empty => false
          )
        | White => (case tmp_2 of
            Full tmp_7 => (case tmp_7 of
                Black => false
              | White => (case tmp_3 of
                  Full tmp_8=> (case tmp_8 of
                     Black => false
                   | White => true
                  )
                | Empty => false
              )
            )
          | Empty => false
        ))
    | Empty => false
```
*isSequenceの中間表現*

パターンマッチで書いた元のコードに比べると複雑ですが、それぞれの構成要素はシンプルなパターンと式だけなので、データ型的には簡潔です。

これを `printPathname` の場合と同じく、C言語へとコンパイルすることを考えます。特に目新しい点はないので、単なる作業として素直に変換をしてみると、次のような結果になります（実際のコンパイラによる変換ではさまざまな処理が入り、変換アルゴリズムによる相違もあるので、常にこのとおりの結果になるというわけではありません）。

```c:リスト6.16
enum stone {
  Black,
  White
};

enum cell_tag {
  Empty,
  Full
};

struct cell {
  enum cell_tag tag;
  union {
    enum stone *full;
  } data;
};

struct tuple_cell_cell_cell {
  struct cell *_1;
  struct cell *_2;
  struct cell *_3;
};

sml_bool
isSequence(struct tuple_cell_cell_cell *s)
{
  struct cell *tmp_1 = s->_1;
  struct cell *tmp_2 = s->_2;
  struct cell *tmp_3 = s->_3;

  switch(tmp_1->tag) {
    case Full: {
      enum stone *tmp_4 = tmp_1->data.full;
      switch (*tmp_4) {
        case Black: {
          switch(tmp_2->tag) {
            case Full: {
              enum stone *tmp_5 = tmp_2->data.full;
              switch(*tmp_5) {
                case Black: {
                  switch(tmp_3->tag) {
                    case Full: {
                      enum stone *tmp_6 = tmp_3->data.full;
                      switch(*tmp_6) {
                        case Black: {
                          return SML_TRUE;
                        }
                        case White: {
                          return SML_FALSE;
                        }
                      }
                    }
                    case Empty: {
                      return SML_FALSE;
                    }
                  }
                }
                case White: {
                  return SML_FALSE;
                }
              }
            }
            case Empty: {
              return SML_FALSE;
            }
          }
        }
      case White: {
          switch(tmp_2->tag) {
            case Full: {
              enum stone *tmp_7 = tmp_2->data.full;
              switch(*tmp_7) {
                case Black: {
                  return SML_FALSE;
                }
                case White: {
                  switch(tmp_3->tag) {
                    case Full: {
                      enum stone *tmp_8 = tmp_3->data.full;
                      switch(*tmp_8) {
                        case Black: {
                          return SML_TRUE;
                        }
                        case White: {
                          return SML_FALSE;
                        }
                      }
                    }
                    case Empty: {
                      return SML_FALSE;
                    }
                  }
                }
              }
            }
            case Empty: {
              return SML_FALSE;
            }
          }
        }
      }
    }
  case Empty: {
      return SML_FALSE;
    }
  }
}
```
*isSequenceの変換後*


もともと数行で書かれていたSMLのプログラムが、100行ほどのC言語のコードに展開されました。パターンマッチの表現力が改めて実感できたかと思います。

## `switch` 文か `if` 文か

リスト6.13の例では、タグが `File` と `Directory` の2通りしかなかったので「case文ではなくif文でいいのでは」と思った方がいるかもしれません。
リスト6.16のような複雑にネストしたパターンマッチの変換も、一つひとつif文で検査していくほうが簡単に実装できそうに思えるかもしれません。実際、そのように実装することも可能です。しかし、それでも可能な限りswitch文を使ったほうがよい理由があります。

if文で実装した場合の `isSequence` の変換結果をリスト6.17に示します。

```c:リスト6.17
boolean
isSequence(struct tuple_cell_cell_cell *s)
{
  if(s->_1.tag == Full && s->_1.data->full == Black &&
     s->_2.tag == Full && s->_2.data->full == Black &&
     s->_3.tag == Full && s->_3.data->full == Black) {
    return True;
  } else if(s->_1.tag == Full && s->_1.data->full == White &&
            s->_2.tag == Full && s->_2.data->full == White &&
            s->_3.tag == Full && s->_3.data->full == White) {
    return True;
  } else {
    return False;
  }
}
```
*if文による `isSequence` の変換結果*


switch文による変換との見た目でわかりやすい違いは、比較の回数です。`(Full White, Full Black, Empty)` というデータだった場合、switch文によるのネストの場合は、 `Full` 、 `White` 、 `Full` 、 `Black` と4回比較した時点でリターンします。しかし、リスト6.17のようなif文の場合は、 `Full` 、 `White` 、 次のif文に移って `Full` 、 `White` 、 `Full` 、 `Black` 、さらに最後のelse節に入ってリターンするので、都合6回比較することになります。うまくネストした比較に落としたほうが比較回数が少なくなるのです。

また、コンストラクタが多い場合は、if文よりもswitch文のほうが高速になります。多少人工的な例ではありますが、 `isWeekend` の例で説明しましょう。

```sml
fun isWeekend w = case w of
                      Sunday => true
                    | Monday => false
                    | Tuesday => false
                    | Wednesday => false
                    | Thursday => false
                    | Friday => false
                    | Saturday => true

```

これは、switch文を使うと以下のような変換結果になります。

```c
switch(w) {
  case Sunday:    return SML_TRUE;
  case Monday:    return SML_FALSE;
  case Tuesday:   return SML_FALSE;
  case Wednesday: return SML_FALSE;
  case Thursday:  return SML_FALSE;
  case Friday:    return SML_FALSE;
  case Saturday:  return SML_TRUE;
}
```

一方、if文だとこうなります。

```c
if (w == Sunday)         return SML_TRUE;
else if (w == Monday)    return SML_FALSE;
else if (w == Tuesday)   return SML_FALSE;
else if (w == Wednesday) return SML_FALSE;
else if (w == Thursday)  return SML_FALSE;
else if (w == Friday)    return SML_FALSE;
else if (w == Saturday)  return SML_TRUE;
```

if文のほうは、もし `w` が `Saturday` だった場合、7回の比較が必要なことが見てとれるでしょう。switch文のほうは7回も比較しません。それどころか、1回も比較せずに実行すべきコードを見つけてしまいます。

少し機械語の知識が必要になりますが、上記の2種類の文をコンパイルして生成される機械語を見比べてみましょう。リスト6.16とリスト6.17のC言語ふうのコードは、 `enum`の定義を補い、関数定義構文で囲ってあげれば、C言語のコードとしてコンパイルが通ります。switch文で書いたほうはリスト6.18、if文で書いたほうはリスト6.19のようなコンパイル結果になります。リスト6.18の12行めに注目すると、`w` が `Saturday` の場合には実質的に`.L6`へのジャンプへと変換されていることがわかります（図6.20）。


```asm:リスト6.18
; isWeekend(week):
  push rbp
  mov rbp, rsp
  mov DWORD PTR [rbp-4], edi
  mov eax, DWORD PTR [rbp-4]
; 入力が6以下であることを確認
  cmp eax, 6
  ja .L2
  mov eax, eax
; 6以下なら、下記のテーブル(.L4)の
; w番めに入っているアドレスにジャンプ
  mov rax, QWORD PTR .L4[0+rax*8]
  jmp rax
; アドレステーブル
.L4:
  .quad .L3
  .quad .L5
  .quad .L6
  .quad .L7
  .quad .L8
  .quad .L9
  .quad .L10
; switchのそれぞれの節
.L3:
  mov eax, 1
  jmp .L1
.L5:
  mov eax, 0
  jmp .L1
.L6:
  mov eax, 0
  jmp .L1
.L7:
  mov eax, 0
  jmp .L1
.L8:
  mov eax, 0
  jmp .L1
.L9:
  mov eax, 0
  jmp .L1
.L10:
  mov eax, 1
  jmp .L1
; 関数から戻る。
; .L2は入力が6以上である場合
; .L1は正常に処理された場合に使われる
.L2:
.L1:
  pop rbp
  ret
```
*switch文の結果*


```asm:リスト6.19
;isWeekend(week):
  push rbp
  mov rbp, rsp
  mov DWORD PTR [rbp-4], edi
; 入力を0と比較して
  cmp DWORD PTR [rbp-4], 0
  ; 異なれば次 (.L2)
  jne .L2
  ; 同じなら値を返す
  mov eax, 1
  jmp .L1
.L2:
; 入力を1と比較して
  cmp DWORD PTR [rbp-4], 1
  ; 異なれば次 (.L4)
  jne .L4
  ; 同じなら値を返す
  mov eax, 0
  jmp .L1
.L4:
; 入力を2と比較して
  cmp DWORD PTR [rbp-4], 2
  ; 異なれば次 (.L5)
  jne .L5
  ; 同じなら値を返す
  mov eax, 0
  jmp .L1
.L5:
; 入力を3と比較して
  cmp DWORD PTR [rbp-4], 3
  ; 異なれば次 (.L6)
  jne .L6
  ; 同じなら値を返す
  mov eax, 0
  jmp .L1
.L6:
; 入力を4と比較して
  cmp DWORD PTR [rbp-4], 4
  ; 異なれば次 (.L7)
  jne .L7
  ; 同じなら値を返す
  mov eax, 0
  jmp .L1
.L7:
; 入力を5と比較して
  cmp DWORD PTR [rbp-4], 5
  ; 異なれば次 (.L8)
  jne .L8
  ; 同じなら値を返す
  mov eax, 0
  jmp .L1
.L8:
; 入力を6と比較して
  cmp DWORD PTR [rbp-4], 6
  ; 異なれば次 (.L9)
  ; 最後のif文にelse節がないので
  ; 次にいくと値が不定になる
  jne .L9
  ; 同じなら値を返す
  mov eax, 1
  jmp .L1
.L9:
.L1:
  pop rbp
  ret
```
*if文の結果*


![図6.20 switch文のコンパイル結果はジャンプになる](/images/compiling_pattern_matching/switchasm.png)
*図6.20 switch文のコンパイル結果はジャンプになる*

switch文のほうはジャンプテーブルを作っているので、 `enum week` のどんな値がきても、 `L4 + rax*8` へ1回、 `L1` へ1回の計2回しかジャンプしません。一方、if文のほうは、もし `Saturday` がきたら `L2` 、 `L4` 、 `L5` 、 `L6` 、 `L7` 、 `L8` 、 `L1` と7回ジャンプすることになってしまいます。switch文を使ったほうが効率的な機械語が生成されるのです。

しかし、先ほどのswitch文による例でも、実はSMLからC言語へのコンパイラが知っているはずの情報が一部抜け落ちてしまっています。具体的には、SMLのコンパイラはタグが網羅的であることを知っているはずなので、最初の値が6以下であるかどうかの確認は不要です。この情報は、GCC拡張の `__builtin_unreachable` を使うことで、C言語のコンパイラへと伝えることができます。

リスト6.21に、`default` 節に `__builtin_unreachable` を置いた例を示します。

```c:リスト6.21
switch(w) {
case Sunday:    return 1;
case Monday:    return 0;
case Tuesday:   return 0;
case Wednesday: return 0;
case Thursday:  return 0;
case Friday:    return 0;
case Saturday:  return 1;
default: __builtin_unreachable();
}
```
*GCC拡張で網羅性を伝える*


リスト6.21を同様にしてコンパイルすると、リスト6.22のような結果になります。


```asm:リスト6.22
isWeekend:
  push rbp
  mov rbp, rsp
  mov DWORD PTR [rbp-4], edi
  mov eax, DWORD PTR [rbp-4]
; いきなりテーブルにジャンプ
  mov rax, QWORD PTR .L4[0+rax*8]
  jmp rax
.L4:
  .quad .L10
  .quad .L9
  .quad .L8
  .quad .L7
  .quad .L6
  .quad .L5
  .quad .L3
.L10:
  mov eax, 1
  jmp .L11
.L9:
  mov eax, 0
  jmp .L11
.L8:
  mov eax, 0
  jmp .L11
.L7:
  mov eax, 0
  jmp .L11
.L6:
  mov eax, 0
  jmp .L11
.L5:
  mov eax, 0
  jmp .L11
.L3:
  mov eax, 1
.L11:
  pop rbp
  ret
```
*リスト6.22のコンパイル結果*


先頭にあった、「値が6以下である」旨の検査が消えました。もちろん、これくらいの式であれば、if文で書いてもC言語のコンパイラの最適化オプションをオンにすればジャンプすらなくなります。しかし、実際に扱うパターンマッチはもっと複雑でしょうから、おそらくその最適化に頼っていては目的のコードが生成されないでしょう。

このように、ある言語のコンパイラを書くときは、もともと知っていた情報をできる限り漏らさず次の言語へと伝えるのが良いコードを生成するコツです。しかしいま見たとおり、多彩な挙動を寛容するC言語では情報の漏れが起きやすいという現実があるので注意が必要です。上記のコードも、C言語の範囲を超えた処理系の拡張を用いてようやく意図したコードになりました。なお、LLVM IRであれば、switch命令のデフォルトの飛び先ブロックの命令を `unreachable` にするなどで同様のことを実現できます。

