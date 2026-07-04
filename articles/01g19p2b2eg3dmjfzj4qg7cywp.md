---
title: "Go Conference 2022 Spring参加レポート"
type: "tech"
description: "Go Conference 2022 Spring参加レポートです。進行と同時に随時更新していきます。"
thumbnail: ""
publish: true
tags:
  - "tech/go"
  - "tech/イベント参加"
---



# はじめに
Goとしばらく離れていましたが、最近作りたいものができたこともあり、変化に追いつくためにもGo Conference 2022 Springに参加した。今回はリモート開催となる。イベント詳細情報は、下記の通り。

* [Go Conference 2022 Spring](https://gocon.jp/2022spring/)公式ページ
* [connpass](https://gocon.connpass.com/event/212162/)のイベント参加ページ

# スケジュール

:::message
4/24(日) AM 7:00 時点で、公開されている発表資料/スライドのリンクを更新しました
不備がありましたら、[こちら](https://github.com/hozi-dev/article/blob/master/articles/01g19p2b2eg3dmjfzj4qg7cywp.md)まで
:::

スライドのリンクが分からない箇所には(未)を入れています

|時刻|Track A|Track B|
|---|---|---|
|10:00|-|-|
|10:10|[人間の直感に対応させた複雑度<Cognitive Complexity>の計測ツールをgo/astで実装してみよう](https://gocon.jp/2022spring/sessions/a1-c/) [📄(未)]()|-|
|10:30|[創業以来のPHPシステムが生み出した混沌をGoへの移行で乗り越えた話](https://gocon.jp/2022spring/sessions/a2-c/) [📄(未)]()|-|
|11:00|[Dissecting Slices, Maps and Channels in Go](https://gocon.jp/2022spring/sessions/a3-l/) [📄(未)]() |[ゼロから作る Protocol Buffer のパーサーとレキサー](https://gocon.jp/2022spring/sessions/b3-l/) [📄](https://speakerdeck.com/yoheimuta/lexer-in-go-from-scratch)|-|
|11:40|[GoのGC(garbage collector)について理解する](https://gocon.jp/2022spring/sessions/a4-s/) [📄](https://speakerdeck.com/uji/gofalsegc-garbage-collector-nituiteli-jie-suru)|[lock free な doubly linked list を実装していたらいつのまにか concurrent skip list map を実装していたでござる](https://gocon.jp/2022spring/sessions/b4-s/) [📄](https://docs.google.com/presentation/d/1ShLO-hWIiRVGm8ZMzUOlHFuUkWMciIjvA3h3fjfEwKo)|
|13:10|[レガシーシステムをGoにリプレースした一年間の振り返り](https://gocon.jp/2022spring/sessions/a5-c/) [📄(未)]()|-|
|13:30|[メタバースを支える技術 ～UGCに溢れる3D空間のリアルタイム同期を支えるGo〜](https://gocon.jp/2022spring/sessions/a6-c/) [📄(未)]()|-|
|14:00|[高速で統一的な自動生成ツールをprotocプラグインとして実装した話](https://gocon.jp/2022spring/sessions/a7-l/) [📄(未)]()|[Go で RDB に SQL でアクセスするためのライブラリ Kra の紹介](https://gocon.jp/2022spring/sessions/b7-l/) [📄](https://drive.google.com/file/d/1lsF7q74o0Akewk-5rUb9Jt9nA2MqpDDw/view)|
|14:40|[database/sqlパッケージを理解する](https://gocon.jp/2022spring/sessions/a8-s/) [📄](https://docs.google.com/presentation/d/11jdcnZUm8pYfv6RijlvWEz8ZgK2tMfloIJyLAo-cMnI/edit) |[「自動コード生成ツール」を20分で作れるようになろう](https://gocon.jp/2022spring/sessions/b8-s/) [📄(未)]()|
|15:20|[GoでAPI クライアントの実装](https://gocon.jp/2022spring/sessions/a9-s/) [📄(未)]()|[GoらしいAPIを求める旅路 並行処理編](https://gocon.jp/2022spring/sessions/b9-s/) [📄(未)]() [🎥](https://www.youtube.com/watch?v=PwtuBnk9yjU)|
|15:40|[testingパッケージを使ったWebアプリケーションテスト（単体テストからE2Eテストまで）](https://gocon.jp/2022spring/sessions/a10-s/) [📄](https://speakerdeck.com/budougumi0617/gocon2022spring)|[HTTP Tunneling in Go](https://gocon.jp/2022spring/sessions/b10-s/) [📄(未)]()|
|16:00|[GoとLambdaを使用した高パフォーマンスでサーバレスなマイクロサービスの開発と運用](https://gocon.jp/2022spring/sessions/a11-s/) [📄](https://speakerdeck.com/saicologic/go-conference-2022-spring-gotolambdawoshi-yong-sitagao-pahuomansudesabaresuna-maikurosabisufalsekai-fa-toyun-yong)|[Go Module with Microservices and Monorepo: Clear Dependencies with Ease of Development](https://gocon.jp/2022spring/sessions/b11-s/) [📄(未)]() [🎥](https://www.youtube.com/watch?v=jEdIXzvw_Ao)|
|16:30|[型パラメータが使えるようになったのでLINQを実装してみた](https://gocon.jp/2022spring/sessions/a12-s/) [📄](https://speakerdeck.com/makki_d/xing-parametagashi-eruyouninatutafalsedelinqwoshi-zhuang-sitemita)|[IoT with TinyGo](https://gocon.jp/2022spring/sessions/b12-s/) [📄](https://docs.google.com/presentation/d/1bOu9oGluosLUhLgPrhJnzGd9IIMSnAa4--AZf2KnfYA)|
|16:50|[Let's contribute to OSS with Go](https://gocon.jp/2022spring/sessions/a13-s/) [📄(未)]()|[Go で始める将棋AI](https://gocon.jp/2022spring/sessions/b13-s/) [📄(未)]()|
|17:10|[The introduction of my way to learn Go together with Go community.](https://gocon.jp/2022spring/sessions/a14-s/) [📄](https://docs.google.com/presentation/d/1TJSGpUS3kuQu7s16GnddyzGrcH0knHGBdGLHwttN2KU)|[Motto Go Forward Goを支える文化とコミュニティ 〜なぜ我々はコミュニティにコントリビュートするのか？〜](https://gocon.jp/2022spring/sessions/b14-s/) [📄(未)]()
|17:45|[Goの標準機能で簡易システムを低コストで作成するテクニック](https://gocon.jp/2022spring/sessions/lt1/) [📄](https://speakerdeck.com/qualiarts/gofalsebiao-zhun-ji-neng-dejian-yi-sisutemuwodi-kosutodezuo-cheng-surutekunituku)|-|
|17:50|[Python製の姓名分割ライブラリをGoに移植した話](https://gocon.jp/2022spring/sessions/lt2/) [📄](https://speakerdeck.com/nagano/pythonzhi-falsexing-ming-fen-ge-raiburariwogoniyi-zhi-sitahua)|-|
|17:55|[外部コマンドの実行を含む関数のテスト](https://gocon.jp/2022spring/sessions/lt3/) [📄](https://speakerdeck.com/pongzu/wai-bu-komandofalseshi-xing-wohan-muguan-shu-falsetesuto)|-|
|18:00|[大規模ゲーム開発におけるContext活用パターン](https://gocon.jp/2022spring/sessions/lt4/) [📄(未)]()|-|
|18:05|[GoとKubernetesを用いたバッチ開発のすすめ](https://gocon.jp/2022spring/sessions/lt5/) [📄](https://speakerdeck.com/waytkheming/gotokuberneteswoyong-itabatutikai-fa-falsesusume)|-|
|18:10|[Gopher, Chrome, Automation in 5m](https://gocon.jp/2022spring/sessions/lt6/) [📄](https://docs.google.com/presentation/d/1tnYI2lJi0YO0GNGeYhM6xru5jUVOwXOeNAvuvWvoD_8)|-|
|18:15|[Go runtime の歩き方](https://gocon.jp/2022spring/sessions/lt7/) [📄](https://speakerdeck.com/convto/how-to-follow-go-runtime-function)|-|
|18:20|[Go言語仕様輪読会の開催を通じた振り返り](https://gocon.jp/2022spring/sessions/lt8/) [📄](https://speakerdeck.com/syumai/goyan-yu-shi-yang-lun-du-hui-falsekai-cui-wotong-zitazhen-rifan-ri)|-|


# セッションの感想

## 人間の直感に対応させた複雑度<Cognitive Complexity>の計測ツールをgo/astで実装してみよう

[📄(未)]()

**メモ**
* 認知的複雑度を測定する静的解析ツール [nrnrk/gocognito](https://github.com/nrnrk/gocognito)

**感想**

`Cyclomatic complexity`(循環環複雑度)は聞いたことがあったが、`Cognitive Complexity`(認知的複雑度)は知らなかった。。人間の直感近い複雑度が計測できるとのこと。
`Cognitive Complexity`を計測するため`go/ast`を使って、静的解析ツールを作るセッション。astの構造体を理解する上で参考にしたいと思いました！

## 創業以来のPHPシステムが生み出した混沌をGoへの移行で乗り越えた話

[📄(未)]()

**メモ**
* CMS側をPHPからGoへ移行した話。テスト,静的解析,クラス,関数全てなしのコードのリプレースに関するセッション
* RDBにJSONのカラム(text)があり、map[string]stringや定義した構造体にパースするため、`sqlx`の`Scanner`や`Valuer`といったインタフェースを駆使し、課題解決した

**感想**
`sqlx`でイレギュラーなRDBの値に対して、対応する手段があるのは便利だなと思いました！


## ゼロから作る Protocol Buffer のパーサーとレキサー

[📄](https://speakerdeck.com/yoheimuta/lexer-in-go-from-scratch)

**感想**
個人的にGoはパーサー実装に強いと思っています。理由として`gojq`や`esbuild`の既存実装を移植したプロジェクトも存在し、どれも本家と遜色ないどころか高パフォーマンスだったりします。
本セッションは、なんらかの理由で(Google公式のprotoパーサーだと全ての文法規則に対応していない?)Protocol Bufferのパーサーを実装するという内容。

木構造の設計の工夫などパーサーだけでなく、静的解析のツールチェインを作ることも考慮したTipsが説明されていた。VimのALEプラグイン経由で自動修正のデモもあり、かっこよかったです。自作言語のもう一歩先を行ったトピックで、自分良い題材が見つかったら参考にしたいと思いました。

go-protoparserを読むのも良さそうです(執筆時15000行くらいはあるので、全体を把握するのは大変そうですが、、)


## GoのGC(garbage collector)について理解する

[📄](https://speakerdeck.com/uji/gofalsegc-garbage-collector-nituiteli-jie-suru)

**メモ**
* discordの事例でGoを利用した結果、2分毎のGCによってSTWが起きて、レイテンシが高くなり、許容できずRustにリライトした
  * Go 1.9.2の利用事例で少し古いという指摘もあり
* GCは生産性を上げるために必要なため、GoにはGCが存在する
* GoはConcurrent Mark Sweep(CMS)を採用
  * 最古のGCアルゴリズムを改良した
* GCもバージョン毎に改善が入っている
  * GoチームでGCのSLOがある

**感想**
GoのGC理解の大一歩としてとても良いセッションと感じました。GoのGCアルゴリズムの概要から始まり、バージョン毎に改善が入っている点、GoチームでGCのSLOがある点など、個人的に知らないこと盛りだくさんでした！


## レガシーシステムをGoにリプレースした一年間の振り返り

[📄(未)]()

**感想**
DDD, Clean Architecture, Onion Architectureを利用したプロダクトの話。並列テストにも少し言及があり、参考になりました。


## メタバースを支える技術 ～UGCに溢れる3D空間のリアルタイム同期を支えるGo〜

[📄(未)]()

**感想**

メタバースなアプリケーションのサーバーサイドでのGo活用事例。UGC(ユーザー側で作成できるコンテンツ)の素材の転送効率の最適化やPub Subサーバー(ユーザーアバター、空間制御周り)でGoを利用しているとのこと。一般的なWebアプリと異なる高負荷、ステーフルな環境下であるメタバース領域でのGo活用事例でとても新鮮でした！


## 高速で統一的な自動生成ツールをprotocプラグインとして実装した話

[📄(未)]()

**メモ**
* ソーシャルゲームのGo事例。マスターデータ量が多く、コードを自動生成しないと辛いレベルの規模感。自動ジェネレートが遅く、Protocol Bufferを活用
* `goimports`をかけたときに、import文を追加する処理は、対象のライブラリが見つかるまで探索(Nまで探索)するので重い。対策として、あらかじめimportを追加しておく、`goimports`のimports削除はAST解析のみでされるので、誤ってimportしてしまうことを許容して問題ないとのこと。
* `filepath.Walk`より`filepath.Walkdir`が速い。(Go 1.16で追加)
* protcプラグインはログに標準エラー出力を使う


**感想**
自動生成コードI/O処理の極小化として、File I/Oはギリギリまで発生させないという話は分かりみが深かった。`goimports`や`gofmt`をライブラリから生成されたバイト配列に対して適用しているとのこと(面白い)。自動生成定義にprotocを使って素振りしたいなと感じました。


## GoとLambdaを使用した高パフォーマンスでサーバレスなマイクロサービスの開発と運用

[📄](https://speakerdeck.com/saicologic/go-conference-2022-spring-gotolambdawoshi-yong-sitagao-pahuomansudesabaresuna-maikurosabisufalsekai-fa-toyun-yong)

**感想**
ランタイムが1つというのは確かにメリットだなと感じました！ベンチマーク結果は、GoはやはりLambdaでも良いパフォーマンス結果が出ていました。ただ実際にアプリを書くと、Nodeは自然と非同期になる一方、Goはgoroutineを使う必要があることを考慮すると要件次第だなぁとも思いました。。

## IoT with TinyGo

[📄](https://docs.google.com/presentation/d/1bOu9oGluosLUhLgPrhJnzGd9IIMSnAa4--AZf2KnfYA)

**メモ**
* fastlyがほぼフルタイムでTinyGoの改善をしているとのこと

**感想**
goroutineも使えるのは意外だった。httpやMQTT(内部的にはpahoを利用)のpub/subが使えるので、それなりに遊びが出来る段階には来ているんだなーと感じました。Wio Terminal持っているし、どこかで試したいですね。


## 外部コマンドの実行を含む関数のテスト

[📄](https://speakerdeck.com/pongzu/wai-bu-komandofalseshi-xing-wohan-muguan-shu-falsetesuto)

**感想**

目から鱗でした。外部コマンド実行結果のモックがサクッと実装できるとテストが捗りそうですね。。標準ライブラリのテスト読んでみることの重要さにも気づけて良かったです。

## Gopher, Chrome, Automation in 5m

**感想**
私が昔Chrome自動化をする際には、[sclevine/agouti](https://github.com/sclevine/agouti)を使っていましたね。紹介されていたライブライどれも使いやすそうです。

* [mafredri/cdp](https://github.com/mafredri/cdp)
* [go-rod/rod](https://github.com/go-rod/rod)
* [chromedp/chromedp](https://github.com/chromedp/chromedp)

この手のchrome自動化ツールはスタンドアローンで起動しますが、プロファイルのcookieやlocal storageを使いたいケースがあり、そこら辺いい感じになっていると嬉しいなーと思いました！


# 最後に

幅広い分野でGoが活用されていることが再認識できました。運営のみなさまこのような機会を設けて頂きありがとうございました！
