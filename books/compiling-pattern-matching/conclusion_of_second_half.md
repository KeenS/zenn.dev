---
title: 後半まとめ
---

パターンマッチのコンパイルのアルゴリズムと実装を2種類紹介しました。
それに付随して、コンパイラの全景やいくつかの中間言語も紹介しました。

バックトラック法はそれぞれの節を `switch` 文に掛けられる塊から試して、駄目だったら戻って別の候補を探すような動きをしました。
決定木法はコンストラクタで分岐した先ごとにマッチする可能性のある節をあつめてくるような動きをしました。

また、パターンマッチのコンパイルに関連する発展的話題も紹介しました。
今回紹介した話題の他にもデータ型の表現方法の工夫やアフィン変換の最適化、 `goto` や例外、関数、継続の関係など関連トピックには事欠きません。

普段は当たり前のように使っているコンパイラですが、本記事で少しでも興味を持つ方が増えたらなと思います。

# 参考文献

本稿のアルゴリズムはバックトラック法は1、決定木法は2をベースにしています。

1では節をいれかえてバックトラック法を改善する方法が紹介されています。

4および2では決定木法を最適する方法が調べられています。

今回紹介できませんでしたが、3はパターンマッチの網羅性判定、非冗長性判定を扱っています。

1. Le Fessant, Fabrice and Maranget, Luc, "Optimizing Pattern Matching," Proceedings of the Sixth ACM SIGPLAN International Conference on Functional Programming (2001), 26-37
2. Maranget, Luc, "Compiling Pattern Matching to Good Decision Trees," Proceedings of the 2008 ACM SIGPLAN Workshop on ML (2008), 35-46
3. Maranget, Luc, "Warnings for pattern matching", J. Funct (2007). Program. Vol.17, 387-421
4. Scott, Kevin and Ramsey, Norman, "When Do Match-Compilation Heuristics Matter?," 2007



