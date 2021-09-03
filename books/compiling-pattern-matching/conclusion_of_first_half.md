---
title: 前半まとめと次回予告
---

本記事では、まずパターンマッチの基本とその裏側を、SMLからC言語へのコンパイルを通して確認しました。さらに、発展的なパターンマッチとして、SML以外の言語で導入されている拡張も見ていきました。また、設計当初はパターンマッチが入っていないCommon LispやJavaといった言語にパターンマッチを入れる例も見ました。

いうまでもありませんが、パターンマッチは本記事で紹介した言語以外にもさまざまな言語で導入されています。本記事でそれらを紹介しなかったことには一応の理由があります。

たとえば、本記事ではHaskellを説明していません。これは、Haskellのように非正格な言語では、パターンマッチに評価タイミングが絡むため、少しデリケートな議論が必要になるからです。

PrologやErlangにおけるパターンマッチも説明しませんでした。特にPrologはパターンが「逆向き」にも動き、SMLのパターンマッチとは少し違った原則で動いているようです。

パターンに特化した「パターン指向言語」として、Egison[^egison]のような言語も開発されています。これもまたSMLのパターンマッチとは違った原則で動いているようです。

[^egison]: [https://www.egison.org/](https://www.egison.org/)

「パターン」という言葉を聞いて、正規表現のことを思い浮かべた方がいるかもしれません。実は、正規表現によるパターンマッチと代数的データ型とパターンマッチは、まったく無関係というわけでもないようです。正規表現の5要素である「空」「文字」「連接」「選択」「繰り返し」のうち、繰り返し以外は代数的データ型とパターンマッチに存在します。

本記事では、あとからパターンマッチが入った言語として、動的型付言語であるCommon Lispと静的型付言語であるJavaの状況を眺めました。これ以外にもRubyやJavaScriptでパターンマッチの導入が議論されているようです[^pattern-ruby][^pattern-js]。RubyはCommon Lispとは違い、外部表現が用意されている `Hash` クラスでも継承が可能なので、継承したクラスのインスタンスがどのように振る舞うのかなど面白い側面があると思います。

[^pattern-ruby]: [https://rubykaigi.org/2019/presentations/k_tsj.html#apr18](https://rubykaigi.org/2019/presentations/k_tsj.html#apr18) （書籍掲載時にはRubyのパターンマッチに関する記事と同時に刊行されました。なお、現在は既にRubyにパターンマッチは入っています。）
[^pattern-js]: [https://github.com/tc39/proposal-pattern-matching](https://github.com/tc39/proposal-pattern-matching)

最後に、今後の予告をします。本記事の最終的なゴールは、パターンマッチのコンパイルのアルゴリズムそのものを紹介することでした。次の記事では、SMLのパターンの部分だけを抜き出した言語から、C言語のように `switch` 文を持った言語へのコンパイラをどのように設計するかを、ソースコード付きで解説する予定です。正規表現エンジンの実装を大きく分けるとVM型とDFA型の2つがあるように、パターンマッチのコンパイルアルゴリズムにも2つの種類が存在するので、その両方を取り扱います。

可能であれば、そこから派生して、発展的なパターンのサポートやコンパイルされたパターンの最適化も紹介できたらと考えています。その過程で、Haskellのように非正格な言語におけるパターンの注意点も浮かび上がるでしょう。


# 参考文献

1.は、本稿の、特に意味の部分のベースです。P. Wadler氏寄稿の4、5章で、パターンマッチについて論じられています。本稿の変換のコード例は、2.および3.の2つの論文にある2種のアルゴリズムをベースにしています。

4.でも、パターンマッチの意味やコンパイルについて、本記事とは別のアプローチで論じられています。本記事では触れませんでしたが、日本語で読める上に直観的で十分にわかりやすい解説になっているので、本稿でこの領域に興味を持たれた方はぜひ参照してみてください。


1. Peyton Jones, Simon L., "The Implementation of Functional Programming Languages (Prentice-Hall International Series in Computer Science)," 1987
2. Le Fessant, Fabrice and Maranget, Luc, "Optimizing Pattern Matching," Proceedings of the Sixth ACM SIGPLAN International Conference on Functional Programming (2001), 26-37
3. Maranget, Luc, "Compiling Pattern Matching to Good Decision Trees," Proceedings of the 2008 ACM SIGPLAN Workshop on ML (2008), 35-46
4. 大堀 淳、纓坂 智, "表示的意味論に基づくパターンマッチングコンパイル方式の構築と実装," コンピュータ ソフトウェア Vol. 24 No. 2 (2007), 113-132
