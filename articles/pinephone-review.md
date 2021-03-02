---
title: "自由なスマホPinePhoneを試す"
emoji: "🍍"
type: "tech"
topics: ["Smartphone", "Linux", "Pine64", "PinePhone"]
published: true
---

κeenです。[PinePhone](https://www.pine64.org/pinephone/)という自由なスマホを手にしたのでそれを紹介します。

# PinePhoneについて

PinePhoneを雑に紹介するとLinuxで動くスマホです。まだまだ開発途上のものなので常用に耐えるものではありませんが、未来予想図的に夢があるよねーといって眺めていただけたら幸いです。

世に出回っているスマートフォンはメーカーやプラットフォーマの権限が強くて安くはない金額で買っているのにも係らず誰のものか分かりませんよね。ハードウェアは自分で修理するのも禁止されていればソフトウェアはアカウントを停止されるとほとんどただのカマボコ板になってしまいます。それに気付いたらGPSデータをサーバに送られていたり、不意にカメラやマイクを起動されてプライバシーがだだ漏れになるリスクもあります。PinePhoneはそういった状況を憂慮する人のためにユーザに最大限の自由を与えたハードウェアの上にオープンソースのソフトウェアを動かすようになっています。また、マイクやカメラなどはハードウェアレベルでoffにできるのでプライバシーを心配する人にも安心です。因みに作っているのは[pine64](https://pine64.org)というところで、他にもSoCやラップトップ、タブレット、スマートウォッチなども出しています。

まあ、それとは別に普通にCPUとメモリのついたデバイスを持ち歩くならLinuxが動いていてほしいよね、という素朴な感情もあります。

# 実物

私が手にしたのはComunity EditionのMobian版です。MobianというのはDebianのモバイル向けのもので、要するに普通のLinuxです。

Twitterで実況したのでそのときのツイート群を貼ります。Zennの仕様でリプライ元のツイートも表示されていますが、本体の方だけ見て下さい。

https://twitter.com/blackenedgold/status/1366305418226532353

開梱したら初手で裏蓋を開けてバッテリーの絶縁を剥がしたりmicroSDカードを挿したりする作業をします。

https://twitter.com/blackenedgold/status/1366663694067789826

このPinePhoneは日本では技適が通ってないので無線機能を使うとお巡りさんがやってきます。なので無線をoffにして使用しています。

WiFiが使えなくてもUSB Type-Cのポートがあるのでコネクタを通して有線LANを挿せばインターネットに繋がります。

https://twitter.com/blackenedgold/status/1366661949895835652

起動するとセットアップがはじまります。

https://twitter.com/blackenedgold/status/1366664127729393664

https://twitter.com/blackenedgold/status/1366666374072868867

https://twitter.com/blackenedgold/status/1366666930615058434

ツイートの間隔を見てもらったら分かるかと思うんですが、10分ほどかかります。

ハードウェアのところで触れますが、低スペックなSoCがベースなので動作は遅いです。スクロールするときも指の動きについてこれないので割と操作しづらいです。

https://twitter.com/blackenedgold/status/1366672816804818953

PinePhoneの大きな特徴の1つにConvergence、つまりてディスプレイやキーボードなどを繋げば普通のLinux PCとしても扱える点があります。

https://twitter.com/blackenedgold/status/1366673379235794945

https://twitter.com/blackenedgold/status/1366675647410544640

正直Linux PCどころかスマホとしてもスペック不足でかなり扱いづらいものですが、こういう面白いことのできる実機があるのはいいですよね。

# Community Editionについてとソフトウェアについて

PinePhoneは普通のLinuxカーネルで動いているらしいです。Linuxカーネルに電話回線のサポートとか入ってるんですね。なのでLinuxで動くスマートフォンの実現への下地は整っていて、あとは諸々のディストリがスマホをサポートするかと、それをインストールできる実機があるかどうかの問題です。ハードウェアとソフトウェアは鶏と卵のような関係で、ハードフェアがないとソフトウェアが開発できないしソフトウェアがないとハードフェアを出荷できません。そこで開発者に実機を届けたり、Linuxスマートフォンという分野があるということを啓蒙するためにPinePhoneが出荷されているようです。Linuxスマートフォンに取り組んでいるところはいくつかあって、Ubuntu Touchで有名な[UBPorts](https://ubports.com/foundation/ubports-foundation)だとか[postmarketOS](https://postmarketos.org)だとか色々あります。その中で私が手にしたのはDebianのモバイル版の[Mobian](https://mobian-project.org)だったという訳です。

Mobianは割と普通のDebianでした。SettingsとかSoftwareとかNautilusとか、普通にDebian（Ubuntu）でみかけるアプリが動いています。違いはデスクトップ環境とウィンドウマネージャがモバイルも意識してるな、くらいでした。UIはウィンドウサイズが小さいとスマホっぽい見た目になるだけで、アプリケーションは今までどおりっぽいです。アプリストアについても普通のDebianのパッケージがそのままのようです（SoftwareがそもそもDebianにある）。カメラや電話などいくつかの機能については専用アプリが用意されています。

プリインストールこそされていないものの、Softwareからターミナルをインストールすれば普通にターミナルが使えるはずです。ですがSoftwareが不安定でインストールする前に落ちてしまうので試せませんでした。今後の課題とします。

開発者に実機を届けるという目的がある程度達成されたのか、[Community Editionが終了する](https://www.pine64.org/2021/02/02/the-end-of-community-editions/)ことがアナウンスされました。色々なコミュニティエディションを出して感触を掴んだり資金繰りがどうこうとか色々な要因があったのだとは思いますが、現状から一歩前進できたという解釈でいいのかな？新機種のアナウンスがくるといいですね。

# ハードウェアについて

* [公式](https://www.pine64.org/pinephone/)
* [Wiki](https://wiki.pine64.org/index.php/PinePhone)

SoCはAndroidでよく使われているクァルコムのSnapdragonではなく、allwinnerの[A64](https://linux-sunxi.org/A64)です。CPUはArmのCortex A-53のクアッドコアなのでRaspberry Pi 3系と同じですね。メモリは2GBとPi 3よりは上ですがPi 4だと2GB以上のモデルがあるので全体的にPi 3より上、Pi 4未満くらいのスペック感です。普通のLinuxのGUIをそのままもってきた（Waylandで動いてるらしいです）のが影響してるのかは分かりませんがスペック不足を感じました。pine64のプロダクトで全体的に似たような構成のものが多いのでテンプレなんでしょう。それでいうとAllwinnerのA64ではなく、クアッドコアのCortex-A53に加えてデュアルコアのCortex-A72が載ったRockchipのRK3399と4GBのメモリで動くプロダクトもあるようなのでそっちベースのスマホが出るといいですね。

因みにCommunity Editionは$149.99、Convergenceのドックを含めても$199.99とスマホにしてはかなり安い値段で売られていました。


いくつか機能を挙げると

* 16GBの内蔵ストレージ
* microSDカードからブート可能
* WiFiはb/g/nまでサポート
* USB Type-Cポートから有線LANやらDisplay Portやら色々出入力できる
* イヤフォンジャックでUART通信できる
* 電話回線やWiFi/Bluetooth、マイクなどいくつかの機能はキルスイッチで物理的にオフにできる
* 色々な部品が交換可能

部品の交換可能性に触れておきましょう。

まずは裏蓋を簡単にはずせるのでバッテリーが手軽に交換可能です。SamsungのJ7の寸法に合えばどれでも使えるっぽいです。

そして基盤（SoC）も交換可能です。何を言っているんだという気になりますが、換装ガイドのYouTube動画もあります。

@[youtube](5GbMoZ_zuZs)

[裏蓋のStepファイル](https://files.pine64.org/doc/PinePhone/PinePhone%20Back%20Cover%20ver%200.5.stp)も公開されているので3Dプリンタなんかで印刷すれば自作のものを作れます。

フロントのガラスも交換できた気がしますが、手順を再度探したら見当りませんでした。

他には実際に使う人がいるか分かりませんが、裏蓋を外した中にPogoピンがあるのでデバイスを拡張することも可能なようです。

![Pogoピンの画像](https://storage.googleapis.com/zenn-user-upload/ww02392x4szuag176t6onxixt77j)

裏蓋のStepファイルが公開されていることも含めて上手くやってといったところなんでしょうか。一応、試作品を作った人はいるみたいです。

https://blog.brixit.nl/making-a-backcover-extension-for-the-pinephone/


ユーザに自由があるのはいいですね。

# まとめ

自由なスマホPinePhoneを試しました。今後に期待する面もあったものの、ちゃんとLinuxで動くスマホというのを実現しているという意味では良いプロダクトなんじゃないでしょうか。Linuxの動きとして一番気になるのは電話回線ですが、技適未承認の機器で電話回線に繋ぐのはまだ制度が整っていないようなので見送りになります。WiFiやBluetoothは届出をすれば技適未承認でも使えますが、普通のデスクトップPCでも使っているのであんまり面白くはないですね。

次のステップはアプリを作ってみるですかね。
