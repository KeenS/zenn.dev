---
title: "BMIを求める"
---

本章ではBMIの計算を通じて簡単なIdrisのプログラムに慣れ親しみます。

# BMIの計算

BMI（Body Mass Index）とはヒトの肥満度を表わす指数です。体重（kg）を身長（m）の二乗で割って得られます。簡単に求められそうですね。BMIを計算する関数を定義してみましょう。

まずは関数の型を書きます。入力は体重（`Double`）と身長（`Double`）の2つで、出力は数値（`Double`）です。こうなりますね。

```idris
bmi : Double -> Double -> Double
```

次に関数本体です。引数を2つ取るので書き出しはこうです。

```idris
bmi : Double -> Double -> Double
bmi weight height = ?body
```

割り算は `/` でできるので素直に計算できますね。最終的にこうなります。

```idris:Bmi.idr
bmi : Double -> Double -> Double
bmi weight height = weight / height / height
```

上記関数を `Bmi.idr` に保存したとします。これをREPLに読み込んで動作を確認しましょう。コマンドラインからIdrisを起動し、 `Bmi.idr` を引数に与えます。

```shell-session
$ idris Bmi.idr
```

すると以下のようにメッセージが表示されたあと、 `*Bmi> ` と入力が促されます。

```
$ idris Bmi.idr
     ____    __     _
    /  _/___/ /____(_)____
    / // __  / ___/ / ___/     Version 1.3.3
  _/ // /_/ / /  / (__  )      https://www.idris-lang.org/
 /___/\__,_/_/  /_/____/       Type :? for help

Idris is free software with ABSOLUTELY NO WARRANTY.
For details type :warranty.
*Bmi>
```

ここで計算をしてみます。`bmi <体重> <身長>` を入力します。身長は単位がmなので間違えないようにして下さいね。


```
*Bmi> bmi 69.5 1.71
23.767996990527003 : Double
*Bmi> bmi 52.5 1.577
21.110373476685506 : Double
```

計算できていそうですね。


# 診断をする

BMIによって体型が診断されます。機関によって様々に定義があるようですが、[日本肥満学会の定義](http://www.jasso.or.jp/data/magazine/pdf/chart_A.pdf)によると以下の表のように診断されるようです。

|        BMI         |      判定    |
|--------------------|-------------|
|         〜 ＜ 18.5 | 低体重      |
| 18.5 ≦ 〜 ＜ 25.0 | 普通体重    |
| 25.0 ≦ 〜 ＜ 30.0 | 肥満（1度） |
| 30.0 ≦ 〜 ＜ 35.0 | 肥満（2度） |
| 35.0 ≦ 〜 ＜ 40.0 | 肥満（3度） |
| 40.0 ≦ 〜         | 肥満（4度） |

この判定も実装しましょう。

実装するのは体重（`Double`）と身長（`Double`）を受け取って診断（`String`）を返す関数です。書き出しはこうなります。

```idris:Bmi.idr
diagnostic : Double -> Double -> String
diagnostic weight height = ?body
```

ここから関数本体を実装します。まずはBMIを `bmi` 関数で計算しましょう。

```idris:Bmi.idr
diagnostic : Double -> Double -> String
diagnostic weight height = bmi weight height
```

…おっと、計算した値に変数を束縛したくなりましたね。ローカル変数は `let 変数 = 式 in ...` で導入できるのでした。そのようにします。

```idris:Bmi.idr
diagnostic : Double -> Double -> String
diagnostic weight height =
  let index = bmi weight height in
  ?body
```

`?body` を埋めましょう。 大小比較は `<` 、分岐は `if ... then ... else ...` があるので以下のように書けます。

```idris:Bmi.idr
diagnostic : Double -> Double -> String
diagnostic weight height =
  let index = bmi weight height in
  if index < 18.5
  then "Underweight"
  else if index < 25.0
  then "Nomal range"
  else if index < 30.0
  then "Pre-obese"
  else if index < 35.0
  then "Obese class I"
  else if index < 40.0
  then "Obese class II"
  else "Obese class III"
```

:::message
Idrisの文字列と日本語の相性がよくないようなので英語で診断を出します
:::


これも試してみましょう。REPLに `Bmi.idr` をリロードします。REPLには `:` ではじまるコマンドでいくつか命令を書けます。リロードは `:reload` と書くとリロードしてくれます。

```
*Bmi> :reload
Type checking ./Bmi.idr
*Bmi> 
```

正しく書けていれば上記のように表示されるはずです。エラーになったらエラーメッセージを読んで修正しましょう。


さて、正しくリロードできていれば診断できるはずです。試してみましょう。

```
*Bmi> diagnostic 63 1.71
"Nomal range" : String
*Bmi> diagnostic 100 1.68
"Obese class II" : String
*Bmi> diagnostic 40 1.58
"Underweight" : String
```

よさそうですね。これでIdrisでプログラムを書いた実績を解除しました！

# 本章のまとめ

本章ではBMIの計算を通じて関数定義、数値計算やローカル変数、条件分岐などを学びました。
