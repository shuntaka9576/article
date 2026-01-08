---
title: "Rust(axum) on Lambda × Aurora DSQL × Next.js on Vercelで個人ブログをリーアーキした話"
type: "tech"
category: []
description: "2020年にNode.js on Lambda × DynamoDB × Next.js on Vercelで運用していたブログのアーキテクチャを刷新しました！"
publish: true
---

## はじめに

年末重い腰をあげて、運用中のこのブログ shuntaka.dev のアーキテクチャを刷新しました

旧構成(2020年に作成)
![2020年shuntaka.dev構成](https://res.cloudinary.com/dkerzyk09/image/upload/v1615111924/blog/01ezsr2jdx19bg00pgwt1rnsk6/hqdqjtntcjed43d0nnjp.webp)

新構成
![2026年shuntaka.dev構成](https://shuntaka9576.github.io/shuntaka-dev/_images/architecture.drawio.png)

主な変更点は以下の通りです。

|要素|変更前|変更後|
|---|---|---|
|フロントエンド|Next.js(Pages Router ISR)|Next.js(App Router ISR)
|サーバサイド|Lambda(TypeScript, Node.js/Single purpose Lambda)|Lambda(Rust axum/Lambdalith/コンテナLambda/Lambda Web Adapter)
|データベース|DynamoDB|Aurora DSQL


## 旧構成のブログを作った経緯

2020年に旧構成でブログを作りました。当時個人開発者のcatnoseさんがZennをローンチし、その影響でVercelとNext.jsのPages Router、ISRに興味を持ち、個人ブログを作りました。詳しい記事は[当時のこちらの記事](https://shuntaka.dev/shuntaka/articles/01f07hctzhjcwtdq4h6ew9stk8)を読んでいただけるとわかります。

Zennはローンチ当初フロントエンドはNext.js on Vercel、サーバサイドはRails on App Engineという構成だったと記憶しています。当時サーバーレスAPIをよく作っていたのとコストが安いので、LambdaとDynamoDBでサーバーサイドを作り、Next.js on Vercelでフロントエンドを作りました。LambdaコールドスタートとISRは相性が良いと思ったというのも理由の1つです。

個人ブログは多くの場合はSSG(静的サイトジェネレーター)で済ませることが多いです。実際過去Hugoで作っていました。ただ静的サイトだとクライアントだけで完結してしまうため、遊べる庭としてはもの足りなさがありました。業務上サーバサイドエンジニアというのもあり、クラサバ分けてHTTP JSONで喋る構成にしました。


## 旧構成でつらかったこと

### マルチレポ構成

以下のように、マルチレポ構成で機能改修が億劫になりやすかったです。

* フロントエンド
* バックエンド
* [マークダウンパーサー/CSS](https://github.com/shuntaka-dev/shuntaka-dev-packages)

pnpmやturbo、renovateなども全てに設定する必要があるのも個人で開発するには手間が多いです。

※ 余談ですが、パーサーとCSSをpublicにしているのは、Zennの影響です😂 [zenn-editor](https://github.com/zenn-dev/zenn-editor)を参考に同じ構成で遊んでました。これがきっかけで、[些細な貢献](https://github.com/zenn-dev/zenn-editor/pull/528)に繋がりました。

### プライベートリポジトリ運用

publicリポジトリなら無料で使えるGitHubの機能や、SaaSサービスが多いです。ある程度育ったコードベースを安い料金でサービスを使えます。加えてpublicリポジトリならリファレンス実装としてシュッと他の人に共有できるのもメリットだと思います。

ですが当時作ることに手が一杯でフロントエンド、バックエンドのコードを公開することはできませんでした。

GitHub ActionsからAWSへデプロイするためセキュリティ的にもある程度考慮することが多いと思っていました。気にするべきことはありますが、実際にはGitHub Actionsの詳細のログまではpublicリポジトリでは見えなかったりします。

以下の画像のように、publicだったらアコーディオンで下に内容が展開されますが、privateでは見れません。
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1767870572/blog/20260108-shuntaka-blog-rearchitecture/qh85qrza9pzivhfbtiom.png)


## 新構成

ソースコードは[github.com/shuntaka9576/shuntaka-dev](https://github.com/shuntaka9576/shuntaka-dev)です。

ディレクトリ構成は以下です。

```
apps/
├── web/          # Next.js 16 フロントエンド (React 19, Tailwind CSS 4)
└── blog-api/     # Rust/Axum バックエンドAPI (SQLx, PostgreSQL/DSQL)

tools/
└── dsql-cli/     # TypeScript マイグレーションCLI (AWS DSQL対応)

iac/
└── aws/          # AWS CDK インフラ (TypeScript)

docs/             # Sphinx ドキュメント (Python/uv)
```


## 移植作業

### 方針

Claude Codeにほとんど書いてもらいました。すいません。。。。

以下のように.legacy領域に利用していたコードを全部叩き込みました。

```
└── .legacy
    ├── dynamo
    ├── shuntaka-dev-backend
    ├── shuntaka-dev-frontend
    ├── shuntaka-dev-packages
    └── specification
```

個別にどうしたか書いていきます。

### DBマイグレーション

DynamoDBからAurora DSQLへマイグレーションするに際して、まず[PostgreSQLのDB定義](https://shuntaka9576.github.io/shuntaka-dev/db/README.html)をしました。

その後[ddbrew](https://github.com/shuntaka9576/ddbrew)というサポートガバガバな自作DynamoDBをdumpするツールでバックアップを取り、生成されたjsonlファイルを指定して、Claude Codeにマイグレーションスクリプトを書かせました。 そのコードは[ここら辺](https://github.com/shuntaka9576/shuntaka-dev/tree/preview/tools/dsql-cli)です

DynamoDBのJSONをSQLのINSERT文に変換し、`tools/dsql-cli/dsl/99_seed_data.sql`として保存します。

```json:DynamoDB形式
{"articleId":"01esxf9w62kx10wfbg8888pqrp","category":[],"content":"...","createAt":1609252595889,"description":"Next.jsを使ってどのようにブログシステムを構築したのか説明します！","publishAt":1609252595889,"title":"Next.jsでブログをリニューアルしました！","type":"tech","typePublishAt":"tech-1609252595889","updateAt":1693781788131,"userId":"cE6nC9hhaPVtILROlaODKaPjUL63"}
```

```sql:DSQLのSQL
INSERT INTO app.articles (article_id, title, slug, user_id, content, thumbnail, description, status, type, published_at, created_at, updated_at) VALUES ('678b52d4-c414-40a9-80c0-54afad8cabea', 'Next.jsでブログをリニューアルしました！', '01esxf9w62kx10wfbg8888pqrp', '00000000-0000-0000-0000-000000000002', '
...

INSERT INTO app.articles (article_id, title, slug, user_id, content, thumbnail, description, status, type, published_at, created_at, updated_at) VALUES ('d6d00549-ddd3-4e91-b59d-f8ddc22f09d0', '2020年の振り返り', '01etqfnfw9h98gffzbqsv4r32w', '00000000-0000-0000-0000-000000000002', '
```

Aurora DSQLにはまともなマイグレーションツールはないのでdsqlディレクトリ配下に流すSQLを並べて、先ほどツールで逐次実行して投入しました。117レコードでデータ量は大したことないです。動作確認の過程で数十回は実行しましたが、Aurora DSQLの無料枠で余裕でした。マイグレして運用5日経ちますが今のところ無料枠内で収まっています。

```
.
├── CLAUDE.md
├── dsl
│   ├── 01_schema.sql
│   ├── 02_users.sql
│   ├── 03_tags.sql
│   ├── 04_articles.sql
│   ├── 05_articles_tags.sql
│   ├── 98_seed_data.sql
│   └── 99_seed_data.sql
├── package.json
├── src
│   ├── convert.ts
│   ├── index.ts
│   └── types.ts
└── tsconfig.json
```

### フロントエンドマイグレーション

フロントエンドは元々Next.jsなのでマイグレーションする必要はないのですが、2020年時点なのでPages RouterだったのとCSSのSassとmarkdownパーサーを別リポジトリでnpm経由で管理しており、やりすぎな部分があったので一旦CSSはNext.jsへ、markdownパーサーは後述のRustバックエンドへ移植しました。

レガシーソースの2つ(shuntaka-dev-frontend, shuntaka-dev-packages)をコンテキストに過去Figmaで自分が設計した配色(ダークモード、ライトモード)通り、Claude Codeに参照してもらい同じデザインで移植できました。全て[global.css](https://github.com/shuntaka9576/shuntaka-dev/blob/preview/apps/web/src/app/globals.css)に定義しているのは草ですが、過去の自分の設計通り移植がされ、別技術スタックを試すためのコンテキストとして使うにはまとまりがあって逆に良いかなと思っています。

```
└── .legacy
    ├── dynamo
    ├── shuntaka-dev-backend
    ├── shuntaka-dev-frontend 👈 Next.js(Pages Router ISR)
    ├── shuntaka-dev-packages 👈 マークダウンパーサー、Sass(CSS)
    └── specification
```

最初はDynamic Rendering(ピュアなSSR)になっていたっぽく、一瞬白枠が見えるくらいには遅かったです。ISRの部分はうまく移植されず、調べた結果[こちらのPR](https://github.com/shuntaka9576/shuntaka-dev/pull/11)でApp Router時代のISR対応できました。従来のPages Router ISRのときと同様の体験になりました。

GitHub AppsでGitHubのリポジトリとAurora DSQLを同期しているので、更新や新規記事の反映はSSGより早いと思います。思いたい。


### バックエンドマイグレーション

旧構成はSingle purpose Lambdaというやつで、WebフレームワークなしでAmazon APIGatewayからルーティングがあったNode.jsのLambdaが呼ばれる構成でした。

![2020年shuntaka.dev構成](https://res.cloudinary.com/dkerzyk09/image/upload/v1615111924/blog/01ezsr2jdx19bg00pgwt1rnsk6/hqdqjtntcjed43d0nnjp.webp)

一方で新構成では対照的にLambdalithという、Amazon APIGatewayに直で1つのLambdaが呼ばれる構成です。RustのWebサーバーFWのaxumのコンテナLambdaがいる状態です。

![2026年shuntaka.dev構成](https://shuntaka9576.github.io/shuntaka-dev/_images/architecture.drawio.png)

図のLWAは、Lambda Web Adapterの略で、Dockerfileに1行足せばLambda特有のリクエスト/レスポンスを変換、中継してくれるので、Lambdaのことを意識せずかつローカルで動作するaxumをそのままデプロイすることが出来ます。

移植は先ほどのレガシーコードをコンテキストにするのに加えて、[RustによるWebアプリケーション開発 設計からリリース・運用まで](https://amzn.asia/d/aOGRIlo)で学んだコードベースを先に作っておき、移植しました。

この本には感謝です。実務で必要なREST APIを作るための知識が網羅されています。発売されてすぐ買って写経して、ブログを移植しようと思い1年以上経過、やっと出来ました。

Swaggerは[api.shuntaka.dev/swagger](https://api.shuntaka.dev/swagger/)で見えますが、機能的なAPIのパスは3本です。

* 記事一覧取得
* 単体記事取得
* GitHub Appsのwebhookの受け口

![image](https://res.cloudinary.com/dkerzyk09/image/upload/v1767876999/blog/20260108-shuntaka-blog-rearchitecture/kh4fkbwhhsbimrifs9we.png)

旧構成ではクライアントからGitHub Appsをインストールしたり謎に機能をつけていましたが、断捨離して利便性を失わない最小構成を考えた結果この3本になりました。

![2026年shuntaka.dev構成](https://shuntaka9576.github.io/shuntaka-dev/_images/architecture.drawio.png)

Amazon APIGatewayは去年末機能アップデートがあり、ストリーミングは15分可能になったので、逐次のチャットボットなんかも作ろうと思えば出来ます。

マークダウンのパースはRust側で実施しています。クライアント側のCSSに合うようにclassタグをつけています。実装は[ここら辺](https://github.com/shuntaka9576/shuntaka-dev/tree/preview/apps/blog-api/markdown)です。多分Vibe味があります。改善します。すいません。。Claude Codeがsyntectを使って書き出して、tree-sitterがいいんじゃないかなーと思ったのですが、動きが不穏だったのでそちらは試しませんでした。。

Aurora DSQLはクエリビルダのsqlxで実行しています。

## その他採用した技術について

### GitHub Self-hosted Runner

axumのWebサーバーのコンパイルとコンテナイメージ作成をx86のUbuntu Runnerでビルドしたところ21分もかかりました。ARM RunnerはEnterprise、GitHub Teams、publicリポジトリでしか使えないようで、今となってはpublicリポジトリなので使えるのですが、最初はクローズドで開発していたので困りました。

![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1767877666/blog/20260108-shuntaka-blog-rearchitecture/pwfmnbotfb5rhqk8ahis.jpg)

結果タイトルの通り、Mac miniをGitHub Self-hosted Runnerとして動かしてビルドすることにしました。結果としてかなり改善しました。ARMに加えてRustのコンパイルが効いているかどうかは分かりませんが、少なくともホストは同じなのでコンテナのキャッシュはかなり効いてるっぽくソースコードに差分があった場合でも2分程度デプロイが完了するケースがありました。現在はpublicリポジトリなのでARM Runnerを使ってもいいのですが、コンテナのキャッシュは効かないのでこのままでいいかなと思っています。もちろんキャッシュできますが、ダウンロードと解凍で大して早くならない印象を持っています。

![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1767878052/blog/20260108-shuntaka-blog-rearchitecture/ynarziabtq6bd2hsbjkq.png)

### AWS CDK

今回はRoute53含めて全てコード化しました。開発環境、本番環境両方作る構成なので、ドメインは2つ必要です。完全に手動なしでいけるかと言われるとそうではなく、ACMの証明書を発行する処理でCDKがDNS検証が終わるまで進行が止まるのでRoute53のNSレコードをレジストラに設定する必要があります。詳しくは[環境構築手順](https://shuntaka9576.github.io/shuntaka-dev/01_development.html)を見れば環境が作れます。


### Renovate

マルチレポ時代に比べるとモノレポ × Renovateで幸せになりました。設定は[こちら](https://github.com/shuntaka9576/shuntaka-dev/blob/preview/renovate.json)です。セキュリティ対策としてminimumReleaseAge=21daysとGitHub Actionsはハッシュでpinしています。

### lefthook

push前に実行しています。git hook系は2重チェック感があり苦手だったのですが、turboと合わることでストレスが激減してメリットが増えます。昨今のClaude Codeとの相性は抜群で便利です。

### Vercel

Trunk-Based Developmentがしたかったのですが、ブランチ=環境の思想が強くて断念しました。

previewブランチとmainブランチの2本運用で、previewに開発用のドメイン、mainブランチにshuntaka.devを割り当てています。previewでは指定したドメインでちゃんとVercelのAuthenticationが入るので良いですね！ここら辺は安定のVercelです。

### Aurora DSQL

プロビジョニングがとても早く、JOINなどSQLはちゃんと使えるので複雑なフィルタがかける点非常に便利。バックアップはAWS Backupが使えるみたいですが、今の流量的には全部SELECTで取れば良いかなと思っています。ALTERでカラムの削除やデータ型の変更、制約の追加・削除、デフォルト値の変更など使えない部分もあります。

今回のようにスキーマが変わりにくく、データ量やリクエストもまちまちといった状態だとほぼ無料で運用できるので便利です。マネージドだと使ってなくてもコンピューティング費用がかかるのと比べたらだいぶいい時代になりました。DynamoDBだとフィルタやソートを考えて設計したり、クエリはSQLではないのでここら辺の認知負荷がないのはやはり偉いと感じました。

## さいごに

運用して大体5日経ちましたが、料金はこんな感じです。1日100pvもないので参考になりませんが...😂 Route53はhosted zoneに対して0.5ドルで、開発と本番2つで1ドルかかっています。DSQLは凡例すらない程度には課金されてないです。

![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1767879224/blog/20260108-shuntaka-blog-rearchitecture/uyaumt0kdkzcwm4rlx8t.png)

このブログは、[ソースコード](https://github.com/shuntaka9576/shuntaka-dev)と[設計ドキュメント](https://shuntaka9576.github.io/shuntaka-dev/)があるので誰でも再現可能です。公開してないコードはありません。RustとAWSでブログを作ってみたい方におすすめです。

5日ほど運用していますが、マルチレポ時代と比べると断然運用しやすくなりました。以前はRenovateも入れてなかったのでほぼ放置状態でした。現在はRenovateからのPRで、GitHub Notificationみてぽちぽちするのが楽しいですね。いずれ飽きますが、しばらく楽しめそうです。

AIコーディングエージェントのおかげで隙間時間でパーサーの拡張もしやすくなったので、今まで出来なかった拡張もどんどんしていきたいなと思います！1ヶ月くらい経ったらDSQLのメトリクスなども公開していこうかなと思います！

それでは今日はこの辺で！
