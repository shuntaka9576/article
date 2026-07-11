---
title: "2022年の振り返り"
type: "misc"
description: "毎年恒例のやつ書きました"
thumbnail: "https://res.cloudinary.com/dkerzyk09/image/upload/v1671863811/blog/01gmj0rnrsx2rwsqyfj2m15ymb/h7iflpmgeoq0pms6rjtv.png"
publish: true
tags:
  - "misc/振り返り"
  - "tech/go"
  - "tech/neovim"
  - "tech/登壇"
---

# はじめに

今年も残すところ約一週間となりました。お世話になった皆様感謝です。2022年の個人活動を振り返ろうと思います。

社会人も7年目になり、新卒のときチュータでこの人凄いなと思っていた先輩と同じくらいのキャリア()になったことを考え軽く...になりましたが元気です。

また今年は特に**吉野家**と**ぎょうざの満州**にお世話になりました。多くのカロリーをご提供頂きこの場を借りて重ねて感謝申し上げます。

# 今年の活動(技術)

## 個人ブログ

個人ブログ自体の改善は、Next12、React18、GA4対応くらい。

1月-4月末のPV数は約800PV/月程度。
![img](https://user-images.githubusercontent.com/12817245/209315474-4057648c-7560-47cd-9298-f35e1b6b2b1d.png)

4月末にUAからGA4へ切り替えた。約300程度。UAとGA4だとメトリクスの概念が違うので、繋げても参考にならないと思います。差がすごい、なんでだろう(思考放棄)
![img](https://user-images.githubusercontent.com/12817245/209314560-508f1126-767d-424d-927e-576fab141bac.png)


記事は3つしか書いていないので、2020, 2021資産がPVに寄与した感じ。Go Conferenceは1年ぶりに参加したが、色んなトピックの話が聞けて参考になった。詳しくは記事を見てね。

### Zig

一時期話題なったので、始めてみた。ziglingを少しやって放置している。サチコンみててもzigが話題になったタイミングでの検索流入が予想より多かった。

### 記事まとめ

|記事タイトル|
|---|
|[新しいJavaScriptランタイムBunとその開発言語Zigの開発環境を作る](https://shuntaka.dev/shuntaka/articles/01g7kk2jy54b9d7ct876mexwkm)|
|[Go Conference 2022 Spring参加レポート](https://shuntaka.dev/shuntaka/articles/01g19p2b2eg3dmjfzj4qg7cywp)|
|[SPAにTwitter OAuth 2.0を組み込む際の雑メモ](https://shuntaka.dev/shuntaka/articles/01fvpj77522jpcagxsget1ejqr)|

以上、素振りネタとセッションレポートと振り返りを書くところになりつつある個人ブログでした！

## Zenn

今年はどちらかというとZennへの投稿が多かった。圧倒的に読まれるので。Recapメールを貼ってみる。

![img](https://github.com/shuntaka9576/zenn-content/assets/12817245/6088993d-5f63-449a-8646-b8a8e05e1f08)
![img](https://user-images.githubusercontent.com/12817245/209311835-534a7b86-d049-466a-8bc3-b8de85930912.png)



### Litestream

特に面白かったのはLitestream on App Runner構成、大体月1000円くらいで可用性とSQLを捨てずにWebアプリが作れる構成。SQLiteとはいえ基本的なクエリはもちろんWindow関数もあるので、個人でやりたいサービスがあればプロトタイピングにピッタリだと思う。業務で本構成を取ることはないので、趣味アーキテクチャな気はする。記事の構成で個人用に運動記録アプリみたいなのを作ってみたりした。

### Cloudflare

昨今パフォーマンスを求められたとき、CDN Edgeを使うという選択肢があり、私の業務的にはここまで必要にならないことは多い。が、このロジックはCDN Edge配信できるなーとか考える機会が増えたので、素振りする価値はあるなという所感。パフォーマンスに悩んだ際に、切れるカードの1つ。

### GitHub CLI(プラグイン)

GitHubの便利ツールは全部GitHub CLIのプラグインでいいなとなった。理由は権限周りの取り回しをGitHub CLI側に委譲できる。PATの管理機構を丸っとサボれる。ただGitHub CLIのラッパーになるので、パフォーマンス重視の場合はAPI直で叩いた方が良い。

### Tauri

スクラップにしか書いてないが、面白かったフレームワークなのであえて書いておきたい。TauriというRust製のデスクトップフレームワークがあり、似たフレームワークとしてelectronがある。結構成熟していて、試していて楽しかった。デクストップアプリの強い領域はあるので、ネタがあれば何かアプリを作ってもいいかもしれない。


### 記事まとめ

|記事タイトル|
|---|
|[GitHub Projectsへ簡単にissueを作成、追加するGitHub CLI拡張を作った](https://zenn.dev/shuntaka/articles/edfd9ce2c0ee52)|
|[Litestream on App RunnerでS3にSQLiteをレプリケートしつつアプリをホスティングする](https://zenn.dev/shuntaka/articles/9d35c4710a4b29)|
|[DynamoDBの個人的便利ツール(CLI)を作った話](https://zenn.dev/shuntaka/articles/68072c00f54700)|
|[AWSのクレデンシャルをローテートしつつ、LitestreamでSQLite3をS3にレプリケートする](https://zenn.dev/shuntaka/articles/8dcfd60d5f21ec)|
|[JestでTwitter OAuth 2.0の認可コード取得処理を自動化する](https://zenn.dev/shuntaka/articles/5b9ea7521c7522c5ab0b)|
|[\[小ネタ\]WebアプリのセッションIDに情報をセキュアに持たせるライブラリ](https://zenn.dev/shuntaka/articles/31f4c228de920b1776e8)|
|[Cloudflare Pages(Functions,KV)を使ったサイトをホスティングする際に行ったこと](https://zenn.dev/shuntaka/articles/db30a06a59a0f98150cc)|
|[スクラップ](https://zenn.dev/dashboard/scraps)|

## その他

### PKCS#11 with AWS IoT Core

ATECC608の外部から読み出せないスロットの秘密鍵でAWS IoT CoreとMQTT通信するのが去年末出来なかったのができるようになっていた。もう少し掘り下げたかったものの、Cを書きまくる必要があり、自然と優先度が下がっていた。

### 記事まとめ

|記事タイトル|
|---|
|[AWS IoTを使ったIoTエッジソフト開発(Python on Raspberry Pi OS)でやって良かったこと](https://dev.classmethod.jp/articles/shuntaka-aws-iot-with-rpi-develop-tips/)
|[\[reTerminal\] PKCS#11を用いて秘密鍵を安全に管理しつつAWS IoTとMQTT通信する](https://dev.classmethod.jp/articles/shuntaka-reterminal-pkcs11/)
|[Kinesis Data StreamsとLambdaのイベントフィルタリング機能を利用して、条件に応じたLambdaを起動する](https://dev.classmethod.jp/articles/shuntaka-event-filtering-lambda-and-kinesis/)
|[Lambdaから別のアカウントのLambdaを呼び出すCDK構成](https://dev.classmethod.jp/articles/shuntaka-cdk-cross-account-invoke-lambda/)


### 登壇

Kyoto.go remote #32 LT会で登壇した。登壇内容は後述します。同じLT会でプロトコル開発に関するLTがあり、自分もPing,ARP,UDPあたりを発表者様のブログを参考にGoで実装した。プロトコルスタックやRFCが少し怖くなくなったのでとても良かった。TCPで挫折したけどね！

|登壇|
|---|
|[GoでDynamoDBのCLI(便利)を作った](https://docs.google.com/presentation/d/1wNYydZQckmyKrU0NgwNlUc2vn6V0TuGEaUh1PGkWZyo/edit#slide=id.p)|


## 作ったツール

今年趣味で作ったツールについて

### [ddbrew](https://github.com/shuntaka9576/ddbrew)

DynamoDBのバックアップ、リストアツール。業務でDynamoDBのマイグレーションやゴミデータの削除が多いので作った。ワーカープールを作って並列書き込みするという内容は、Goととても相性が良かった。なによりもgoroutineやチャネルを活用してCLIを作るのは楽しい。

### [gh-p2](https://github.com/shuntaka9576/gh-p2)

GitHub Projects V2のカード(Issue)作る君。これも業務で必要なので作った。V1のときよりできることが増えていて、カンバンのレーンに直接カードを配置できたり、Pointの設定もAPI経由で出来る。業務で毎週使っていて、来年も使うことになりそう。既に宣伝済みだけど、機能増やしてもっと便利にしてより広めてもいいかも。

### [preview-hozi-dev](https://github.com/shuntaka9576/preview-hozi-dev)

汎用的なツールではなく、個人ブログのプレビューツール。なのであまりここに書くのは適切ではない、、記事の水増し。過去にNeovimでmsgpackrpcの仕組みを使ったリモートプラグインという仕組みで作っていたのをdenopsベースにした。理由は単にリモートプラグイン版が動かなくなり、保守も面倒になったため。

denoはnpmパッケージをサポートしているため、自作のnpmマークダウンパーサーをそのまま利用した。最高に便利。サーバー側でパースしてHTMLにして、Web側へWebsocketで送るというこの手のツールではよくある方法。denoが成熟してきて来年は積極的に使っていきたい。

swaggerとasciidocのプレビューワーをリモートプラグイン作っているので、全てdenopsへ移行する足がかりになった。来年気が向いたらやる。

## おしごと

10月一杯で3年くらいやっていたIoTの案件が終わり、11月からは普通のWebアプリのサーバーサイドやっている。Fargate,Aurora,Nest,Prismaあたりを触っている。来年はここら辺の発信が増えるかも(?)。とはいえ不確定要素が多く、全然違うことやっている可能性もある。

## 読んで良かった本

* 詳解Go言語Webアプリケーション開発
  * 実践的で良かった。サンプルを応用してsqlite with sqlcでAppRunnerで動くアプリを作ったりしてみた
* 実用 Go言語 ―システム開発の現場で知っておきたいアドバイス
  * Goを書く上での迷いみたいなのが結構解消される気がしている。Goに慣れて来た人におすすめ
* MySQL徹底入門 第4版 MySQL 8.0対応
  * FULL TEXT INDEX周り詳しく書かれていて良かった。

# 今年の活動(技術以外)

## 健康面

運動の習慣がついた。今年の後半から週最低1回はジムに行って運動している。バイク漕ぐか軽いウェイトトレーニングのみではあるものの、体調は良くなった。書いている時点で体重が約60kgで体脂肪率約17%なので今年中に15%きりたい。来年も早起きと週3以上の運動は維持したい。


# 振り返り
## 振り返りの振り返り

> `shuntaka.dev`は他媒体と比較しても一番使いやすいので、引き続き記事を書いていく


3記事書けたので満足。2022年はブログを作ったばかりで住み分けが曖昧だったけど、素振り、セッションレポート、振り返りネタを書いていくスタイルに落ち着きそう。

> * 技術の素振り場所を増やす

今年はスクラップをそれなりに書いた気がする。今年スクラップに書いたネタは来年記事になるはず

> * 個人ブログとは別にWebアプリを1つを作り、運用する
>    * WASM, Deno, CDN関連Saas(Cloudflare Workers, Workers KVとか)あたりが組み込めるテーマがいいな..

DenoやCloudflareは今年試せたので、書いた当初より理解度が上がった結果、Denoは刺さったもののCloudflareはそこまで頑張らなくていいかなと落ち着いた。採用できる機会少ないし。

> * M5Stack(ESP32, Arduino)を使って何かアプリを作る(ｽﾀｯｸﾁｬﾝとか..)

Cが必要で、1度は業務をこなさないと時間ばかりかかってしまうので断念。

> * `shuntaka.dev`の採用技術の見直し、素振り
>   * 例えば、VercelをAmplifyにする、DynamoDB, Firebase Authenticationをsupabaseにする等

Fargate Aurora構成にしたい気持ちがあるものの、手軽さ、金銭面の問題がネックで辞めた。

## 2023年に向けて

迷走しまくるとは思うけど、とりあえず書いてみた。

**Neovimの設定ファイル改善とプラグイン開発(deno, VimScript, Lua)**
気が向いたらやるみたいなスタイルだと、思いだすコストが高いので習慣にしたい。

**自作CLIの作成と改善(Go)**
UIライブラリの`bubbletea`周りの組み込み周りを慣らしたい。自作CLIは作って使われなくても知識は横展可能な割りに、熱が冷めてやらなくなるのは防ぎたい。成果物より過程に意味があるくらい(実際そうだと思っている)には割り切ってやりきるのが大事な気がしている。過程で得たことを発信してモチベーションを保つとか色々方法はありそう、模索していきたい。

**記事を書く**
Zennのスクラップを書くことが増えたので、満足せず定期的にまとめる。2021年21本(自-18,Z-0,他-3) 2022年14本(自-3,Z-7,他-4)で総数は少ないが、Zennのへ投稿が増えた。本数より質だけど、指標があると頑張れるのも事実なので、21記事個人以外の媒体で書くことを目標にしたい。

**競技プログラミング始めてもいいかも**
始めてから書け。あまりここら辺の話論理立てて説明できないので、必要な気がしている。まず書籍の`問題解決のための「アルゴリズム×数学」が基礎からしっかり身につく本`が良さそうだったのでやってから考える。zigの学習と絡めてもいいかも。

# 最後に

来年はコロナが落ち着いてリアルイベントが増えそう。埼玉の奥地だし引っ越しも必要になりそうかなぁ。とはいえやることは変わらないので引き続きという感じ。サムネイルは大阪出張で行った新大阪の串カツ屋です。個人のお店でとても美味しかったでまた行きたい。あと年末暇すぎて書きすぎた。

ということで、1年間お世話になりましたー。良いお年を〜。
