---
title: "個人ブログのサーバサイドをAurora DSQLからTiDB Self-Managedへ移行した"
description: "個人ブログのデータベースをAurora DSQLからMini PC 3台で動かすTiDB Self-Managedへ移行。LambdaからTailscale経由で接続する構成、レイテンシ、運用費を実測値とともにまとめます。"
publish: true
tags:
  - "tech/ブログ"
  - "tech/tidb"
  - "tech/kubernetes"
  - "tech/aws/lambda"
  - "tech/tailscale"
---

## はじめに

このブログ shuntaka.dev は、2020年にNext.js + Lambda + DynamoDBの構成で作りました。その後、2026年1月にフロントエンドをNext.jsのApp Router、サーバサイドをRust(axum) on Lambda、データベースをAurora DSQLへリニューアルしています。過去の構成や移行の経緯は、以下の記事にまとめています。

https://shuntaka.dev/shuntaka/articles/01f07hctzhjcwtdq4h6ew9stk8

以下の記事はほぼOpus 4.6で驚くほど早くリアーキテクチャできました。定性的な感覚で恐縮ですが、1か月くらいかかりそうなところを1日、2日という感じです。

https://shuntaka.dev/shuntaka/articles/20260108-shuntaka-blog-rearchitecture

このリアーキテクチャでモノレポ化したこともあり、MarkdownパーサーからフロントエンドのCSSまで一気通貫でAIが走りやすくなり、メンテナンス性の次元が変わりました。

そこでAIで工数を圧縮できることから、もう少しストレッチな内容を試そうと思いました。

今回は、そのサーバサイドをさらにリニューアルし、データベースをAurora DSQLからTiDB Self-Managedへ移行しました。単にデータベースを置き換えただけではなく、Lambdaから自宅ネットワークへ到達する経路や監視も含めて作り直しています。

主な変更点は以下です。

| 要素 | 変更前 | 変更後 |
| --- | --- | --- |
| サーバサイド | Rust(axum) on Lambda | Rust(axum) on Lambda |
| データベース | Aurora DSQL | TiDB Self-Managed v8.5.7 |
| Lambdaのネットワーク | VPC外からAWSサービスへ接続 | VPC内からFargateプロキシを経由 |
| Self-Managed環境への接続 | なし | FargateからTailscale経由 |
| 検索 | なし | TiFlash + HNSWによる意味検索 |
| 監視 | CloudWatch、Aurora DSQLメトリクス | OpenTelemetry、X-Ray、CloudWatch、TiDB Dashboard、Grafana |

最終的な構成は次のようになりました。

![ブログとTiDB Self-Managedクラスタのアーキテクチャ](https://res.cloudinary.com/dkerzyk09/image/upload/v1784334412/blog/2026-07-16-selfhosted-tidb-vector-plamo-embedding-1b-hybrid-search/ah18icf4fmbaexxlrjog.png)

## Aurora DSQLからTiDBへ移行

Aurora DSQLは、小規模なこのブログではほぼ無料で運用できていました。今回の移行はコスト削減が主目的ではなく、TiDBを実運用しながら触りたかったことと、同じデータベース内でベクトル検索を完結させたかったことが大きいです。

移行は次の流れで行いました。

1. GitHub AppのWebhookを一時停止して、移行中の書き込みを止める
2. Aurora DSQLから4テーブルをTSVへエクスポートする
3. TiDB用のDDLを適用する
4. `LOAD DATA LOCAL INFILE`でTSVを投入する
5. テーブルごとの行数と`SHOW WARNINGS`を確認する
6. blog-apiをTiDB向けにデプロイする
7. APIの疎通確認後、Webhookを再開する

Aurora DSQLは`COPY ... TO STDOUT`をサポートしていないため、主キーでページングしながら`SELECT`し、PostgreSQLのTEXT形式と互換性のあるTSVを自前で生成しました。NULLは`\N`、タブや改行、バックスラッシュもPostgreSQLと同じ形式でエスケープしています。TiDB側の`LOAD DATA`も`\N`をNULLとして解釈するため、データ形式を二重に変換せず投入できました。

スキーマとアプリケーションには、PostgreSQLとMySQL互換のTiDBの差分を反映しています。

| Aurora DSQL / PostgreSQL | TiDB / MySQL |
| --- | --- |
| `UUID DEFAULT gen_random_uuid()` | `CHAR(36) DEFAULT (UUID())` |
| `TIMESTAMP WITH TIME ZONE` | UTC運用の`DATETIME(6)` |
| バインドパラメータ `$1`, `$2` | `?` |
| `sqlx::PgPool` | `sqlx::MySqlPool` |

移行ツールは開発環境と本番環境で同じDDLとロード処理を使い、データベース名だけ`blog_dev`と`blog_prd`で切り替えられるようにしました。ブログのデータ量は大きくないので、複雑なCDCを組まず、一度書き込みを止めて全件移行する方がシンプルでした。

## Mini PC 3台でTiDBクラスタを構築

過去にRaspberry Piでクラスタを組んだことがあったのですが、電源周りやARMアーキテクチャに悩まされることが多く、9割くらいは電源の記憶です...。GPIOを使うわけでもないので、今回はx86_64のMini PCを3台購入しました。

### ハードウェア

| ノード | CPU | メモリ | ストレージ |
| --- | --- | --- | --- |
| node1 | Ryzen 7 7730U（8コア / 16スレッド） | 32GB | 1TB NVMe SSD |
| node2 | Ryzen 7 7730U（8コア / 16スレッド） | 32GB | 1TB NVMe SSD |
| node3 | Ryzen 7 7730U（8コア / 16スレッド） | 32GB | 1TB NVMe SSD |

### TiDBクラスタの構成

Ubuntu Server 24.04、kubeadm、CiliumでKubernetesクラスタを作り、TiDB OperatorからPD / TiKV / TiDB / TiFlashをデプロイしています。永続ボリュームはlocal-path-provisionerを使い、各ノードのNVMe SSDへ配置しました。

| コンポーネント | レプリカ | CPU request | メモリ request / limit | ストレージ |
| --- | ---: | ---: | ---: | ---: |
| PD | 3 | 100m | 1GiB / — | 10GiB |
| TiKV | 3 | 500m | 8GiB / 12GiB | 100GiB |
| TiDB | 3 | 500m | 2GiB / — | — |
| TiFlash | 1 | 500m | 4GiB / 8GiB | 50GiB |

node1のcontrol-plane taintを外し、`topologySpreadConstraints`（`kubernetes.io/hostname`、`maxSkew: 1`）を指定しています。PD、TiKV、TiDBは各ノードへ1 Podずつ配置しています。

### Pod配置の検証

初期構築時はcontrol-plane taintが残り、データベースPodがnode2、node3へ偏っていました。node1上のsysbenchからNodePort経由で`oltp_point_select`を15秒間、128並列で実行した結果は以下です。

| 検証時の構成 | QPS | 補足 |
| --- | ---: | --- |
| TiDB 2 Pod / TiKV 3 Pod（node2、node3へ偏在） | 49,193 | p95 4.33ms |
| taintを外し、TiDBを3 Podに変更 | 41,012 | node1でsysbenchとTiDBが同居 |
| TiKVを4 Podに変更し、データ移動後に再計測 | 30,940 | node1でsysbenchとTiKVが競合し、leaderも偏在 |

Pod数とベンチマーククライアントの配置条件が変わっているため、この結果から均等配置による性能差は判断できません。現在の3ノード均等構成も、クラスタ外の専用クライアントからは再計測していません。

`topologySpreadConstraints`を採用した目的は性能向上ではなく、3ノードへの障害分散とPod配置の偏りを防ぐことです。local-path-provisionerのPVはノードに固定されるため、taintの解除と分散制約は再構築時から適用しています。

### TiFlashとベクトル検索

TiFlashはPingCAPの公式ハードウェア要件よりかなり小さい構成です。それでも10万ベクトル規模のHNSWインデックスを構築でき、検証用途ではOOMすることなく動いています。当然ながら本番向けの推奨構成ではなく、この環境で得た絶対的な性能値も参考値です。

![「酸っぱい」で検索したセマンティック検索の結果](https://res.cloudinary.com/dkerzyk09/image/upload/v1785290462/blog/20260729-shuntaka-blog-server-side-renewal/semantic-search-suppai-iphone-se.png)

ベクトル検索とPLaMo-Embedding-1BのCPU推論については、以下の記事にまとめています。日本語版Wikipediaのデータを投入した際のCPU・メモリ負荷や、入力トークン数による推論時間の違いも確認できます。

https://shuntaka.dev/shuntaka/articles/20260716-selfhosted-tidb-vector-plamo-embedding-1b-hybrid-search

https://shuntaka.dev/shuntaka/articles/20260720-selfhosted-tidb-wikipedia-100k-vectors-full-scan-vs-ann

## LambdaからTiDB Self-Managedへ接続する

Lambda自身はTailnetへ参加させず、VPC内のFargateプロキシからTailscale経由でTiDB Self-Managedが動くKubernetesへ接続しています。開発環境と本番環境でFargateタスクを共有し、データベースを`blog_dev`と`blog_prd`に分けています。

![ブログとTiDB Self-Managedクラスタのアーキテクチャ](https://res.cloudinary.com/dkerzyk09/image/upload/v1784334412/blog/2026-07-16-selfhosted-tidb-vector-plamo-embedding-1b-hybrid-search/ah18icf4fmbaexxlrjog.png)

### Fargateプロキシの構成

| 通信 | 経路 |
| --- | --- |
| DB接続 | Lambda → `tidb-proxy.internal:13306` → L4 TCP forwarder → Tailscale → TiDB |
| 埋め込み生成 | Lambda → `tidb-proxy.internal:18080` → L4 TCP forwarder → Tailscale → PLaMo |
| 外部HTTPS | Lambda → `tidb-proxy.internal:3128` → Squid → インターネット |
| テレメトリ | Lambda → `tidb-proxy.internal:4318` → ADOT Collector |

FargateタスクはARM64の0.25 vCPU / 0.5GBで、TCP forwarder、Squid、ADOT Collector、FireLensを同居させています。devとprdで1タスクを共有し、データベースとSecurity Groupで環境を分離しました。

LambdaはNAT GatewayのないPrivate Subnetに配置しています。DB以外にもGitHub APIやOGP取得で外部HTTPSが必要になるため、SquidをL7 forward proxyとして置きました。LambdaのegressはFargateプロキシの13306番、18080番、3128番、4318番に絞り、プロキシから先はTailscale ACLとTiDBユーザーの権限で制御しています。

### LambdaをTailnetへ直接参加させなかった理由

当初はLambdaへ`tsnet` forwarderを同梱し、cold start時にLambda自身をTailnetへ参加させていました。しかしcold startのたびにノードが作られ、管理画面には17個以上、同時に約10個がConnectedの状態になりました。

TailscaleのPersonalプランではephemeral resourceが[月1,000分](https://tailscale.com/pricing)までのため、10個が同時接続すると約100分で到達します。また、[4時間以上存在すると通常のtagged resourceとして扱われます](https://tailscale.com/docs/features/ephemeral-nodes)。Lambda実行環境の寿命に左右されないよう、Tailnetへの接続を常駐Fargateプロキシ1台へ集約しました。

https://x.com/shuntaka_jp/status/2071371477568598087?s=20

### Fargate Spotを使わなかった理由

当初はFargate Spotを使う予定でしたが、東京リージョンでSpot capacityを確保できず、タスクが数十分起動しないことがありました。1タスク構成ではそのままブログのDB接続断になるため、月5ドルほど高くてもFargate On-Demandを選びました。個人ブログなのでプロキシがSPOFであることは許容しています。Vercel側にはISRのキャッシュもあるため、短時間の停止で直ちに全ページが見えなくなるわけではありません。

## LambdaからTiDBまでのレイテンシを観測する

最も気になったのは、AWS東京リージョンからTiDB Self-ManagedまでをTailscaleで往復する遅延です。LambdaとFargate上のforwarderをOpenTelemetryで計装し、CloudWatch Dashboardでp50 / p95 / p99を確認できるようにしました。

以下は2026年7月28日09:00から29日09:00までの24時間を対象に、本番環境のトレースを集計した結果です。アプリケーションで使用していないメンテナンス用クエリは含めていません。

| 計測対象 | 計測範囲 | サンプル数 | p50 | p95 | p99 |
| --- | --- | ---: | ---: | ---: | ---: |
| アプリケーションのDBクエリ | Lambdaでクエリを開始してからTiDBの結果を受け取るまで | 531 | 25ms | 50ms | 83ms |
| 経路確認の`SELECT 1` | Lambda → Fargate → Tailscale → TiDBの往復 | 69 | 18ms | 20ms | 23ms |
| Tailscale経由の接続確立 | FargateのforwarderからTiDBへ接続するまで | 339 | 5ms | 6ms | 7ms |

Tailscale経由の接続確立はp99でも7ms、経路全体の`SELECT 1`もp99で23msでした。現時点ではTailscaleの遅延はボトルネックになっていません。`SELECT 1`だけが悪化すればネットワーク経路、アプリケーションのDBクエリだけが悪化すればSQLや結果転送を疑う、という切り分けができます。

## 費用

### Mini PCの購入費と電気代

Mini PC 3台の購入額は239,970円でした。

![Mini PCを3台購入した注文履歴](https://res.cloudinary.com/dkerzyk09/image/upload/v1785278369/blog/20260729-poem/shoasqhrlrnjl6dubyna.png)

深夜のアイドル寄りな時間帯にCPUパッケージ電力を測ると、3台合計で約22Wでした。メモリ、NVMe SSD、NIC、ファン、ACアダプタの変換ロスとスイッチを加えた壁コンセント側は50〜65W程度と見積もっています。月36〜48kWh、電気代は月1,200〜1,600円ほどです。

### Auroraとの比較

参考として、同じ32GiBメモリのAuroraインスタンスを3台構成にした場合とも比較しました。ここで比較しているのは以前利用していたAurora DSQLの実際の請求額ではなく、Aurora PostgreSQL Standardを同じメモリ容量で常時稼働させる仮定です。

| 項目 | Mini PC | Aurora |
| --- | --- | --- |
| 構成 | Ryzen 7 7730U / 32GB × 3台 | [`db.r7g.xlarge`](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/AuroraUserGuide/Concepts.DBInstanceClass.Summary.html) × 3台（Aurora PostgreSQL Standard） |
| CPU / メモリ（1台） | 8コア / 16スレッド、32GB | 4vCPU、32GiB |
| 初期費用 | 239,970円 | なし |
| 30日間の費用 | 電気代 約1,200〜1,600円 | $1,436.40（229,824円、1ドル160円換算） |
| 料金の前提 | 3台を24時間稼働 | [東京リージョンのオンデマンド料金](https://aws.amazon.com/jp/rds/aurora/pricing/) $0.665/時間 × 3台 × 720時間 |

この仮定なら、Mini PC 3台の本体代はAuroraの約1か月分です。ただし、Auroraのストレージ・I/O料金は含めておらず、CPU性能、可用性、バックアップ、障害対応、運用負荷もまったく異なるので、単純な優劣比較にはなりません。

単価は2026年7月29日にAWS Price List APIで取得しました。レスポンスの`effectiveDate`は2026年7月1日で、同じ条件のI/O-Optimizedは$0.865/時間、Aurora Standardは$0.665/時間でした。

### AWS側の運用費

ドメインの登録・更新料を除き、30日間の月額目安とCost Explorerで確認した2026年7月28日現在の実績をまとめました。金額は1セント単位に切り上げています。

| 項目 | 内訳 | 月額目安 | 7月1〜28日の実績 |
| --- | --- | ---: | ---: |
| Fargate | ARM64、0.25 vCPU / 0.5GB | $8.89 | $8.04 |
| Public IPv4 | 1個 | $3.60 | $3.27 |
| Route 53 Hosted Zone | 3ゾーン | $1.50 | $1.50 |
| Secrets Manager | 4個（EventBridge接続用2個、管理画面用2個） | $1.60 | $0.40 |
| Cloud Map | 1リソース | $0.10 | 未計上 |
| 固定費小計 |  | $15.69（約$15.7） | $13.19 |
| 従量課金 | 主にS3（$1.62） | 変動 | $2.22 |
| 合計 |  | 約$15.7 + 変動 | $15.40 |

Secrets Managerは月の途中で作成され、Cloud Mapはまだ計上されていません。また、表示額の丸めにより内訳の単純合計とは1セントの差があります。

## 使ってみて

Mini PCにしたことで、Raspberry Piクラスタで悩まされていた電源やARM固有の問題はかなり減りました。32GBメモリのx86_64ノードが3台あると、TiDBだけでなくPLaMoのCPU推論、Prometheus、Grafana、Hubbleなども同居させられます。検証した内容をそのままブログの機能へ入れられるので、すでに元は取った気持ちです。

短い日本語クエリの埋め込み生成はp50約0.76秒で、現在のブログ検索には十分です。一方、Wikipedia本文を最大1,024トークンに分割したところ、上限に近い1,023トークンのチャンクはp50 6.97秒かかりました。full self-attentionのAttention部分は入力長`n`に対して`O(n^2)`で計算量が増えるため、長い入力が重くなる一因です。2 Pod・同時4でも実効スループットは約0.28〜0.29チャンク/秒で、追加の89,983チャンクに89時間かかりました。大量投入やモデル検証を気軽に回すため、次はGPUを搭載した専用の推論サーバーが欲しくなりました。

また、自宅回線や機器、OS、Kubernetes、TiDBの面倒を見るのもこの構成の一部です。マネージドサービスの代替として万人に勧められる構成ではありません。特にネットワーク越しに大きな結果セットを返す処理では、自宅回線が明確なボトルネックになります。

普通のブログなら、Aurora DSQLやDynamoDB、あるいは静的サイトで十分です。それでも、個人ブログを技術検証の遊び場として使うという目的には、今回の構成はかなり気に入っています。

構成、IaC、Kubernetesマニフェスト、移行ツール、設計ドキュメントは以下のリポジトリで公開しています。

https://github.com/shuntaka9576/shuntaka-dev
