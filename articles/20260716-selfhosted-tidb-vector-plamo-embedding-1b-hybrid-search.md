---
title: "セルフホストTiDB + PLaMo-Embedding-1B(CPU推論)で日本語セマンティック検索を実測してみた"
description: "セルフホストTiDBにTiFlashを追加してHNSWベクトルインデックスを構築し、CPU推論のPLaMo-Embedding-1Bで日本語Wikipedia 1万チャンクをセマンティック検索。埋め込み生成のスループットから検索レイテンシ・recallまで実測して実用性を検証します"
publish: true
tags:
  - "tech/tidb"
  - "tech/tiflash"
  - "tech/kubernetes"
  - "tech/plamo-embedding-1b"
---

## はじめに

Self-Managed(セルフホスト)のTiDBにTiFlashを追加し、ベクトル検索による日本語セマンティック検索を構築して、どこまで実用になるかを検証します。埋め込みモデルには、以前の記事で紹介したPLaMo-Embedding-1Bを採用し、こちらもGPUなしのCPU推論でセルフホストします。

https://dev.classmethod.jp/articles/shuntaka-try-plamo-embedding-1b/

この構成の最大のメリットは、PostgreSQLにおけるpgvectorと同様に、検索専用のミドルウェアを別に立てず、アプリケーションDB内で検索が完結することです。

- **同期パイプラインが不要**: OpenSearchのような検索エンジンを併設する構成で必要になる、元DBとの同期(CDCやバッチ)の設計・運用がなくなります。埋め込みの生成自体はどの構成でも必要で、不要になるのは同期部分です
- **二重データの管理が不要**: 検索用のデータストアにデータを二重に持たないため、鮮度・整合性の管理から解放されます
- **業務条件の絞り込みをSQLでそのまま書ける**: 公開状態・ユーザー・タグといった条件を、検索と同じクエリのWHEREやJOINで適用できます

実測の数字と結論だけを読みたい場合は、[まとめ](#まとめ)へどうぞ。

## アーキテクチャと検証環境

### 全体構成

全体構成を示します。本記事で扱うのは右側のKubernetes部分のみで、左側はそれを利用するWebシステムです。

![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1784334412/blog/2026-07-16-selfhosted-tidb-vector-plamo-embedding-1b-hybrid-search/ah18icf4fmbaexxlrjog.png)

左側のWebシステムは、個人運用で料金を抑えることを最優先にしているため、一般的な設計とは異なる変則的な構成です。あくまで検証の背景として見てください。Kubernetesクラスタを含むベース環境の詳しい構成・構築方法は、次のリポジトリを参照してください。

https://github.com/shuntaka9576/shuntaka-dev

:::message
本構成で利用しているTailscaleのPersonalプランは、個人による非商用利用が対象です。本記事のWebシステムは、筆者が個人で非商用運営しているサービスであり、この前提でPersonalプランを利用しています。業務・商用目的で同様の構成を採用する場合はPersonalプランの対象外となるため、用途に合った有料プランを検討してください。最新の適用条件は[Tailscaleの料金ページ](https://tailscale.com/pricing)を確認してください。
:::

### 検証環境

検証環境は、ミニPC3台のKubernetesクラスタ上でTiDB Operatorにより運用しているTiDB v8.5.7(PD / TiKV / TiDB構成)です。GPUはありません。

| ノード | CPU                              | メモリ | 本記事での役割     |
| ------ | -------------------------------- | ------ | ------------------ |
| node1  | Ryzen 7 7730U (8コア/16スレッド) | 32GB   | TiFlash            |
| node2  | Ryzen 7 7730U (8コア/16スレッド) | 32GB   | PLaMo推論サービス  |
| node3  | Ryzen 7 7730U (8コア/16スレッド) | 32GB   | PLaMo推論サービス  |

埋め込み生成のCPU推論とベクトルインデックスのビルドは、どちらもCPU負荷の高いワークロードです。リクエストだけに基づくKubernetesのスケジューラ任せにすると同じノードに同居し得るため、この配置はnodeAffinityで明示的に固定しています(詳細はPLaMoの章で説明します)。

:::message
本環境は、TiDBが公式に定める[ハードウェア要件](https://docs.pingcap.com/tidb/stable/hardware-and-software-requirements/)のうち、開発・テスト環境向けの最低要件も下回る構成です(特にTiFlashの最低要件は32コア/64GB)。本記事のレイテンシやスループットの絶対値は、この環境固有の参考値として見てください。主眼は、同一環境内での各検索経路の相対比較と、セルフホスト構成の実用性確認にあります。
:::

クラスタへはTailscale経由で到達し、TiDBにはMagicDNS名(`tidb.<tailnet>`)で接続します。以降のコマンドは次の環境変数を前提にします。

```bash
export TAILNET=$(tailscale status --json | jq -r '.MagicDNSSuffix')
```

## TiDBとMySQLの全文検索、ベクトル検索事情

### TiDB

TiDBの検索機能は提供形態ごとに対応が分かれています。

| 機能                 | Cloud Starter / Essential         | Cloud Dedicated    | Self-Managed             | TiFlash              |
| -------------------- | --------------------------------- | ------------------ | ------------------------ | -------------------- |
| FULLTEXT INDEX       | ○ (Preview / 一部 AWS リージョン) | ✕ (構文パースのみ) | ✕ (構文パースのみ)       | 必須                 |
| VECTOR 型            | ○                                 | ○ (v8.4.0 +)       | ○ (v8.4.0 +)             | 不要                 |
| VECTOR + HNSW[^hnsw] | ○                                 | ○ (v8.4.0 +)       | ○ (v8.4.0 +、v8.5+ 推奨) | 必須                 |

[^hnsw]: HNSW(Hierarchical Navigable Small World)は、ANN(近似最近傍探索)インデックスの代表的なアルゴリズムです。クエリのベクトルに近いベクトルを、全件比較せずにグラフ探索で高速に見つけます。pgvector、OpenSearch、QdrantなどのベクトルDB・検索エンジンでも広く採用されているデファクトスタンダードで、TiDBのベクトルインデックスもこれを使います。

VECTOR型は、格納と距離関数(`VEC_COSINE_DISTANCE`など)だけならTiKVのみで動きますが、インデックスがないため距離計算は全件スキャンになります。HNSWインデックスを張る場合はTiFlashが必須です。

本記事で使うクラスタはSelf-Managed v8.5.7です。FULLTEXT INDEXは構文としてパースされるだけでインデックスは作られず、`fts_match_word`も動きません。VECTOR + HNSWは動きます。検索をTiDB内で完結させるなら、ベクトル検索が現実的な選択肢になります。

なお、TiDBの全文検索は現在ベータ版です。今後の改善予定として、CJK(中日韓)言語に対する検索結果の改善と並んで「TiDB Self-Managedクラスタへの対応」が明記されています🔥

https://pingcap.co.jp/blog/introducing-full-text-search-for-tidb/

### MySQL

同じMySQL互換でも、本家MySQLは状況が逆です。

| 機能           |   MySQL   | 備考                           |
| -------------- | :-------: | ------------------------------ |
| FULLTEXT INDEX |     ○     | `ngram` パーサで日本語にも対応 |
| VECTOR 型      | ○ (9.0 +) | 格納用のデータ型のみ           |
| VECTOR + HNSW  |     ✕     | ANN インデックスは未対応       |

全文検索は以前からできる一方、ベクトルは格納できるだけです。前節の通りTiDBはTiKVのみでも距離関数による全件スキャンの類似検索は書けますが、MySQLは距離関数自体がHeatWave[^heatwave]限定のため、SQLでは類似検索そのものが書けません。ベクトル検索基盤として採用するのは厳しい状況です。

[^heatwave]: HeatWaveは、OracleがOCIなどのクラウドで提供するMySQLベースのマネージドサービスです。インメモリの分析エンジンやベクトル処理といった拡張機能を持ち、コミュニティ版のMySQLにない機能が先行して提供されます。ベクトルの距離関数もその1つです。

## 検索ミドルウェアの比較

検討した候補を本構成の要件で整理すると次の通りです。MySQL互換のDBを探しているという前提に寄った比較ですが、その前提を素直に書き出した結果として見てください。

| 本構成の要件 | TiDB | MySQL | pgvector | OpenSearch |
| --- | :-: | :-: | :-: | :-: |
| ANN (HNSW) が使える | ○ | ✕ | ○ | ○ |
| 検索データの同期パイプラインが不要 | ○ | ○ | ○ | ✕ |
| MySQL 互換（既存資産をそのまま使える） | ○ | ○ | - | - |
| 公開状態・タグ等の絞り込みを検索と同一クエリで書ける | ○ | ○ | ○ | ✕ |
| 2048次元をそのままインデックス化できる | ○ | ✕ | ○[^halfvec] | ○ |

補足が必要な行だけ触れておきます。

- **MySQL互換**: 既存のスキーマ・クエリ・ドライバといった資産をそのまま持ち込めるかという観点です。pgvectorを選ぶ場合はPostgreSQLへの移行が前提になります。
- **公開状態・タグ等の絞り込み**: OpenSearchで同じことをするには、これらの属性をドキュメントに焼き込んで、変更のたびに再インデックスすることになります。タグ階層のような再帰的な展開が必要になった場合も、RDBなら再帰CTEで同一クエリに寄せられます。
- **2048次元をそのままインデックス化できる**: 本記事で採用する埋め込みモデルPLaMo-Embedding-1B(詳細は後述)の出力が2048次元です。

[^halfvec]: pgvectorのvector型はインデックス上限が2000次元のため、halfvec(半精度)へキャストしてインデックスを張るひと手間が必要です。fp16化による検索品質への影響は実用上ほぼ無視できます。

## TiDBの準備

この環境にベクトルインデックスの実行基盤となるTiFlashを追加し、検証用のデータベースとチャンク[^chunk]テーブルを作ります。検証データには日本語Wikipediaを使います。

[^chunk]: 記事本文を埋め込みモデルの入力上限(1,024トークン)に合わせて分割した断片を、本記事ではチャンクと呼びます。1チャンクがテーブルの1行・1ベクトルに対応し、格納と検索の単位になります。分割方法はインデックス化の章で、実測のトークン数(1チャンク平均822トークン)は投入結果の節で扱います。

### TiFlashの追加

TidbClusterのマニフェストに`tiflash`セクションを追記して適用します。

```yaml:tidb-cluster.yaml(抜粋)
tiflash:
  baseImage: pingcap/tiflash
  replicas: 1
  requests:
    cpu: '500m'
    memory: '4Gi'
  limits:
    memory: '8Gi'
  storageClaims:
    - resources:
        requests:
          storage: 50Gi
      storageClassName: local-path
  config:
    config: |
      [logger]
        level = "info"
      [profiles.default]
        max_memory_usage = 0
    proxy: |
      log-level = "info"
  topologySpreadConstraints:
    - topologyKey: kubernetes.io/hostname
      maxSkew: 1
```

TiFlash特有の注意点として、`spec.tiflash.config`はTiFlash本体の`config`と、組み込みTiKV Learnerの`proxy`の2つのサブキーを持ちます(`pd` / `tikv` / `tidb`の`config: |`単一キーとは書式が異なります)。また`storageClaims`はリストで、データディレクトリを複数PVに分散できる設計です。

```bash
kubectl -n tidb-cluster apply -f tidb-cluster.yaml
kubectl -n tidb-cluster get pods -l app.kubernetes.io/component=tiflash -w
# basic-tiflash-0 が 4/4 Running になるまで待つ (5〜15分)
```

PDがTiFlashをストアとして認識していることを確認します。

```bash
kubectl -n tidb-cluster exec -it basic-pd-0 -- /pd-ctl store
# labels に engine=tiflash の store が1つ見えればOK
```

### 検証用DBとチャンクテーブルのDDL

検証データは既存のアプリケーションのDBと混ぜず、専用のデータベースに入れます。分離しておくことで、検証が終わったら`DROP DATABASE`一発で片付けられます。

```bash
mysql -h tidb.${TAILNET} -P 4000 -u root -e "CREATE DATABASE bench_wiki;"
```

Wikipediaのページをチャンク単位で格納するテーブルを作ります。ポイントは`embedding VECTOR(2048)`列と、テーブル単位でTiFlashレプリカを張る`SET TIFLASH REPLICA`です。

```bash
mysql -h tidb.${TAILNET} -P 4000 -u root bench_wiki <<'SQL'
CREATE TABLE wiki_embedding_chunks (
  chunk_id CHAR(36) NOT NULL DEFAULT (UUID()),
  page_id BIGINT UNSIGNED NOT NULL,
  title VARCHAR(512) NOT NULL,
  chunk_index INT UNSIGNED NOT NULL,
  content LONGTEXT NOT NULL,
  token_count INT UNSIGNED NOT NULL,
  embedding VECTOR(2048) NOT NULL,
  created_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  PRIMARY KEY (chunk_id),
  UNIQUE KEY uq_wiki_embedding_chunks_page_index (page_id, chunk_index)
);

ALTER TABLE wiki_embedding_chunks SET TIFLASH REPLICA 1;
SQL
```

- `embedding VECTOR(2048)`: 採用モデルの出力次元に合わせます。`NOT NULL`にしている理由はインデックス化の章で説明します
- `page_id` / `title`: WikipediaダンプのページIDとタイトルです。検索結果を目視確認するときのためにタイトルも持たせます

レプリカの同期状況は`INFORMATION_SCHEMA`で確認できます。

```bash
mysql -h tidb.${TAILNET} -P 4000 -u root -e "
SELECT TABLE_SCHEMA, TABLE_NAME, REPLICA_COUNT, AVAILABLE, PROGRESS
  FROM INFORMATION_SCHEMA.TIFLASH_REPLICA
 WHERE TABLE_NAME = 'wiki_embedding_chunks';"
# AVAILABLE = 1, PROGRESS = 1 になれば同期完了
```

:::message
HNSWインデックスはこの時点では作りません。データの投入が終わった後に作成します。順序を誤るとTiFlashがクラッシュする罠があり、これもインデックス化の章で説明します。
:::

## PLaMoのセルフホスト

### はじめに

埋め込みモデルには、Preferred NetworksがApache License 2.0で公開している日本語特化のPLaMo-Embedding-1Bを採用します。出力は2048次元で、日本語テキスト埋め込みベンチマークのJMTEBで公開時点トップクラスのスコアを記録しているモデルです。GPUなしのCPUで推論できるため、ミニPCクラスタでのセルフホストと相性が良いのが採用理由です。外部の埋め込みAPIに依存しないので、課金もデータの外部送信もありません。

https://huggingface.co/pfnet/plamo-embedding-1b

:::message
同じPLaMo-Embedding-1BはCloudflare Workers AI(`@cf/pfnet/plamo-embedding-1b`、$0.019/100万トークン)でも提供されています。1万チャンク(実測の合計は約824万トークン)の埋め込み生成で比べると次の通りです。それでもセルフホストする理由はまとめで扱います。

|          | 本構成(CPU推論) | Workers AI          |
| -------- | --------------- | ------------------- |
| 所要時間 | 約10時間        | 分〜数十分          |
| 料金     | 電気代のみ      | 約$0.16(約24円)     |
:::

### HTTPサービス化

PLaMo-Embedding-1Bはsentence-transformers形式ではなく、`AutoModel`から`encode_query` / `encode_document`を呼ぶカスタム実装です。そのままではHTTPで叩けないため、FastAPIで薄いラッパーを書きます。モデルはPod起動時に1回だけロードして常駐させます。

```python:server.py(抜粋)
MODEL_ID = os.environ.get("MODEL_ID", "pfnet/plamo-embedding-1b")
DEVICE = "cuda" if torch.cuda.is_available() else "cpu"

tokenizer = AutoTokenizer.from_pretrained(MODEL_ID, trust_remote_code=True)
model = AutoModel.from_pretrained(MODEL_ID, trust_remote_code=True).to(DEVICE).eval()

@app.post("/embed", response_model=EmbedResponse)
def embed(req: EmbedRequest) -> EmbedResponse:
    if req.mode not in ("query", "document"):
        raise HTTPException(status_code=400, detail="mode must be 'query' or 'document'")
    with torch.inference_mode():
        if req.mode == "query":
            vec = model.encode_query(req.text, tokenizer)
        else:
            vec = model.encode_document(req.text, tokenizer)
    vec_list = vec.squeeze(0).cpu().float().tolist()
    return EmbedResponse(vector=vec_list, dim=len(vec_list))
```

検索クエリと格納ドキュメントでエンコード用APIが分かれている点(非対称埋め込み)が重要です。検索時のクエリは`encode_query`、インデックス時の本文は`encode_document`で埋め込みます。

### モデルをイメージに組み込む

モデルファイル(約2.3GB)はDockerイメージのビルド時にダウンロードし、イメージに含めます。Pod起動時のダウンロードをなくし、Hugging Faceへのネットワーク依存をビルド時に閉じ込めるためです。

```dockerfile:Dockerfile
# syntax=docker/dockerfile:1

# CPU 版 PyTorch を linux/amd64 向けにインストール (MiniPC cluster は Ryzen 7 7730U)。
# amd64 wheel は https://download.pytorch.org/whl/cpu に登録済み。
FROM python:3.12-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

RUN pip install --no-cache-dir \
      --index-url https://download.pytorch.org/whl/cpu \
      torch==2.5.1 \
    && pip install --no-cache-dir \
      transformers==4.46.0 \
      sentencepiece==0.2.0 \
      fastapi==0.115.4 \
      uvicorn==0.32.0 \
      pydantic==2.9.2

COPY server.py chunking.py /app/

# ビルド時にモデルファイルを事前ダウンロードし image に焼き込む (~2.3GB)。
# AutoModel.from_pretrained() を使うと RAM に model を全展開して verify する
# ため Docker Desktop 既定メモリ (2-4GB) を超えて OOM になる。snapshot_download は
# ファイル取得だけで RAM を使わない (transformers の transitive dep なので追加
# install 不要)。runtime の server.py は HF cache から load するので network 不要。
ARG MODEL_ID=pfnet/plamo-embedding-1b
ENV MODEL_ID=${MODEL_ID}
RUN python -c "from huggingface_hub import snapshot_download; \
  snapshot_download(repo_id='${MODEL_ID}')"

EXPOSE 8080
CMD ["uvicorn", "server:app", "--host", "0.0.0.0", "--port", "8080"]
```

モデルを組み込む理由と`snapshot_download`を使う理由は、コメントの通りです。ハマりどころが1つあり、Buildxがデフォルトで付けるattestation manifestを古いバージョンのcontainerdが解釈できず、`no match for platform in manifest`でプルに失敗することがあります。`--provenance=false --sbom=false`を付けて単一プラットフォームのマニフェストにすると回避できます。

### GHCRへのプッシュ

イメージはGitHub Container Registry(GHCR、ghcr.io)に置きます。GHCRの`docker login`はGitHubのWebパスワードを受け付けず、PAT(Personal Access Token)が必須です。`gh` CLIを使っている場合は、既存の認証に`packages`スコープを追加してトークンを流用するのが最短です。

```bash
gh auth refresh --scopes write:packages,read:packages
gh auth token | docker login ghcr.io -u <your-account> --password-stdin
# → "Login Succeeded" が出ればOK
```

ビルドとプッシュはBuildxで行います。ターゲットノードはamd64なので、Apple Silicon Macから実行する場合はQEMUエミュレーションを挟んだクロスビルドになります(PyTorchのインストールが重く、初回は10〜20分かかります)。

```bash
docker buildx build \
  --platform linux/amd64 \
  --provenance=false \
  --sbom=false \
  -t ghcr.io/<your-account>/plamo-embedding:latest \
  --push \
  .
```

プッシュ後の注意として、GHCRのパッケージはリポジトリの公開設定にかかわらず、必ず`private`で作られます。`private`のままだとkubeletがプルできず`ErrImagePull`になるため、imagePullSecretsを設定するか、公開して問題ないイメージであればパッケージを`public`へ切り替えます。個人アカウントのパッケージの可視性はREST APIから変更できず、Web UI(パッケージのSettings)からのみ操作できます。

### Kubernetesへの配置

:::details deployment.yaml(全文)

```yaml:deployment.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: plamo-embedding
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: plamo-embedding
  namespace: plamo-embedding
spec:
  # node1 は TiFlash の vector index build 用に空け、node2 / node3 へ 1 Pod ずつ配置する。
  replicas: 2
  # maxSurge=0 で更新中にも追加 Pod を作らず、モデル分のメモリ増加を防ぐ。
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
  selector:
    matchLabels:
      app: plamo-embedding
  template:
    metadata:
      labels:
        app: plamo-embedding
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: kubernetes.io/hostname
                    operator: In
                    values:
                      - node2
                      - node3
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: plamo-embedding
              topologyKey: kubernetes.io/hostname
      containers:
        - name: server
          image: ghcr.io/<your-account>/plamo-embedding:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            periodSeconds: 10
            timeoutSeconds: 3
          startupProbe:
            httpGet:
              path: /healthz
              port: 8080
            # node ごとの初回 image pull と model load を最大 10 分待つ。
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 60
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            periodSeconds: 30
            timeoutSeconds: 5
            failureThreshold: 3
          resources:
            requests:
              cpu: '500m'
              memory: '4Gi'
            limits:
              # 常駐約5.5GiB + 同時リクエスト時のアクティベーションで6Giでは
              # OOMKillされたため8Gi(詳細はインデックス化の章)
              memory: '8Gi'
---
apiVersion: v1
kind: Service
metadata:
  name: plamo-embedding
  namespace: plamo-embedding
spec:
  type: ClusterIP
  selector:
    app: plamo-embedding
  ports:
    - port: 80
      targetPort: 8080
```

:::

配置の意図(TiFlash用にnode1を空ける、`maxSurge: 0`、モデルロードを見込んだstartupProbe)は、マニフェスト内のコメントの通りです。

`resources.requests`が`cpu: 500m`と実際の推論負荷(8コア、後述)より大幅に小さいのは意図的です。実態に合わせて`cpu: 8`を要求するとノードの半分を常時予約する硬直した配置になるため、requestsは小さく保ち、代わりにnodeAffinity / podAntiAffinityで配置を固定しています。スケジューラはrequestsしか見ないので、この縛りがないとTiFlashのいるnode1へPLaMoが同居し得ます。

ServiceはTailscale Kubernetes Operator経由で`plamo-embedding.<tailnet>`として公開し、動作確認します。

```bash
curl -s -X POST http://plamo-embedding.${TAILNET}/embed \
  -H 'Content-Type: application/json' \
  -d '{"text":"日本語の検索テスト","mode":"query"}' | jq '.dim'
# → 2048
```

### 推論の負荷特性

常駐メモリはアイドル状態(リクエストなし、CPUほぼゼロ)でPodあたり約5.5〜5.7GiBでした。モデルをロードしたプロセスがこのサイズで常駐するため、メモリ上限までの余裕は大きくありません。`deployment.yaml`の`maxSurge: 0`で更新中に追加Podを作らないようにしているのは、ノード上にもう1つ常駐させる余裕がないためです。なお、当初の上限は6Giとしていましたが、これは同時1リクエストを前提とした値で、後の投入中にOOMKilledとなり8Giへ引き上げることになります(インデックス化の章で後述)。

![PLaMo Podのアイドル時メモリ。node2が5.72GiB、node3が5.45GiB](https://res.cloudinary.com/dkerzyk09/image/upload/v1784334522/blog/2026-07-16-selfhosted-tidb-vector-plamo-embedding-1b-hybrid-search/tdqugtvfcwuwmltd4vw1.png)
*GrafanaのPod一覧。plamo-embeddingの2 Podがメモリ使用量の上位2つを占め、アイドル時でも5.5GiB前後を常時使用している*

:::details コマンドでの実測

```bash
for p in $(kubectl -n plamo-embedding get pods -o name); do
  echo "== $p"
  # uvicornプロセス(PID 1)のRSS = モデル常駐分の実メモリ
  kubectl -n plamo-embedding exec "$p" -- grep VmRSS /proc/1/status
  # コンテナcgroup全体のメモリ使用量
  kubectl -n plamo-embedding exec "$p" -- cat /sys/fs/cgroup/memory.current \
    | awk '{printf "cgroup memory.current: %.2f GiB\n", $1/1073741824}'
done

== pod/plamo-embedding-7f46d8c85f-5vfm7
VmRSS:	 5795120 kB
cgroup memory.current: 5.45 GiB
== pod/plamo-embedding-7f46d8c85f-r6pwv
VmRSS:	 6083584 kB
cgroup memory.current: 5.72 GiB
```

:::

エンコード時間は次の方法で計測しました。

- `/chunks`で上限いっぱい(1,023トークン)のチャンクを1件生成し、`mode: document`のリクエストJSONを作る
- ウォームアップ3回を捨てたあと、同一リクエストを逐次20回実行する(同時実行なし)
- 計測値はTailscale経由のクライアントで取ったcurlの`time_total`(E2E)。ローカルネットワークのRTTは数msなので誤差範囲です

:::details 計測スクリプト(リクエストJSONの生成と計測ループ)

リクエストJSONの生成です。

```python
import json, urllib.request

TAILNET = "<tailnet>"
text = "日本語のベンチマーク用テキストです。埋め込みモデルの推論時間を計測しています。" * 400
body = json.dumps({"title": "bench", "description": "", "content": text,
                   "max_tokens": 1024, "overlap_tokens": 128}).encode()
req = urllib.request.Request(f"http://plamo-embedding.{TAILNET}/chunks",
                             data=body, headers={"Content-Type": "application/json"})
resp = json.load(urllib.request.urlopen(req, timeout=120))
chunk = resp["chunks"][0]
print("token_count:", chunk["token_count"])  # → 1023
with open("embed-req.json", "w") as f:
    json.dump({"text": chunk["embedding_text"], "mode": "document"}, f, ensure_ascii=False)
```

計測ループです。

```bash
# ウォームアップ3回
for i in 1 2 3; do
  curl -s -o /dev/null -X POST http://plamo-embedding.${TAILNET}/embed \
    -H 'Content-Type: application/json' -d @embed-req.json
done

# 本計測20回 → 分布を集計
for i in $(seq 1 20); do
  curl -s -o /dev/null -w '%{time_total}\n' \
    -X POST http://plamo-embedding.${TAILNET}/embed \
    -H 'Content-Type: application/json' -d @embed-req.json
done | sort -n | awk '{a[NR]=$1} END {
  printf "min: %.3fs\np50: %.3fs\np95: %.3fs\nmax: %.3fs\n", a[1], a[int(NR*0.5)], a[int(NR*0.95)], a[NR]}'
```

:::

結果は次の通りです。

| 指標 | 時間  |
| ---- | ----- |
| min  | 6.88秒 |
| p50  | 6.97秒 |
| p95  | 7.04秒 |
| max  | 7.06秒 |

**1チャンク(約1,000トークン)のエンコードに約7秒**かかります。ばらつきが3%未満と極めて安定しているのは、CPU推論らしい挙動です。エンコード時間は入力トークン数にほぼ比例するため、検索クエリのような短文は1秒未満で返る一方、1,000トークン級のドキュメントチャンクはこの重さになります。つまり、CPU推論の実用性は「クエリ側(検索)は問題ないが、ドキュメント側(インデックス化)は時間との勝負」という非対称な結論になります。

ベンチマーク中のノードCPU使用率は40%前後までしか上がりません。それでも並列リクエストを投げたときのスループットはほとんど伸びず、実測上限はPodあたり約1/7 req/s、同時4(Podあたり2)の合計で0.26 req/sでした。次章のデータ投入は、この値を前提に同時4で実行します。「CPUが余っているのになぜ伸びないのか」の深掘りは付録に回します。

![逐次ベンチマーク中のノード別CPU使用率。node3が約38%、node2が約20%で頭打ち](https://res.cloudinary.com/dkerzyk09/image/upload/v1784334520/blog/2026-07-16-selfhosted-tidb-vector-plamo-embedding-1b-hybrid-search/hsu8hiyb3qpfanhqdg2z.png)
*逐次ベンチマーク中のノードCPU。リクエストは2 Podへ接続単位で分散するため、各ノードのCPU使用率は最大でも40%前後にとどまる*

## Wikipediaのインデックス化

### データの取得

検証コーパスには日本語WikipediaのCirrusSearchダンプを使います。通常のXMLダンプと違い、wikitext(マークアップ)を除去済みのプレーンテキストが`text`フィールドに入っているため、前処理がほぼ不要です。

なお配布場所は`other/cirrussearch/`(週次の単一巨大ファイル)から`other/cirrus_search_index/`(約640MBのシャード分割)へ移行しています。日本語Wikipedia本文(`jawiki_content`)は全14シャード・合計約9GBですが、1万チャンクの投入なら1シャードで十分です。ダウンロードは実行環境となるnode1上で直接行います(手順は後述)。

中身はNDJSONで、`{"index": {"_id": ...}}`の行と、`page_id` / `title` / `text`を持つドキュメント行が2行1組で並んでいます。使うのはドキュメント行だけです。

### チャンク化と投入

ブログ記事と違い、WikipediaはMarkdownの見出し構造を持たないため、PLaMoトークナイザによる固定長ウィンドウ(1,024トークン、128トークンのオーバーラップ)で分割します。各チャンクの先頭にはページタイトルを付与してから、`mode=document`で埋め込みます。投入スクリプトの流れは次の通りです。

1. ダンプをストリームで読み、`page_id` / `title` / `text`を取り出す(200文字未満のスタブは除外)
2. PLaMoサービスの`POST /chunks`でトークナイズして固定長チャンクに分割する
3. `POST /embed`(`mode=document`)で2048次元ベクトルを得る(並列度は負荷特性の実測に基づく同時4)
4. `wiki_embedding_chunks`へINSERTする(embeddingは`'[0.02,-0.11,...]'`形式の文字列リテラル)

この流れをPythonスクリプト`ingest_wiki.py`に実装しています。11時間級の長時間実行になるため、途中で停止しても投入済みページをスキップして再開できる作りにしておくことが重要です。INSERTをページ単位のトランザクションにし、起動時に投入済みの`page_id`を読み飛ばすようにしています。

実行環境はnode1です。約11時間の長時間実行になるため、手元のPCではなく常時稼働しているクラスタノードへSSH接続し、`nohup`でバックグラウンド実行します(手元でtmuxを使っているとリモートのtmuxとネストして操作が複雑になるため、リモート側では使いません)。node1を選ぶのは、PLaMoのPodが載っておらず、投入処理が推論のCPUと競合しないためです。node1自体もtailnetに参加しているので、接続先の組み立てにはこれまでと同じ`TAILNET`を使えます。

```bash
# node1に作業ディレクトリを作り、スクリプトを送る
ssh node1 'mkdir -p ~/work/20260717'
scp ingest_wiki.py node1:~/work/20260717/

ssh node1
cd ~/work/20260717

# uvの初回インストール(依存のpymysqlはPEP 723メタデータからuv runが解決する)
curl -LsSf https://astral.sh/uv/install.sh | sh && source ~/.local/bin/env

# シャード(約640MB)を作業ディレクトリへ直接ダウンロード(日付は取得時点の最新に読み替え)
curl -LO "https://dumps.wikimedia.org/other/cirrus_search_index/20260712/index_name%3Djawiki_content/jawiki_content-20260712-00000.json.bz2"

# 接続先(PLaMoサービスとTiDB)はTAILNETから組み立てる
export TAILNET=$(tailscale status --json | jq -r '.MagicDNSSuffix')

# まず小さく動作確認
uv run ingest_wiki.py jawiki_content-20260712-00000.json.bz2 --target-chunks 100

# 本実行(1万チャンク、約11時間)。nohupでバックグラウンドに投げたらsshを切ってよい
nohup uv run ingest_wiki.py jawiki_content-20260712-00000.json.bz2 --target-chunks 10000 \
  > ingest-10k.log 2>&1 &

# embed / INSERTなしでチャンク数の見積もりだけ行う場合
uv run ingest_wiki.py jawiki_content-20260712-00000.json.bz2 --dry-run
```

進捗はバッチ(8チャンク、約30秒)ごとにログへ出ます。投入済み件数、実効スループット、残りのETAが確認でき、中断しても再実行すれば投入済みページをスキップして続きから再開します。この再開ロジックがあるため、動作確認で入れた分の削除は不要で、本実行がそのまま引き継いで1万件の一部になります。チャンク分割のパラメータを変えて入れ直す場合だけ、`TRUNCATE TABLE wiki_embedding_chunks`でリセットしてください。

```text
[21:15:32] 248/10000 chunks (102 pages, 0.26 chunks/s, ETA 10.4h)
```

`nohup`で実行した後は、そのままSSH接続を切断できます。途中経過は手元から次のように確認できます。

```bash
ssh node1 'tail -3 ~/work/20260717/ingest-10k.log'
```

スループットの支配項は埋め込み生成です。前章の実測(1チャンク約7秒、同時4で0.26チャンク/s)から、**1日あたり約2.2万チャンクが上限**になります。1万チャンクで約11時間、10万チャンクで約4.5日という規模感で、100万チャンク分の実埋め込み生成(約45日)は現実的ではありません。CPU推論のセルフホストは、検索クエリの埋め込みには十分でも、大規模コーパスの一括投入では明確なボトルネックになります。本記事ではまず1万チャンクを投入して検索性能を計測します。

### 投入開始1分でPodがOOMKillされる

本実行を開始して約1分、ログにHTTPリトライが並びました。

```text
[04:05:58] HTTP retry 1/3 http://plamo-embedding.<tailnet>/embed: Remote end closed connection without response
```

Podを確認すると、node3側が終了コード137(OOMKilled)で再起動していました。原因は同時リクエストです。常駐約5.5GiBのところへ、同時に処理する推論のアクティベーション(中間バッファ)が積み上がり、当初の`limits.memory: 6Gi`を超えました。実際、上限引き上げ後に計測した投入中のPodメモリは約6.1GiBで、変更前の上限をまさに超える値です。負荷特性の章で触れた「わずかなヘッドルーム」は、同時1リクエスト前提の値だったわけです。

対処はメモリ上限を8Giへ引き上げるだけです。`maxSurge: 0 / maxUnavailable: 1`のローリング更新なので投入ジョブを止める必要はなく、切り替え中の失敗はスクリプトのリトライが吸収し、ページ単位トランザクションのおかげでデータの不整合もありませんでした(PLaMoの章に掲載しているマニフェストは修正済みの8Giです)。

教訓は「**アイドル時の実測だけでメモリ上限を決めない**」です。同時実行ではCPUだけでなく、リクエスト数に比例してアクティベーション分のメモリも必要になります。

### 1万チャンクの埋め込み完了

本実行は序盤にOOMKillが発生したものの、そのまま完走しました。ログの抜粋です(時刻はUTCで、JSTでは13:05開始、22:45完了です)。

```text
$ ssh node1 'tail -f ~/work/20260717/ingest-10k.log'

nohup: ignoring input
[04:05:49] resume: 13 pages / 17 chunks already in DB
[04:05:49] reading jawiki_content-20260712-00000.json.bz2
[04:05:58] HTTP retry 1/3 http://plamo-embedding.tailea8e2.ts.net/embed: Remote end closed connection without response
[04:05:58] HTTP retry 1/3 http://plamo-embedding.tailea8e2.ts.net/embed: Remote end closed connection without response
[04:05:58] HTTP retry 1/3 http://plamo-embedding.tailea8e2.ts.net/embed: Remote end closed connection without response
[04:05:58] HTTP retry 1/3 http://plamo-embedding.tailea8e2.ts.net/embed: Remote end closed connection without response
[04:06:50] 27/10000 chunks (2 pages, 0.16 chunks/s, ETA 16.9h)
[04:07:10] 35/10000 chunks (8 pages, 0.22 chunks/s, ETA 12.5h)
[04:07:51] 48/10000 chunks (12 pages, 0.25 chunks/s, ETA 10.9h)
[04:08:20] 57/10000 chunks (17 pages, 0.27 chunks/s, ETA 10.4h)
[04:08:41] 65/10000 chunks (22 pages, 0.28 chunks/s, ETA 9.9h)
[04:09:02] 73/10000 chunks (28 pages, 0.29 chunks/s, ETA 9.5h)

(中略)

[13:40:17] 9935/10000 chunks (4041 pages, 0.29 chunks/s, ETA 0.1h)
[13:40:59] 9943/10000 chunks (4045 pages, 0.29 chunks/s, ETA 0.1h)
[13:41:20] 9951/10000 chunks (4049 pages, 0.29 chunks/s, ETA 0.0h)
[13:41:56] 9960/10000 chunks (4051 pages, 0.29 chunks/s, ETA 0.0h)
[13:42:27] 9970/10000 chunks (4057 pages, 0.29 chunks/s, ETA 0.0h)
[13:42:49] 9978/10000 chunks (4062 pages, 0.29 chunks/s, ETA 0.0h)
[13:43:16] 9986/10000 chunks (4067 pages, 0.29 chunks/s, ETA 0.0h)
[13:43:55] 9997/10000 chunks (4068 pages, 0.29 chunks/s, ETA 0.0h)
[13:45:22] 10017/10000 chunks (4070 pages, 0.29 chunks/s, ETA 0.0h)
[13:45:22] done: 10017 chunks (4070 pages this run)

```

約9時間40分で10,017チャンク(4,083ページ、合計約824万トークン)が入りました。1チャンクあたりのトークン数は、上限1,024に対して平均822です。実効スループットは0.29チャンク/s(約240トークン/s)で、事前見積もりの0.26チャンク/s(約11時間)を1割ほど上回っています。見積もりの根拠にした付録の並列スイープは、計測中にOOMKillで片方のPodが一時離脱していた疑いがあるため、健全な2 Pod・同時4の実力値はこちらと考えてよさそうです。

### 投入中のノード負荷

投入時間帯(JST 13:05〜22:45)のノードCPUとメモリです。

![投入中のノード別CPU使用率。node2とnode3が20〜97%の間で振動し、node1は3%前後のまま](https://res.cloudinary.com/dkerzyk09/image/upload/v1784334525/blog/2026-07-16-selfhosted-tidb-vector-plamo-embedding-1b-hybrid-search/vrmmjwb7gz0ozuupnu9h.png)
*投入中のノードCPU。推論を担うnode2 / node3だけが約10時間振動し続け、TiFlash用に空けたnode1はほぼ無風*

node2 / node3のCPU使用率は、20〜97%の間を振動し続けます。同時4のリクエストは接続ごとに2 Podへランダムに分散するため、瞬間ごとにPodあたり0〜2件の推論が重なります。2件重なればSMT込みの論理コアまで埋まって90%を超え、1件なら50%前後になるという、付録で見た挙動を行き来している形です。一方、node1は投入スクリプトが動いているにもかかわらず、3%前後のままです。スクリプトの処理はHTTP応答待ちとINSERTだけなので負荷はほぼなく、nodeAffinityで狙った「TiFlash用にnode1を空ける」配置が投入中も維持できています。

![投入中のノード別メモリ使用率。node2とnode3が投入開始とともに31〜32%から38〜40%へ上昇し、投入終了後も下がらない](https://res.cloudinary.com/dkerzyk09/image/upload/v1784334528/blog/2026-07-16-selfhosted-tidb-vector-plamo-embedding-1b-hybrid-search/elxzowokri2kassd9cic.png)
*ノードメモリ。node2 / node3は投入開始とともに上昇し、投入が終わっても元の水準へ戻らない*

メモリで目を引くのは、投入が終わってもnode2 / node3が38〜40%のまま、投入前の31〜32%へ戻らない点です。正体はPLaMo Podでした。投入完了から約7時間後のアイドル状態で実測すると、Podのメモリは投入前より1〜1.8GiB増えたままです。

| PLaMo Pod | 投入前アイドル | 投入中ピーク(memory.peak) | 投入後アイドル |
| --- | --: | --: | --: |
| node2側 | 5.72GiB | 7.31GiB | 6.81GiB |
| node3側 | 5.45GiB | 7.81GiB | 7.21GiB |

増加分は、cgroupの内訳を見るとほぼ全量がanon(ヒープ)です。これはリークではなく、同時リクエストの推論用に確保したアクティベーションのメモリを、プロセスのアロケータ(glibc)がOSへ返さず保持し続けるためです。確保済みの領域は次の推論で再利用されるので、実害は「使用量がピークに張り付いて見える」ことだけで、放置して問題ありません。なお、ノード全体の増加分(2〜2.5GB)のうちPodの増加で説明できない残りは、書き込みを受けたTiKV側のキャッシュ類とみられます。

:::details コマンドでの実測(投入完了から約7時間後)

```bash
for p in $(kubectl -n plamo-embedding get pods -o name); do
  echo "== $p"
  # 現在の使用量とcgroup作成以降のピーク
  kubectl -n plamo-embedding exec "$p" -- cat /sys/fs/cgroup/memory.current \
    | awk '{printf "memory.current: %.2f GiB\n", $1/1073741824}'
  kubectl -n plamo-embedding exec "$p" -- cat /sys/fs/cgroup/memory.peak \
    | awk '{printf "memory.peak:    %.2f GiB\n", $1/1073741824}'
  # 内訳のうち匿名ページ(ヒープ)。currentとの差がファイルキャッシュ等
  kubectl -n plamo-embedding exec "$p" -- grep -E '^anon ' /sys/fs/cgroup/memory.stat \
    | awk '{printf "anon:           %.2f GiB\n", $2/1073741824}'
done

== pod/plamo-embedding-7db9654b75-kl4kq   (node2)
memory.current: 6.81 GiB
memory.peak:    7.31 GiB
anon:           6.78 GiB
== pod/plamo-embedding-7db9654b75-rn9wz   (node3)
memory.current: 7.21 GiB
memory.peak:    7.81 GiB
anon:           7.19 GiB
```

:::

むしろ見るべきはピーク値です。node3側のmemory.peakは7.81GiBで、引き上げ後の上限である8Giまで200MiB弱しか余裕がありませんでした。同時4でこの水準なので、並列度をさらに上げる余地は、メモリの観点ではほぼ残っていません。OOMKillの教訓に「上限の妥当性はアイドル時の実測ではなくmemory.peakで確認する」を付け加えておきます。

### HNSWインデックスの作成

全チャンクの投入完了後、TiFlashレプリカの同期を待ってからCOMPACTを実行し、その後にHNSWインデックスを作ります。

```bash
mysql -h tidb.${TAILNET} -P 4000 -u root bench_wiki <<'SQL'
ALTER TABLE wiki_embedding_chunks COMPACT;
CREATE VECTOR INDEX idx_wiki_embedding_chunks_embedding
  ON wiki_embedding_chunks ((VEC_COSINE_DISTANCE(embedding))) USING HNSW;
SQL
```

ビルドの進捗は`INFORMATION_SCHEMA.TIFLASH_INDEXES`で確認できます。

```bash
mysql -h tidb.${TAILNET} -P 4000 -u root -e "
SELECT TIDB_DATABASE, TIDB_TABLE, INDEX_NAME,
       ROWS_STABLE_INDEXED, ROWS_STABLE_NOT_INDEXED,
       ROWS_DELTA_INDEXED, ROWS_DELTA_NOT_INDEXED, ERROR_MESSAGE
  FROM INFORMATION_SCHEMA.TIFLASH_INDEXES
 WHERE TIDB_DATABASE = 'bench_wiki'
   AND TIDB_TABLE = 'wiki_embedding_chunks';"
# ROWS_STABLE_NOT_INDEXED と ROWS_DELTA_NOT_INDEXED が0、
# ERROR_MESSAGE が空ならビルド完了
```

1万チャンク時点の実測です。COMPACTとインデックス作成を合わせたコマンド全体が8.8秒で返りました。TiFlashのサーバーログを見ると、HNSWインデックスのビルド本体は7.9秒です(抜粋、時刻はUTC)。

```text
[2026/07/17 21:25:14.602] EnsureStableLocalIndex - Begin building index, dm_files=[dmf_11(v=0)]
[2026/07/17 21:25:22.456] EnsureStableLocalIndex - Finish building index, dm_files=[dmf_11(v=0)]
```

DDLが返ったのはビルド完了の0.2秒後でした。`CREATE VECTOR INDEX`は既存データのビルド完了を待ってから完了するため、返ってきた時点で`TIFLASH_INDEXES`は全行インデックス済みを示します。埋め込み生成の9時間40分に対して、1万ベクトル(2048次元)のインデックス作成は8秒と、完全に誤差の範囲でした。

サイズも見ておきます。HNSWインデックスの実体はTiFlashのDMFileディレクトリに`idx_<index_id>.vector`として置かれ、79.6MiBでした。embedding列の生データ(10,017行×2048次元×4バイト≒78.3MiB)とほぼ同じサイズで、HNSWがインデックス内にベクトル本体も持つ構造のため、ベクトル分のディスクはおおよそ2倍になります。

:::details TiFlash Pod上での確認(テーブルIDはビルドログのtable_id=438から)

```bash
kubectl -n tidb-cluster exec basic-tiflash-0 -c tiflash -- \
  ls -la /data0/db/data/t_438/stable/dmf_11

-rw-r--r-- 1 root root   393408 Jul 17 21:25 0.merged
-rw-r--r-- 1 root root 14475467 Jul 17 21:25 5.dat
-rw-r--r-- 1 root root 82383193 Jul 17 21:25 7.dat          # embedding列(理論値82,059,264Bとほぼ一致)
-rw-r--r-- 1 root root 83509628 Jul 17 21:25 idx_3.vector   # HNSWインデックス本体
-rw-r--r-- 1 root root     1695 Jul 17 21:25 meta
-rw-r--r-- 1 root root     1722 Jul 17 21:25 v1.meta
```

:::

### 順序を誤るとTiFlashが落ちる

「データ投入→COMPACT→インデックス作成」の順序は重要です。別の検証で、行が入っていない状態でHNSWインデックスを先行作成したところ、TiFlash(v8.5.7)が既存DMFileへのインデックスビルドで0除算を起こし、終了コード136でCrashLoopに陥りました。

```text
Received signal Floating point exception(8).
Integer divide by zero.
FramedChecksumReadBuffer<XXH3>::doSeek
DMFileVectorIndexWriter::buildIndexForFile
```

厄介なのは、TiDB側でインデックスをDROPしても、TiFlashがPVC上に残ったlocal-indexタスクを起動直後に再開し、クラッシュを繰り返す点です。復旧には`SET TIFLASH REPLICA 0`でレプリカを一度切り離し、Podを再作成する必要がありました。チャンクテーブルの`embedding`を`NOT NULL`にしているのも、この事象を踏まえて「埋め込みのない行がインデックスビルドの対象にならない」ことを保証するためです。

## 検索性能

同じ近傍検索クエリを3つの実行経路で実行し、実行計画とレイテンシを比較します。対象は投入済みの1万チャンク(10,017行)です。以降、経路の呼び方は次の表で固定します。HNSWという語は、インデックスの実装を指すときだけ使います。

| パターン | 呼び方 | 実行場所 | 距離計算 |
| :-: | --- | --- | --- |
| 1 | TiKV全件スキャン | TiKV(行指向) | 全件と比較(近似なし) |
| 2 | TiFlash全件スキャン | TiFlash(列指向) | 全件と比較(近似なし) |
| 3 | ANN | TiFlash + HNSWインデックス | グラフ探索(近似) |

HNSWインデックスは前章で作成済みのため、パターン2(TiFlash全件スキャン)はそのままではANNに乗ってしまいます。インデックスを避ける書き方はパターン2の節で説明します。

### クエリベクトルの生成

検索クエリの埋め込みはPLaMoの`mode=query`で生成します(ドキュメント側の`mode=document`と非対称です)。

```bash
QVEC=$(curl -s -X POST http://plamo-embedding.${TAILNET}/embed \
  -H 'Content-Type: application/json' \
  -d '{"text":"日本の城の石垣の構造","mode":"query"}' | jq -c '.vector')
```

### パターン1: TiKVで全件スキャン

インデックスもTiFlashも使わない経路です。VECTOR型と距離関数だけで動くため、TiFlashを追加していない素のTiDBの実力に相当します。

```bash
mysql -h tidb.${TAILNET} -P 4000 -u root bench_wiki -e "
SELECT /*+ READ_FROM_STORAGE(TIKV[c]) */
       page_id, title,
       VEC_COSINE_DISTANCE(embedding, '${QVEC}') AS distance
  FROM wiki_embedding_chunks AS c
 ORDER BY VEC_COSINE_DISTANCE(embedding, '${QVEC}')
 LIMIT 10;"
```

実行計画は`cop[tikv]`の`TableFullScan`になり、全チャンクとの距離計算が行指向ストレージ上で走ります。

:::details 実行計画(EXPLAIN全文)

```text
+----------------------------------+----------+-----------+---------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| id                               | estRows  | task      | access object | operator info                                                                                                                                                                                                                                   |
+----------------------------------+----------+-----------+---------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Projection_8                     | 10.00    | root      |               | bench_wiki.wiki_embedding_chunks.page_id, bench_wiki.wiki_embedding_chunks.title, vec_cosine_distance(bench_wiki.wiki_embedding_chunks.embedding, [-1.5,1.7,3.8,-2.8,-1,(2043 more)...])->Column#12                                             |
| └─Projection_16                  | 10.00    | root      |               | bench_wiki.wiki_embedding_chunks.page_id, bench_wiki.wiki_embedding_chunks.title, bench_wiki.wiki_embedding_chunks.embedding                                                                                                                    |
|   └─TopN_9                       | 10.00    | root      |               | Column#13, offset:0, count:10                                                                                                                                                                                                                   |
|     └─Projection_17              | 10.00    | root      |               | bench_wiki.wiki_embedding_chunks.page_id, bench_wiki.wiki_embedding_chunks.title, bench_wiki.wiki_embedding_chunks.embedding, vec_cosine_distance(bench_wiki.wiki_embedding_chunks.embedding, [-1.5,1.7,3.8,-2.8,-1,(2043 more)...])->Column#13 |
|       └─TableReader_15           | 10.00    | root      |               | data:TopN_14                                                                                                                                                                                                                                    |
|         └─TopN_14                | 10.00    | cop[tikv] |               | vec_cosine_distance(bench_wiki.wiki_embedding_chunks.embedding, [-1.5,1.7,3.8,-2.8,-1,(2043 more)...]), offset:0, count:10                                                                                                                      |
|           └─TableFullScan_13     | 10017.00 | cop[tikv] | table:c       | keep order:false                                                                                                                                                                                                                                |
+----------------------------------+----------+-----------+---------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

:::

### パターン2: TiFlashで全件スキャン

同じ全件スキャンを列指向のTiFlashで実行します。近似なしの正確な結果(いわゆるexact search)を返すため、後述のrecall@10[^recall]の正解データもこの経路で作ります。

[^recall]: HNSWインデックスを使う近似検索(ANN)は、近似なしの全件スキャン(exact search)と同じ結果を返す保証がありません。そのズレを測るのがrecall@10で、全件スキャンのtop10のうちANNのtop10にも入っていた件数÷10です(全一致で1.0、6件なら0.6)。

注意点として、HNSWインデックスがあると`ORDER BY 距離関数 + LIMIT`の形は自動的にANNへ乗るため、hintでTiFlashを指定するだけでは全件スキャンになりません。ORDER BY式に`+ 0`を足して、インデックスが効く形から意図的に外します(0を足しても順序は変わらないため、結果は近似なしの全件スキャンと同一です)。パターン1とのdiffで示します。

```diff bash
 mysql -h tidb.${TAILNET} -P 4000 -u root bench_wiki -e "
-SELECT /*+ READ_FROM_STORAGE(TIKV[c]) */
+SELECT /*+ READ_FROM_STORAGE(TIFLASH[c]) */
        page_id, title,
        VEC_COSINE_DISTANCE(embedding, '${QVEC}') AS distance
   FROM wiki_embedding_chunks AS c
- ORDER BY VEC_COSINE_DISTANCE(embedding, '${QVEC}')
+ ORDER BY VEC_COSINE_DISTANCE(embedding, '${QVEC}') + 0
  LIMIT 10;"
```

EXPLAINの`TableFullScan`から、次節で触れる`annIndex`表記が消えていれば全件スキャンです。実行計画上はソートキーが`plus(vec_cosine_distance(...), 0)`になっており、インデックスの適用対象から外れたことが読み取れます。列指向フォーマットとベクトル化実行の効果がどの程度出るかを、パターン1との差分で見ます。

:::details 実行計画(EXPLAIN全文)

```text
+----------------------------------------+----------+--------------+---------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| id                                     | estRows  | task         | access object | operator info                                                                                                                                                                                                                                            |
+----------------------------------------+----------+--------------+---------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Projection_8                           | 10.00    | root         |               | bench_wiki.wiki_embedding_chunks.page_id, bench_wiki.wiki_embedding_chunks.title, vec_cosine_distance(bench_wiki.wiki_embedding_chunks.embedding, [-1.5,1.7,3.8,-2.8,-1,(2043 more)...])->Column#12                                                      |
| └─Projection_21                        | 10.00    | root         |               | bench_wiki.wiki_embedding_chunks.page_id, bench_wiki.wiki_embedding_chunks.title, bench_wiki.wiki_embedding_chunks.embedding                                                                                                                             |
|   └─TopN_11                            | 10.00    | root         |               | Column#14, offset:0, count:10                                                                                                                                                                                                                            |
|     └─Projection_22                    | 10.00    | root         |               | bench_wiki.wiki_embedding_chunks.page_id, bench_wiki.wiki_embedding_chunks.title, bench_wiki.wiki_embedding_chunks.embedding, plus(vec_cosine_distance(bench_wiki.wiki_embedding_chunks.embedding, [-1.5,1.7,3.8,-2.8,-1,(2043 more)...]), 0)->Column#14 |
|       └─TableReader_18                 | 10.00    | root         |               | MppVersion: 2, data:ExchangeSender_17                                                                                                                                                                                                                    |
|         └─ExchangeSender_17            | 10.00    | mpp[tiflash] |               | ExchangeType: PassThrough                                                                                                                                                                                                                                |
|           └─Projection_19              | 10.00    | mpp[tiflash] |               | bench_wiki.wiki_embedding_chunks.page_id, bench_wiki.wiki_embedding_chunks.title, bench_wiki.wiki_embedding_chunks.embedding                                                                                                                             |
|             └─TopN_16                  | 10.00    | mpp[tiflash] |               | Column#13, offset:0, count:10                                                                                                                                                                                                                            |
|               └─Projection_20          | 10017.00 | mpp[tiflash] |               | bench_wiki.wiki_embedding_chunks.page_id, bench_wiki.wiki_embedding_chunks.title, bench_wiki.wiki_embedding_chunks.embedding, plus(vec_cosine_distance(bench_wiki.wiki_embedding_chunks.embedding, [-1.5,1.7,3.8,-2.8,-1,(2043 more)...]), 0)->Column#13 |
|                 └─TableFullScan_14     | 10017.00 | mpp[tiflash] | table:c       | keep order:false                                                                                                                                                                                                                                         |
+----------------------------------------+----------+--------------+---------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

:::

### パターン3: TiFlash + HNSWインデックス(ANN)

HNSWインデックス作成後は、`ORDER BY 距離関数 + LIMIT`の形のクエリが自動的にインデックスへ乗ります。SQLはパターン2から`+ 0`を外しただけ、つまりパターン1のhintを`TIFLASH`に変えた素の形です。パターン2とのdiffで示します。

```diff bash
 mysql -h tidb.${TAILNET} -P 4000 -u root bench_wiki -e "
 SELECT /*+ READ_FROM_STORAGE(TIFLASH[c]) */
        page_id, title,
        VEC_COSINE_DISTANCE(embedding, '${QVEC}') AS distance
   FROM wiki_embedding_chunks AS c
- ORDER BY VEC_COSINE_DISTANCE(embedding, '${QVEC}') + 0
+ ORDER BY VEC_COSINE_DISTANCE(embedding, '${QVEC}')
  LIMIT 10;"
```

インデックスが使われているかは、`EXPLAIN`で`TableFullScan`の`operator info`に`annIndex:COSINE(...)`が現れるかで判断します。HNSW利用時もexecutor名は`TableFullScan`のままである点に注意してください。

```text
TableFullScan_21  mpp[tiflash]  table:c,
  index:idx_wiki_embedding_chunks_embedding(embedding),
  annIndex:COSINE(embedding..[...], limit:10)
```

:::details 実行計画(EXPLAIN全文)

```text
+----------------------------------------+---------+--------------+---------------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| id                                     | estRows | task         | access object                                                 | operator info                                                                                                                                                                                                                                   |
+----------------------------------------+---------+--------------+---------------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Projection_8                           | 10.00   | root         |                                                               | bench_wiki.wiki_embedding_chunks.page_id, bench_wiki.wiki_embedding_chunks.title, vec_cosine_distance(bench_wiki.wiki_embedding_chunks.embedding, [-1.5,1.7,3.8,-2.8,-1,(2043 more)...])->Column#12                                             |
| └─Projection_27                        | 10.00   | root         |                                                               | bench_wiki.wiki_embedding_chunks.page_id, bench_wiki.wiki_embedding_chunks.title, bench_wiki.wiki_embedding_chunks.embedding                                                                                                                    |
|   └─TopN_12                            | 10.00   | root         |                                                               | Column#14, offset:0, count:10                                                                                                                                                                                                                   |
|     └─Projection_28                    | 10.00   | root         |                                                               | bench_wiki.wiki_embedding_chunks.page_id, bench_wiki.wiki_embedding_chunks.title, bench_wiki.wiki_embedding_chunks.embedding, vec_cosine_distance(bench_wiki.wiki_embedding_chunks.embedding, [-1.5,1.7,3.8,-2.8,-1,(2043 more)...])->Column#14 |
|       └─TableReader_24                 | 10.00   | root         |                                                               | MppVersion: 2, data:ExchangeSender_23                                                                                                                                                                                                           |
|         └─ExchangeSender_23            | 10.00   | mpp[tiflash] |                                                               | ExchangeType: PassThrough                                                                                                                                                                                                                       |
|           └─Projection_25              | 10.00   | mpp[tiflash] |                                                               | bench_wiki.wiki_embedding_chunks.page_id, bench_wiki.wiki_embedding_chunks.title, bench_wiki.wiki_embedding_chunks.embedding                                                                                                                    |
|             └─TopN_22                  | 10.00   | mpp[tiflash] |                                                               | Column#13, offset:0, count:10                                                                                                                                                                                                                   |
|               └─Projection_26          | 10.00   | mpp[tiflash] |                                                               | bench_wiki.wiki_embedding_chunks.page_id, bench_wiki.wiki_embedding_chunks.title, bench_wiki.wiki_embedding_chunks.embedding, vec_cosine_distance(bench_wiki.wiki_embedding_chunks.embedding, [-1.5,1.7,3.8,-2.8,-1,(2043 more)...])->Column#13 |
|                 └─TableFullScan_21     | 10.00   | mpp[tiflash] | table:c, index:idx_wiki_embedding_chunks_embedding(embedding) | keep order:false, annIndex:COSINE(embedding..[-1.5,1.7,3.8,-2.8,-1,(2043 more)...], limit:10)                                                                                                                                                   |
+----------------------------------------+---------+--------------+---------------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

:::

なお、件数が少ないうちは、ヒントなしだとオプティマイザがTiKV全件スキャンを選ぶことがあります。経路を比較する計測では、`READ_FROM_STORAGE`ヒントでストレージエンジンを固定します。

### 計測結果

計測方法は次の通りです。

- クライアントは手元のPC(Tailscale経由)です。PyMySQLの持続接続を使い、mysqlコマンドの起動・接続のオーバーヘッドを除外します
- 意味の異なる20クエリを用意し、それぞれを`mode=query`で埋め込み、3パターンを各1回実行します。レイテンシの分布はこの20サンプルから取ります
- 計測前に、計測対象外のクエリで各経路を1回ウォームアップします

同一クエリの繰り返しで試行回数を稼がないのには理由があります。最初は同一クエリ20回で計測したところ、TiKV全件スキャンがp50 9.6msという不自然に速い値になりました。原因はTiDBのcoprocessor cacheで、同じ`EXPLAIN ANALYZE`を2回続けて実行すると確認できます。

```bash
# 同じクエリのEXPLAIN ANALYZEを2回続けて打つ
mysql -h tidb.${TAILNET} -P 4000 -u root bench_wiki -e "
EXPLAIN ANALYZE SELECT /*+ READ_FROM_STORAGE(TIKV[c]) */ page_id, title,
       VEC_COSINE_DISTANCE(embedding, '${QVEC}') AS distance
  FROM wiki_embedding_chunks AS c
 ORDER BY VEC_COSINE_DISTANCE(embedding, '${QVEC}')
 LIMIT 10;"
```

1回目は`copr_cache_hit_ratio: 0.00`で全体174.5ms(TiKVが10,017キーを172msかけて処理)、2回目は`copr_cache_hit_ratio: 1.00`で全体1.58msです。2回目はTiKVで距離計算をしておらず、TiDBがキャッシュ済みのTopN結果を返しているだけです。TiFlash(MPP)経路はこのキャッシュを使わないため、同一クエリの繰り返しではTiKVだけが不当に速く見えます。実際のベクトル検索ではクエリごとに埋め込みが異なり、このキャッシュに乗りません。そのため、実態に合わせて「異なる20クエリ×各1回」で計測します。

:::details EXPLAIN ANALYZE全文(1回目: copr_cache_hit_ratio 0.00)

```text
+----------------------------------+----------+---------+-----------+---------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------+---------+
| id                               | estRows  | actRows | task      | access object | execution info                                                                                                                                                                                                                                                                                                                                                                                                             | operator info                                                                                                                                                                                                                                   | memory  | disk    |
+----------------------------------+----------+---------+-----------+---------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------+---------+
| Projection_8                     | 10.00    | 10      | root      |               | time:174.5ms, loops:2, RU:1764.60, Concurrency:OFF                                                                                                                                                                                                                                                                                                                                                                         | bench_wiki.wiki_embedding_chunks.page_id, bench_wiki.wiki_embedding_chunks.title, vec_cosine_distance(bench_wiki.wiki_embedding_chunks.embedding, [-1.5,1.7,3.8,-2.8,-1,(2043 more)...])->Column#12                                             | 97.4 KB | N/A     |
| └─Projection_16                  | 10.00    | 10      | root      |               | time:174.4ms, loops:2, Concurrency:OFF                                                                                                                                                                                                                                                                                                                                                                                     | bench_wiki.wiki_embedding_chunks.page_id, bench_wiki.wiki_embedding_chunks.title, bench_wiki.wiki_embedding_chunks.embedding                                                                                                                    | 97.7 KB | N/A     |
|   └─TopN_9                       | 10.00    | 10      | root      |               | time:174.4ms, loops:2                                                                                                                                                                                                                                                                                                                                                                                                      | Column#13, offset:0, count:10                                                                                                                                                                                                                   | 81.2 KB | 0 Bytes |
|     └─Projection_17              | 10.00    | 10      | root      |               | time:174.3ms, loops:2, Concurrency:OFF                                                                                                                                                                                                                                                                                                                                                                                     | bench_wiki.wiki_embedding_chunks.page_id, bench_wiki.wiki_embedding_chunks.title, bench_wiki.wiki_embedding_chunks.embedding, vec_cosine_distance(bench_wiki.wiki_embedding_chunks.embedding, [-1.5,1.7,3.8,-2.8,-1,(2043 more)...])->Column#13 | 80.7 KB | N/A     |
|       └─TableReader_15           | 10.00    | 10      | root      |               | time:174.1ms, loops:2, cop_task: {num: 1, max: 174ms, proc_keys: 10017, tot_proc: 172ms, tot_wait: 646.5µs, copr_cache_hit_ratio: 0.00, build_task_duration: 4.05µs, max_distsql_concurrency: 1}, rpc_info:{Cop:{num_rpc:1, total_time:173.9ms}}                                                                                                                                                                           | data:TopN_14                                                                                                                                                                                                                                    | 80.7 KB | N/A     |
|         └─TopN_14                | 10.00    | 10      | cop[tikv] |               | tikv_task:{time:172ms, loops:10}, scan_detail: {total_process_keys: 10017, total_process_keys_size: 111856352, total_keys: 10018, get_snapshot_time: 477.1µs, rocksdb: {delete_skipped_count: 1832, key_skipped_count: 21865, block: {cache_hit_count: 2350}}}, time_detail: {total_process_time: 172ms, total_suspend_time: 137.1µs, total_wait_time: 646.5µs, total_kv_read_wall_time: 74ms, tikv_wall_time: 172.8ms}    | vec_cosine_distance(bench_wiki.wiki_embedding_chunks.embedding, [-1.5,1.7,3.8,-2.8,-1,(2043 more)...]), offset:0, count:10                                                                                                                      | N/A     | N/A     |
|           └─TableFullScan_13     | 10017.00 | 10017   | cop[tikv] | table:c       | tikv_task:{time:74ms, loops:10}                                                                                                                                                                                                                                                                                                                                                                                            | keep order:false                                                                                                                                                                                                                                | N/A     | N/A     |
+----------------------------------+----------+---------+-----------+---------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------+---------+
```

:::

:::details EXPLAIN ANALYZE全文(2回目: copr_cache_hit_ratio 1.00)

```text
+----------------------------------+----------+---------+-----------+---------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------+---------+
| id                               | estRows  | actRows | task      | access object | execution info                                                                                                                                                                                                                                  | operator info                                                                                                                                                                                                                                   | memory  | disk    |
+----------------------------------+----------+---------+-----------+---------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------+---------+
| Projection_8                     | 10.00    | 10      | root      |               | time:1.58ms, loops:2, RU:0.48, Concurrency:OFF                                                                                                                                                                                                  | bench_wiki.wiki_embedding_chunks.page_id, bench_wiki.wiki_embedding_chunks.title, vec_cosine_distance(bench_wiki.wiki_embedding_chunks.embedding, [-1.5,1.7,3.8,-2.8,-1,(2043 more)...])->Column#12                                             | 97.4 KB | N/A     |
| └─Projection_16                  | 10.00    | 10      | root      |               | time:1.51ms, loops:2, Concurrency:OFF                                                                                                                                                                                                           | bench_wiki.wiki_embedding_chunks.page_id, bench_wiki.wiki_embedding_chunks.title, bench_wiki.wiki_embedding_chunks.embedding                                                                                                                    | 97.7 KB | N/A     |
|   └─TopN_9                       | 10.00    | 10      | root      |               | time:1.51ms, loops:2                                                                                                                                                                                                                            | Column#13, offset:0, count:10                                                                                                                                                                                                                   | 81.2 KB | 0 Bytes |
|     └─Projection_17              | 10.00    | 10      | root      |               | time:1.45ms, loops:2, Concurrency:OFF                                                                                                                                                                                                           | bench_wiki.wiki_embedding_chunks.page_id, bench_wiki.wiki_embedding_chunks.title, bench_wiki.wiki_embedding_chunks.embedding, vec_cosine_distance(bench_wiki.wiki_embedding_chunks.embedding, [-1.5,1.7,3.8,-2.8,-1,(2043 more)...])->Column#13 | 80.7 KB | N/A     |
|       └─TableReader_15           | 10.00    | 10      | root      |               | time:1.36ms, loops:2, cop_task: {num: 1, max: 1.27ms, proc_keys: 0, tot_proc: 2.44µs, tot_wait: 619.9µs, copr_cache_hit_ratio: 1.00, build_task_duration: 6.81µs, max_distsql_concurrency: 1}, rpc_info:{Cop:{num_rpc:1, total_time:1.25ms}}    | data:TopN_14                                                                                                                                                                                                                                    | 80.6 KB | N/A     |
|         └─TopN_14                | 10.00    | 10      | cop[tikv] |               | tikv_task:{time:172ms, loops:10}, scan_detail: {get_snapshot_time: 580.1µs, rocksdb: {block: {}}}, time_detail: {total_process_time: 2.44µs, total_wait_time: 619.9µs, tikv_wall_time: 785.3µs}                                                 | vec_cosine_distance(bench_wiki.wiki_embedding_chunks.embedding, [-1.5,1.7,3.8,-2.8,-1,(2043 more)...]), offset:0, count:10                                                                                                                      | N/A     | N/A     |
|           └─TableFullScan_13     | 10017.00 | 10017   | cop[tikv] | table:c       | tikv_task:{time:74ms, loops:10}                                                                                                                                                                                                                 | keep order:false                                                                                                                                                                                                                                | N/A     | N/A     |
+----------------------------------+----------+---------+-----------+---------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------+---------+
```

:::

:::details 計測スクリプト(bench_search.py)

```python:bench_search.py
# /// script
# requires-python = ">=3.11"
# dependencies = ["pymysql>=1.1"]
# ///
"""TiDBベクトル検索の3経路レイテンシとrecall@10を計測する。

同一クエリを繰り返すとTiKV経路がTiDBのcoprocessor cacheに乗って
実測にならないため、異なる20クエリを各1回ずつ流して分布を取る。

usage: TAILNET=... uv run bench_search.py
"""
import json
import math
import os
import time
import urllib.request

import pymysql

TAILNET = os.environ["TAILNET"]
EMBED_URL = f"http://plamo-embedding.{TAILNET}/embed"
QUERIES = [
    "日本の城の石垣の構造",
    "相対性理論における時間の遅れ",
    "江戸時代の参勤交代の制度",
    "ワクチンで免疫がつく仕組み",
    "新幹線が開業するまでの歴史",
    "火山の噴火が起きる仕組み",
    "平安時代の貴族の暮らし",
    "サッカーのオフサイドのルール",
    "株式会社と合同会社の違い",
    "太陽系の惑星の並び順",
    "発酵食品と微生物の関係",
    "CPUが計算を実行する仕組み",
    "織田信長と本能寺の変",
    "地震の震度とマグニチュードの違い",
    "世界遺産に登録される条件",
    "オリンピックの開催地の決め方",
    "日本酒の醸造工程",
    "遺伝子の突然変異と進化",
    "気候変動が農業に与える影響",
    "印象派の画家と代表作",
]
# (hint, ORDER BY式)。tiflash_exactの「+ 0」はANNインデックス回避(本文参照)
PATTERNS = {
    "tikv_full":     ("READ_FROM_STORAGE(TIKV[c])",    "VEC_COSINE_DISTANCE(embedding, %s)"),
    "tiflash_exact": ("READ_FROM_STORAGE(TIFLASH[c])", "VEC_COSINE_DISTANCE(embedding, %s) + 0"),
    "tiflash_ann":   ("READ_FROM_STORAGE(TIFLASH[c])", "VEC_COSINE_DISTANCE(embedding, %s)"),
}


def embed_query(text: str) -> tuple[str, float]:
    body = json.dumps({"text": text, "mode": "query"}).encode()
    req = urllib.request.Request(EMBED_URL, data=body,
                                 headers={"Content-Type": "application/json"})
    t0 = time.perf_counter()
    vec = json.load(urllib.request.urlopen(req, timeout=60))["vector"]
    return json.dumps(vec), time.perf_counter() - t0


def search(cur, pattern: str, qvec: str) -> tuple[tuple, float]:
    hint, order_by = PATTERNS[pattern]
    sql = (f"SELECT /*+ {hint} */ chunk_id, title, "
           f"VEC_COSINE_DISTANCE(embedding, %s) AS distance "
           f"FROM wiki_embedding_chunks AS c "
           f"ORDER BY {order_by} LIMIT 10")
    t0 = time.perf_counter()
    cur.execute(sql, [qvec] * sql.count("%s"))
    return cur.fetchall(), time.perf_counter() - t0


def stats(label: str, samples: list[float]) -> None:
    s = sorted(samples)
    n = len(s)
    p50 = s[math.ceil(0.50 * n) - 1] * 1000
    p95 = s[math.ceil(0.95 * n) - 1] * 1000
    print(f"{label:14s} min={s[0] * 1000:6.1f}ms p50={p50:6.1f}ms "
          f"p95={p95:6.1f}ms max={s[-1] * 1000:6.1f}ms")


def main() -> None:
    conn = pymysql.connect(host=f"tidb.{TAILNET}", port=4000, user="root",
                           database="bench_wiki")
    cur = conn.cursor()

    # ウォームアップ(計測対象外のクエリで各経路を一度ずつ温める)
    warm, _ = embed_query("ウォームアップ用のクエリ")
    for pattern in PATTERNS:
        search(cur, pattern, warm)

    embed_times: list[float] = []
    times: dict[str, list[float]] = {p: [] for p in PATTERNS}
    recalls: list[float] = []
    for q in QUERIES:
        qvec, embed_sec = embed_query(q)
        embed_times.append(embed_sec)
        rows = {}
        for pattern in PATTERNS:
            rows[pattern], sec = search(cur, pattern, qvec)
            times[pattern].append(sec)
        exact = {r[0] for r in rows["tiflash_exact"]}
        ann = {r[0] for r in rows["tiflash_ann"]}
        recalls.append(len(exact & ann) / len(exact))
        print(f"recall@10={recalls[-1]:.2f} {q}")

    print(f"mean recall@10={sum(recalls) / len(recalls):.2f} (n={len(QUERIES)})")
    stats("query_embed", embed_times)
    for pattern in PATTERNS:
        stats(pattern, times[pattern])


if __name__ == "__main__":
    main()
```

:::

実行と結果です(20クエリの分布)。

```bash
TAILNET=$(tailscale status --json | jq -r '.MagicDNSSuffix') uv run bench_search.py
```

| 検索経路 | min | p50 | p95 | max |
| --- | --: | --: | --: | --: |
| パターン1: TiKV全件スキャン | 179.2ms | 187.7ms | 193.7ms | 194.1ms |
| パターン2: TiFlash全件スキャン | 68.8ms | 73.4ms | 130.0ms | 155.2ms |
| パターン3: ANN(HNSWインデックス) | 17.1ms | 21.7ms | 27.7ms | 30.5ms |

検索とは別に、前段で毎回発生するクエリ埋め込み(PLaMoのCPU推論)も同じ20回で計測しました。E2Eではこちらが支配項になります。

| 前処理 | min | p50 | p95 | max |
| --- | --: | --: | --: | --: |
| クエリ埋め込み(PLaMo `mode=query`) | 710.7ms | 764.9ms | 857.7ms | 871.8ms |

recall@10(パターン2の全件スキャンのtop10を正解としたパターン3の一致率)は、**20クエリ平均0.83**でした。ばらつきは0.50〜1.00で、クエリによっては10件中5件が正解と入れ替わります。

recall@10が測っているのは「全件スキャンが返す真の近傍top10のうち、ANNが何件を見つけられたか」というインデックスの忠実度で、検索結果が意味的にヒットしているかどうかとは別物です(そちらは検索精度の章で見ます)。取りこぼしが多かったクエリの1つ「日本の城の石垣の構造」(0.60)で、全件スキャンとANNのtop10を突き合わせると何が起きているかが見えます。

:::details 全件スキャンとANNのtop10の突き合わせ(recall@10=0.60のクエリ)

パターン2と3のSQLの取得列を変えた2本です(`${QVEC}`はクエリベクトルの生成の節と同じ「日本の城の石垣の構造」)。

```bash
# 全件スキャン(真の近傍top10 = 正解)
mysql -h tidb.${TAILNET} -P 4000 -u root bench_wiki -e "
SELECT /*+ READ_FROM_STORAGE(TIFLASH[c]) */ LEFT(chunk_id,8) AS id, title, chunk_index,
       ROUND(VEC_COSINE_DISTANCE(embedding, '${QVEC}'),4) AS distance
  FROM wiki_embedding_chunks AS c
 ORDER BY VEC_COSINE_DISTANCE(embedding, '${QVEC}') + 0
 LIMIT 10;"

# ANNのtop10
mysql -h tidb.${TAILNET} -P 4000 -u root bench_wiki -e "
SELECT /*+ READ_FROM_STORAGE(TIFLASH[c]) */ LEFT(chunk_id,8) AS id, title, chunk_index,
       ROUND(VEC_COSINE_DISTANCE(embedding, '${QVEC}'),4) AS distance
  FROM wiki_embedding_chunks AS c
 ORDER BY VEC_COSINE_DISTANCE(embedding, '${QVEC}')
 LIMIT 10;"
```

結果を突き合わせたものです。

```text
全件スキャン top10(正解)             ANNで見つかったか
 1  0.3995  スラブ (chunk 0)             ✓
 2  0.4055  ジグザグ橋 (chunk 0)         ✓
 3  0.4118  走向 (chunk 2)               ✗ 取りこぼし
 4  0.4134  秋月氏 (chunk 3)             ✗ 取りこぼし
 5  0.4220  北山安夫 (chunk 1)           ✓
 6  0.4225  切山城 (chunk 1)             ✗ 取りこぼし
 7  0.4227  桜田門 (chunk 0)             ✓
 8  0.4243  鎹 (chunk 0)                 ✓
 9  0.4249  桜田門 (chunk 2)             ✓
10  0.4253  走向 (chunk 1)               ✗ 取りこぼし

代わりにANNのtop10へ繰り上がったもの(真の11位以下)
    0.4269  廊下 (漁場建築) (chunk 1)
    0.4323  由良川 (鳥取県) (chunk 0)
    0.4373  桜田門 (chunk 1)
    0.4384  更上閣 (chunk 0)
```

一致は6件で6/10 = 0.60です。ANNは真の3位・4位・6位・10位を見つけ損ね、代わりに真の11位以下が繰り上がっています。ANN(HNSWインデックス)は全件と距離計算をせず、グラフの繋がりを辿って「近そうな方向」を探しに行くため、このクエリのようにtop10が0.40〜0.43の僅差に密集していると、辿った経路から漏れる点が出ます。逆にrecall@10=1.00だったクエリは本命が0.2台で明確に近く、グラフを辿れば迷わず到達できる形です。

:::

:::details 実行ログ(recall@10の個別値)

```text
recall@10=0.60 日本の城の石垣の構造
recall@10=0.80 相対性理論における時間の遅れ
recall@10=0.70 江戸時代の参勤交代の制度
recall@10=0.80 ワクチンで免疫がつく仕組み
recall@10=0.90 新幹線が開業するまでの歴史
recall@10=0.80 火山の噴火が起きる仕組み
recall@10=0.60 平安時代の貴族の暮らし
recall@10=0.90 サッカーのオフサイドのルール
recall@10=0.90 株式会社と合同会社の違い
recall@10=1.00 太陽系の惑星の並び順
recall@10=1.00 発酵食品と微生物の関係
recall@10=0.90 CPUが計算を実行する仕組み
recall@10=0.70 織田信長と本能寺の変
recall@10=0.90 地震の震度とマグニチュードの違い
recall@10=0.50 世界遺産に登録される条件
recall@10=0.90 オリンピックの開催地の決め方
recall@10=1.00 日本酒の醸造工程
recall@10=1.00 遺伝子の突然変異と進化
recall@10=0.90 気候変動が農業に与える影響
recall@10=0.80 印象派の画家と代表作
mean recall@10=0.83 (n=20)
```

:::

この実測から得られた、1万チャンク時点の結論は「**HNSWインデックスはまだ要らない**」です。

- 検索単体はどの経路も実用域です。最も遅いTiKV全件スキャンでもp50 187.7msで、E2Eの支配項はクエリ埋め込みのp50 765ms(ANN検索を足してもE2Eは1秒弱)にあるため、経路差の最大約170msは体感にほぼ効きません
- ANNで稼げるその約170msには、recall@10=0.83の取りこぼしという代償が付きます。この規模では割に合わず、近似なしの全件スキャンで十分です(取りこぼしが実害になる実例は検索精度の章で示します)
- インデックスの真価が問われるのはデータを増やしてからです。全件スキャンの2経路は行数に線形に伸びるため、単純比例なら10万チャンクで全件スキャンが0.7秒(TiFlash)〜1.9秒(TiKV)、100万チャンクで7〜19秒と実用域を外れていき、ここで初めてHNSWインデックスが必須になります

## 検索精度

レイテンシに続いて、意図した記事にヒットするかを確認します。ヒットの評価はANNの取りこぼし(recall@10=0.83)と切り分けたいので、正解はTiFlash全件スキャン(パターン2の`+ 0`付きSQL)で作り、章の最後にANNでもヒットが変わらないかを確認します。

### 言い換え・表記揺れクエリの実例

セマンティック検索の価値は、キーワードが一致しないクエリでも目的の記事にヒットすることです。ヒットさせたい記事のタイトルの語を使わない言い換えクエリを12本用意し、それぞれtop3(タイトル・チャンク番号・distance・本文冒頭)を確認しました。

:::details 検証スクリプト(eval_quality.py)

```python:eval_quality.py
# /// script
# requires-python = ">=3.11"
# dependencies = ["pymysql>=1.1"]
# ///
"""言い換え・表記揺れクエリの検索精度を目視確認する。

各クエリをTiFlash全件スキャン(`+ 0`付き)でtop3まで取り、
タイトル・chunk_index・distance・本文冒頭を表示する。
top1がchunk_index > 0の場合は、同じページの先頭チャンク(chunk_index=0)の
distanceも表示する(「1記事1ベクトル(先頭1024トークン)相当」との比較用)。

usage: TAILNET=... uv run eval_quality.py
"""
import json
import os
import time
import urllib.request

import pymysql

TAILNET = os.environ["TAILNET"]
EMBED_URL = f"http://plamo-embedding.{TAILNET}/embed"
QUERIES = [
    # 言い換え(タイトルの語を使わない)
    "地球の表面の7割を覆う塩水の領域",
    "都市ガスの原料になる化石燃料",
    "人間の背の高さの平均や測り方",
    "火星にある太陽系最大級の峡谷",
    "マルクスとともに資本主義を批判した思想家",
    "生まれつき顔の筋肉が動かず表情が作れない難病",
    "ヤードポンド法で使われる体積の単位",
    "サンフランシスコ湾岸地域の交通系ICカード",
    "家庭で使った水の量を測る器具",
    "将棋で光速の寄せと呼ばれた棋士",
    "異なる品種を掛け合わせること",
    # 表記揺れ(ベニス⇔ヴェネツィア)
    "ベニス共和国と東ローマ帝国の戦い",
]

SQL_TOP3 = """
SELECT /*+ READ_FROM_STORAGE(TIFLASH[c]) */
       page_id, title, chunk_index, LEFT(content, 90) AS head,
       VEC_COSINE_DISTANCE(embedding, %s) AS distance
  FROM wiki_embedding_chunks AS c
 ORDER BY VEC_COSINE_DISTANCE(embedding, %s) + 0
 LIMIT 3
"""

SQL_CHUNK0 = """
SELECT /*+ READ_FROM_STORAGE(TIFLASH[c]) */
       VEC_COSINE_DISTANCE(embedding, %s) AS distance
  FROM wiki_embedding_chunks AS c
 WHERE page_id = %s AND chunk_index = 0
"""


def embed_query(text: str) -> str:
    body = json.dumps({"text": text, "mode": "query"}).encode()
    req = urllib.request.Request(EMBED_URL, data=body,
                                 headers={"Content-Type": "application/json"})
    return json.dumps(json.load(urllib.request.urlopen(req, timeout=60))["vector"])


def main() -> None:
    conn = pymysql.connect(host=f"tidb.{TAILNET}", port=4000, user="root",
                           database="bench_wiki")
    cur = conn.cursor()
    for q in QUERIES:
        qvec = embed_query(q)
        cur.execute(SQL_TOP3, [qvec, qvec])
        rows = cur.fetchall()
        print(f"\n## {q}")
        for page_id, title, chunk_index, head, distance in rows:
            head = head.replace("\n", " ")
            print(f"  {distance:.4f}  {title} (chunk {chunk_index})  | {head}")
        top = rows[0]
        if top[2] > 0:
            cur.execute(SQL_CHUNK0, [qvec, top[0]])
            d0 = cur.fetchone()[0]
            print(f"  -> top1と同ページの先頭チャンク(chunk 0): {d0:.4f} "
                  f"(チャンク方式との差 {d0 - top[4]:+.4f})")


if __name__ == "__main__":
    main()
```

:::

結果は、**12クエリ中11クエリで意図した記事がtop1**でした。外れた1つ(「マルクスとともに資本主義を批判した思想家」)も、top1は強く関連する「第一インターナショナル創立宣言」で、本命のフリードリヒ・エンゲルスがtop2に入ります。代表例を抜粋します。

| クエリ | top1 (distance) | 補足 |
| --- | --- | --- |
| 家庭で使った水の量を測る器具 | 水道メーター (0.2572) | クエリに「水道」も「メーター」もなし |
| ベニス共和国と東ローマ帝国の戦い | ヴェネツィア・東ローマ戦争 (1118年) (0.2554) | ベニス⇔ヴェネツィアの表記揺れを吸収 |
| ヤードポンド法で使われる体積の単位 | ガロン (0.2708) | |
| 地球の表面の7割を覆う塩水の領域 | 海 (0.2728) | |
| 異なる品種を掛け合わせること | 交雑 (0.3302) | 記事側の表現は「交雑」「異種交配」 |
| 生まれつき顔の筋肉が動かず表情が作れない難病 | メビウス症候群 (0.3677) | 症状の描写だけで病名に到達 |

今回の観測範囲では、意図した記事はdistance 0.21〜0.37に収まり、無関係な記事はおおむね0.4より遠いという分布でした。

:::details 12クエリの実行ログ全文

```text
$ TAILNET=$(tailscale status --json | jq -r '.MagicDNSSuffix') uv run eval_quality.py

## 地球の表面の7割を覆う塩水の領域
  0.2728  海 (chunk 0)  | 海（うみ、英: the sea または the ocean）は、地球上の陸地以外の部分で、海水に満たされたところ。 大小さまざまな広がり方があり、特に大きな広がり（だけ）は海洋（t
  0.3106  海 (chunk 2)  | 一般的な海域においては海からの水分や風が雨をもたらし降水量が多くなる傾向がある。寒流が流れる海岸部においては、上層の空気は暖かいのに対し、下層の空気は寒流によって冷やされるため、上
  0.3114  海 (chunk 9)  | た地名が数多く存在する。ただし、火星には地質時代には海があった可能性がある。 木星や土星の氷衛星のいくつかは、氷の地殻の下に液体の水の海があると推測されている。エウロパ、ガニメデ、

## 都市ガスの原料になる化石燃料
  0.2823  天然ガス (chunk 7)  | れた天然ガスは、-113℃以上に暖められるまでは空気よりも重いため、極低温のガスが地上に滞留する。LNGタンクが作られた初期の1944年10月20日、アメリカ合衆国のオハイオ州クリ
  0.3019  天然ガス (chunk 0)  | 天然ガス（てんねんガス、英: natural gas）とは、メタンを主成分とし、エタンやプロパンなどを含む化石燃料の一種。 気体燃料は天然ガス、石炭系ガス（石炭ガス、水性ガス、発生
  0.3168  天然ガス (chunk 10)  | 新聞』朝刊2018年12月4日（企業3面）2018年12月21日閲覧。 ^ Johnson, Jeff. LNG WEIGHS ANCHOR. Chemical & Enginee
  -> top1と同ページの先頭チャンク(chunk 0): 0.3019 (チャンク方式との差 +0.0195)

## 人間の背の高さの平均や測り方
  0.2114  身長 (chunk 0)  | 身長（しんちょう）は、人間（ヒト）が直立した時の、床又は地面から頭頂までの高さ。身の丈（みのたけ）、上背（うわぜい）、背丈（せたけ）、立っ端（たっぱ）とも言う。通常、メートル法を使
  0.2684  身長 (chunk 8)  | にもより、中年期から始まる人もいるが、高齢者には普遍的な傾向が見られる。この身長の低下は、乾燥による椎間板の高さの低下、軟部組織の萎縮、変性疾患に伴う姿勢の変化などが原因である。2
  0.2775  身長 (chunk 5)  | 。 先行人類ではホモ・ハイデルベルゲンシスの身長が男性で175 cm、女性で157 cmくらいと推定されている。また、ホモ・ネアンデルターレンシスの身長は男性で166 cm、女性で

## 火星にある太陽系最大級の峡谷
  0.2724  マリネリス峡谷 (chunk 1)  | マッチ記念ステーション→ ジェラルド・ソッフェン記念ステーション→ 立体的に見たマリネリス峡谷 マリネリス峡谷の標高図 メラス谷の立体モデル 峡谷内のクレーターで見つかった塩水の流
  0.2779  マリネリス峡谷 (chunk 0)  | マリネリス峡谷（ラテン語: Valles Marineris ヴァリス・マリネリス）とは、火星の赤道に沿って伸びる巨大な峡谷。マリナー峡谷（英: Mariner Valleys）と
  0.3686  ノクティス迷路 (chunk 0)  | ノクティス迷路（英: Noctis Labyrinthus; "the labyrinth of the night" 夜の迷路）は、火星のマリネリス峡谷とタルシス地域の境にある迷
  -> top1と同ページの先頭チャンク(chunk 0): 0.2779 (チャンク方式との差 +0.0056)

## マルクスとともに資本主義を批判した思想家
  0.3743  第一インターナショナル創立宣言 (chunk 3)  | 大月書店、1959年。  ハリンリヒ・ゲムコー（ドイツ語版）,マルクス＝レーニン主義研究所 著、土屋保男,松本洋子 訳『フリードリヒ・エンゲルス 一伝記(上)、(下)』大月書店、1
  0.3750  フリードリヒ・エンゲルス (chunk 77)  | ・イデオロギー』（マルクスとの共著）(ドイツ語: Die deutsche Ideologie, (mit Marx) 1845) 『共産主義の原理』(ドイツ語: Grundsät
  0.3791  フリードリヒ・エンゲルス (chunk 87)  | 大月書店、1976年。  大内兵衛『マルクス・エンゲルス小伝』岩波書店、1964年。  トリストラム・ハント（英語版） 著、東郷えりか 訳『エンゲルス: マルクスに将軍と呼ばれた男
  -> top1と同ページの先頭チャンク(chunk 0): 0.4953 (チャンク方式との差 +0.1210)

## 生まれつき顔の筋肉が動かず表情が作れない難病
  0.3677  メビウス症候群 (chunk 0)  | メビウス症候群（メビウスしょうこうぐん、Möbius (或は Moebius) syndrome）は、非常に稀な神経異常。 メビウス症候群は、1888年に神経学者のパウル・メビウス
  0.4178  キャシー中島 (chunk 7)  | でも涙は見せたくない（その12）”. 産経新聞のウェブサイト (2016年7月18日). 2023年6月28日閲覧。 ^ “キャシー中島、皮膚がんの手術を受けていた…右目の下、テー
  0.4355  斎藤洋介 (chunk 6)  | 旭化成ホームズ/へーベルハウス「FREX-3」「DUET」(1987年) トヨタ実写版ドラえもん - 教習所指導員 ソニースタミナハンディカム(内藤剛志と共演) サークルK ヴィク

## ヤードポンド法で使われる体積の単位
  0.2708  ガロン (chunk 1)  | 穀物用 1824年、イギリスはエールガロンに近い値を英ガロンとして採用した。リットルが1キログラムの水の体積として定義されたのに影響を受けて、英ガロンは 10 ポンドの水の体積と定
  0.2762  ガロン (chunk 0)  | ガロン（gallon, 記号: gal）は、ヤード・ポンド法の体積の計量単位である。 国や用途によって各種のガロンの定義があるが、3.7 リットルから 4.6 リットルの範囲内にあ
  0.3010  ガロン (chunk 3)  | ^ 明治酪農牛乳容量の欄、沖縄明治乳業株式会社 ^ “沖縄の牛乳パックは「946ml」”. 沖縄おもしろミニ情報. 2018年4月21日時点のオリジナルよりアーカイブ。2018年4
  -> top1と同ページの先頭チャンク(chunk 0): 0.2762 (チャンク方式との差 +0.0053)

## サンフランシスコ湾岸地域の交通系ICカード
  0.2572  クリッパーカード (chunk 0)  | クリッパーカード（Clipper Card）は、サンフランシスコ・ベイエリアで利用できる公共交通機関向け非接触型ICカードである。2021年現在、ベイエリアの24の公共交通機関で利
  0.2878  クリッパーカード (chunk 1)  | .org/web/20110504231738/http://www.mtc.ca.gov/news/current_topics/10-10/clipper_chinese.ht
  0.3439  サウスベイ (サンフランシスコ・ベイエリア) (chunk 1)  | 歴史公園、博物館、セントラルサンノゼ サンノゼ子供発見博物館、サンノゼ中心街 ノーマン・Y・ミネタ・サンノゼ国際空港、サンノゼ市 サンノゼ美術館、サンノゼ市 サンノゼ州立大学、サン

## 家庭で使った水の量を測る器具
  0.2572  水道メーター (chunk 0)  | 水道メーター（すいどうメーター）とは、水道での水の使用量を記録するための計器。検針や点検の際に用いる。量水器（りょうすいき）ともよばれる。 使用水量を立方メートル単位で3桁あるいは
  0.4062  ガロン (chunk 2)  | この定義値は、米国の（リットルによる）定義値の小数7桁目を四捨五入したものである。 会員制販売店のコストコでは、ガロン容量で販売している飲料やエンジンオイルが発売されているが、詳細
  0.4120  酸素濃度計 (chunk 1)  | 装置 食品包装 焼成炉内組込 グローブボックス 医療機器 排気ガスの分析 エンジンの制御 環境計測 窯業 ダイビング 東レエンジニアリング 新コスモス電機 横河電機 ^ a b c

## 将棋で光速の寄せと呼ばれた棋士
  0.3038  谷川浩司 (chunk 21)  | 1月16日時点のオリジナルよりアーカイブ。 ^ 「将棋 谷川浩司十七世名人 史上3人目の通算1400勝を達成」『NHK 日本放送協会』2025年1月15日。2025年1月16日時点
  0.3228  谷川浩司 (chunk 10)  | 率0.610）を達成。2019年1月22日には、第32期竜王戦4組ランキング戦で船江恒平に勝ち、中原誠を超え歴代4位の1309勝を、同年9月12日には、第78期順位戦B級1組で松尾
  0.3328  谷川浩司 (chunk 17)  | 神戸文化栄誉賞 2002年00月 - 神戸市特別表彰 2007年00月 - 兵庫県文化賞 2014年11月 - 紫綬褒章（将棋界12人目の受賞） 光速の寄せ 戦型別終盤の手筋（全5
  -> top1と同ページの先頭チャンク(chunk 0): 0.4070 (チャンク方式との差 +0.1033)

## 異なる品種を掛け合わせること
  0.3302  交雑 (chunk 0)  | 交雑（こうざつ）または異種交配（いしゅこうはい）（英語: crossbreed）とは、生物学においては、異なる種や異なる亜種の関係にある動物・植物を特に人工的に組み合わせて交配させ
  0.3803  カブ (chunk 8)  | ）2015年6月8日閲覧 ^ a b c d e 藤田智監修 NHK出版編 2019, p. 126. ^ a b c d e f g 主婦の友社編 2011, p. 179. ^
  0.3897  カブ (chunk 7)  | いた状態で販売されている事が多い。 ロシアでは、『おおきなかぶ』のように民話の題材になるほど馴染みのある野菜である。一方、カブがあまり好まれないフランスでは、大根役者に相当する「カ

## ベニス共和国と東ローマ帝国の戦い
  0.2554  ヴェネツィア・東ローマ戦争 (1118年) (chunk 0)  | ヴェネツィア・東ローマ戦争は、1118年にヴェネツィア共和国と東ローマ帝国との間で行われた一連の戦いである。ヴェネツィア共和国が勝利をおさめた。 1118年、父の死によって即位した
  0.3167  アンコーナ (chunk 3)  | 20,000m²以上の面積をもつ5角形の建物。船舶と共に市街へすぐ到達する伝染病の危険から軍事防衛幹部を守るため建てられた。のち、軍事病院や兵舎としても使われた・現在は文化展示に用
  0.3261  メンターナ (chunk 1)  | を占領してイタリア王国に組み込もうとしたジュゼッペ・ガリバルディが率いる義勇兵と、教皇軍・フランス軍部隊との間での戦闘が行われた（メンターナの戦い（英語版））。この戦いは、教皇軍・
```

:::

### ANNの取りこぼしは実害になるか

ここまでの評価は全件スキャンでした。実運用で使うのはANNなので、recall@10=0.83の取りこぼしがヒットを壊すのかを、同じ12クエリで両経路のtop1を突き合わせて確認します。

:::details 検証スクリプト(check_ann_top1.py)

```python:check_ann_top1.py
# /// script
# requires-python = ">=3.11"
# dependencies = ["pymysql>=1.1"]
# ///
"""言い換え12クエリを全件スキャン(exact)とANNの両経路で流し、top1が変わるか確認する。"""
import json
import os
import urllib.request

import pymysql

TAILNET = os.environ["TAILNET"]
EMBED_URL = f"http://plamo-embedding.{TAILNET}/embed"
QUERIES = [
    "地球の表面の7割を覆う塩水の領域",
    "都市ガスの原料になる化石燃料",
    "人間の背の高さの平均や測り方",
    "火星にある太陽系最大級の峡谷",
    "マルクスとともに資本主義を批判した思想家",
    "生まれつき顔の筋肉が動かず表情が作れない難病",
    "ヤードポンド法で使われる体積の単位",
    "サンフランシスコ湾岸地域の交通系ICカード",
    "家庭で使った水の量を測る器具",
    "将棋で光速の寄せと呼ばれた棋士",
    "異なる品種を掛け合わせること",
    "ベニス共和国と東ローマ帝国の戦い",
]

SQL = """
SELECT /*+ READ_FROM_STORAGE(TIFLASH[c]) */ title, chunk_index
  FROM wiki_embedding_chunks AS c
 ORDER BY VEC_COSINE_DISTANCE(embedding, %s){plus}
 LIMIT 1
"""


def embed_query(text):
    body = json.dumps({"text": text, "mode": "query"}).encode()
    req = urllib.request.Request(EMBED_URL, data=body,
                                 headers={"Content-Type": "application/json"})
    return json.dumps(json.load(urllib.request.urlopen(req, timeout=60))["vector"])


conn = pymysql.connect(host=f"tidb.{TAILNET}", port=4000, user="root",
                       database="bench_wiki")
cur = conn.cursor()
same = 0
for q in QUERIES:
    qvec = embed_query(q)
    cur.execute(SQL.format(plus=" + 0"), [qvec])
    exact = cur.fetchone()
    cur.execute(SQL.format(plus=""), [qvec])
    ann = cur.fetchone()
    mark = "same" if exact == ann else "DIFF"
    same += exact == ann
    print(f"{mark}  exact={exact[0]}(chunk {exact[1]})  ann={ann[0]}(chunk {ann[1]})  | {q}")
print(f"top1一致: {same}/{len(QUERIES)}")
```

:::

結果は**12クエリ中11でtop1が一致**しました。

| クエリ | 全件スキャンのtop1 | ANNのtop1 | 一致 |
| --- | --- | --- | :-: |
| 地球の表面の7割を覆う塩水の領域 | 海 | 海 | ○ |
| 都市ガスの原料になる化石燃料 | 天然ガス | 天然ガス | ○ |
| 人間の背の高さの平均や測り方 | 身長 | 身長 | ○ |
| 火星にある太陽系最大級の峡谷 | マリネリス峡谷 | マリネリス峡谷 | ○ |
| マルクスとともに資本主義を批判した思想家 | 第一インターナショナル創立宣言 | 第一インターナショナル創立宣言 | ○ |
| 生まれつき顔の筋肉が動かず表情が作れない難病 | メビウス症候群 | メビウス症候群 | ○ |
| ヤードポンド法で使われる体積の単位 | ガロン | ガロン | ○ |
| サンフランシスコ湾岸地域の交通系ICカード | クリッパーカード | クリッパーカード | ○ |
| 家庭で使った水の量を測る器具 | 水道メーター | 水道メーター | ○ |
| 将棋で光速の寄せと呼ばれた棋士 | 谷川浩司 | 谷川浩司 | ○ |
| 異なる品種を掛け合わせること | 交雑 | **カブ** | **×** |
| ベニス共和国と東ローマ帝国の戦い | ヴェネツィア・東ローマ戦争 (1118年) | ヴェネツィア・東ローマ戦争 (1118年) | ○ |

:::details 実行ログ(check_ann_top1.py)

```text
$ TAILNET=$(tailscale status --json | jq -r '.MagicDNSSuffix') uv run check_ann_top1.py
same  exact=海(chunk 0)  ann=海(chunk 0)  | 地球の表面の7割を覆う塩水の領域
same  exact=天然ガス(chunk 7)  ann=天然ガス(chunk 7)  | 都市ガスの原料になる化石燃料
same  exact=身長(chunk 0)  ann=身長(chunk 0)  | 人間の背の高さの平均や測り方
same  exact=マリネリス峡谷(chunk 1)  ann=マリネリス峡谷(chunk 1)  | 火星にある太陽系最大級の峡谷
same  exact=第一インターナショナル創立宣言(chunk 3)  ann=第一インターナショナル創立宣言(chunk 3)  | マルクスとともに資本主義を批判した思想家
same  exact=メビウス症候群(chunk 0)  ann=メビウス症候群(chunk 0)  | 生まれつき顔の筋肉が動かず表情が作れない難病
same  exact=ガロン(chunk 1)  ann=ガロン(chunk 1)  | ヤードポンド法で使われる体積の単位
same  exact=クリッパーカード(chunk 0)  ann=クリッパーカード(chunk 0)  | サンフランシスコ湾岸地域の交通系ICカード
same  exact=水道メーター(chunk 0)  ann=水道メーター(chunk 0)  | 家庭で使った水の量を測る器具
same  exact=谷川浩司(chunk 21)  ann=谷川浩司(chunk 21)  | 将棋で光速の寄せと呼ばれた棋士
DIFF  exact=交雑(chunk 0)  ann=カブ(chunk 8)  | 異なる品種を掛け合わせること
same  exact=ヴェネツィア・東ローマ戦争 (1118年)(chunk 0)  ann=ヴェネツィア・東ローマ戦争 (1118年)(chunk 0)  | ベニス共和国と東ローマ帝国の戦い
top1一致: 11/12
```

:::

本命がはっきり近いクエリでは、取りこぼしが起きても下位の入れ替わりにとどまります。

問題は残る1本です。「異なる品種を掛け合わせること」では、全件スキャンの本命だった交雑(distance 0.3302、2位に0.05差をつけた明確なtop1)が**ANNのtop10から完全に消え**、top1がカブ(0.3803)に入れ替わりました。

:::details 「異なる品種を掛け合わせること」のANN top10(交雑が消えている)

```bash
QVEC=$(curl -s -X POST http://plamo-embedding.${TAILNET}/embed \
  -H 'Content-Type: application/json' \
  -d '{"text":"異なる品種を掛け合わせること","mode":"query"}' | jq -c '.vector')

mysql -h tidb.${TAILNET} -P 4000 -u root bench_wiki -e "
SELECT /*+ READ_FROM_STORAGE(TIFLASH[c]) */ title, chunk_index,
       ROUND(VEC_COSINE_DISTANCE(embedding, '${QVEC}'),4) AS distance
  FROM wiki_embedding_chunks AS c
 ORDER BY VEC_COSINE_DISTANCE(embedding, '${QVEC}')
 LIMIT 10;"
```

```text
カブ        8   0.3803   ← 全件スキャンでは2位。ANNではこれがtop1になる
カブ        7   0.3897
カブ        5   0.3908
ワイン醸造  1   0.3909
カブ        4   0.3966
水耕栽培    0   0.3993
ワイン醸造  11  0.4014
カブ        2   0.4036
カブ        1   0.4115
ワイン醸造  2   0.4155
```

:::

取りこぼしは、計測結果の節で見た「僅差の下位の入れ替わり」だけでなく、まれにこのような「明確な本命の消失」としても現れます。1万チャンクなら全件スキャンが73msで引けるので、正確さが要る場面では全件スキャンを使うのが現実的です(レイテンシの章の結論とも整合します)。

## まとめ

### 実測値と結論

Self-ManagedのTiDBにTiFlashを追加し、PLaMo-Embedding-1BをGPUなしのCPU推論でセルフホストして、日本語Wikipedia 1万チャンク(4,083記事、約824万トークン)のセマンティック検索をアプリケーションDB内で完結させました。本記事で得た実測値の一覧です。

| 項目 | 実測値 |
| --- | --- |
| 埋め込み生成(投入) | 10,017チャンク(約824万トークン) / 9時間40分(0.29チャンク/s ≈ 約240トークン/s、同時4) |
| HNSWインデックス | ビルド7.9秒、79.6MiB(ベクトル生データとほぼ同サイズ) |
| 検索レイテンシ(p50) | ANN 21.7ms < TiFlash全件スキャン 73.4ms < TiKV全件スキャン 187.7ms(速い順に期待通り) |
| 検索E2E(クエリ埋め込み込み) | 1秒弱(支配項はCPU推論の765ms) |
| recall@10(ANN) | 0.83(全件スキャンのtop10のうち平均8.3件が返る) |
| 言い換え12クエリ | 11クエリで意図した記事がtop1 |

結論は、投入と検索で非対称でした。

検索側は実用になります。言い換え・表記揺れに強い検索が1秒弱で返り、同期パイプラインも二重データも持たずに、業務条件の絞り込みをWHERE/JOINでそのまま重ねられます。この規模ならHNSWインデックスは不要で、近似なしの全件スキャンで十分です。ANNにはrecall@10=0.83の取りこぼし(全件スキャンのtop10のうち平均8.3件しか返らない)があり、まれに本命そのものが結果から消えます(実例は検索精度の章)。

一方の投入側は時間との勝負です。CPU推論は実測0.29チャンク/s(約240トークン/s、1日約2.5万チャンク≒2,000万トークン)が上限で、10万チャンクなら約4日、100万チャンクは約40日かかって非現実的です。コーパスの規模が投入計画をそのまま決めます。

### セルフホスト vs Workers AI

PLaMoの章で触れた通り、同じPLaMo-Embedding-1BはCloudflare Workers AI(`@cf/pfnet/plamo-embedding-1b`)でも使えます。[Workers AIの料金表](https://developers.cloudflare.com/workers-ai/platform/pricing/)では$0.019/100万トークン(1,689 Neurons/100万トークン)です。1万チャンク(実測の合計約824万トークン)の埋め込み生成で比べます。

|          | 本構成(CPU推論)                                        | Workers AI                                             |
| -------- | ------------------------------------------------------ | ------------------------------------------------------ |
| 所要時間 | 9時間40分                                              | 分〜数十分(並列度とレート制限次第)                     |
| 費用     | 電気代25〜40円程度[^denki]                             | 約$0.16(約24円)。無料枠(10,000 Neurons/日)の2日分以内 |

[^denki]: 高負荷だったnode2 / node3の2台を、ミニPCの実効消費電力40〜60Wと置いて 2台 × 10時間 × 31円/kWh で概算した値です。

*電気代 25〜40円。API料金 約24円。埋め込みの完了をただ待つ9時間40分、priceless。*

所要時間は大きく異なる一方、費用はほぼ同額です。埋め込みを作ること自体が目的なら、Workers AIを選ばない理由はありません。それでも本構成を選ぶ理由は次の4つで、いずれも「埋め込み生成の効率」とは異なる軸です。

- **データを外部へ送らない**: 本文も検索クエリも外に出ません。社内文書や個人メモなど、外部APIへ投げること自体が論点になるデータで効きます
- **外部API非依存**: クラスタ内で完結するため、APIの仕様変更・障害・レート制限と無縁です。TiDBと同じ運用の輪の中に検索が収まります
- **従量課金なし**: チャンク分割のパラメータを変えて全件入れ直すような試行錯誤を、費用を気にせず回せます
- **検証・学習**: PodのOOMKillやcoprocessor cacheなど、本記事の学びの大半はセルフホストでしか得られませんでした

逆に、この4つに価値を感じないなら、埋め込み生成だけWorkers AIに寄せ、TiDB側(VECTOR + HNSWインデックス)をセルフホストする折衷構成が最もコスパの良い落とし所になるはずです。

### 今後

1万チャンク(日本語Wikipedia 4,083記事、約824万トークン)では「全件スキャンで十分」でした。ただし全件スキャンのレイテンシは行数に線形に伸びるため、単純比例なら10万チャンクで0.7秒(TiFlash)〜1.9秒(TiKV)、100万チャンクでは7〜19秒と実用域を外れ、ここで初めてHNSWインデックスが必須になります。規模感としては、本記事と同じ比率なら10万チャンクが約4万記事、100万チャンクが約40万記事で、後者は日本語Wikipedia全体(約140万記事)の3割弱に相当します。

次の検証は、この10万・100万チャンクの規模で、HNSWのビルド時間・検索レイテンシ・recall@10が1万チャンクの実測値(ビルド7.9秒 / p50 21.7ms / 0.83)からどう変わるかの実測です。ただし、100万チャンクの投入はCPU推論では約40日かかるため、この規模の検証は付録で試算したGPUでの投入(半日〜1日)が前提になります。

## 付録

### 各データベースの日本語全文検索事情

本編で扱ったのはセマンティック検索ですが、検討の過程で調べた「日本語の全文検索を各データベースでやるなら何を使うか」を付録として残しておきます。

| DB | 日本語FTSの手段 |
| --- | --- |
| MySQL | `ngram`パーサ(組み込み、バイグラム) / MeCabパーサプラグイン(公式、形態素解析) |
| PostgreSQL | pg_bigm(バイグラム) / PGroonga + TokenMecab(形態素解析) |
| TiDB | Self-Managedは現状なし。Cloud Starter/EssentialのFULLTEXT INDEX(内蔵パーサ)のみ |
| OpenSearch / Elasticsearch | kuromoji analyzer(形態素解析) |

- kuromojiはJava実装でLucene系(OpenSearch / Elasticsearch / Solr)専用のため、MySQL・PostgreSQL・TiDBには組み込めません。RDB側で形態素解析をやる場合のルートはMeCabです。
- PostgreSQLの組み込みFTS(`tsvector`)は空白区切りの言語が前提で、日本語はそのままではトークナイズできないため、拡張の利用が前提になります。
- MySQLとPostgreSQLはプラグイン・拡張の機構があるため日本語FTSが成立します。TiDBにはその機構がなく、Self-Managedでは本編で触れた公式のロードマップを待つことになります。

### CPU推論は並列で伸ばせるか

本編で触れた「CPUが余って見えるのに並列で伸びない」理由を深掘りします。結論から言うと、ほとんど伸びません。制約は2つあります。

- **制約1(Pod内): CPUバウンド** — PyTorchは物理コア数と同じ8スレッドで推論し、1リクエストで物理コアの演算能力を使い切ります。CPU使用率が40〜50%に見えるのは、SMT込みの論理16コアを基準にしているためです。並列に実行すると、リクエストごとにスレッドチームが立って論理16コアまで埋まり、ノードのCPU使用率は90%近くまで上がります。しかし、追加で使われる論理コアはSMTの相方として物理コアの演算器を共有するため、スループットはほとんど増えません。cgroupの実測では、2リクエストを同時処理するPodが15コアを使いつつ、処理能力は1リクエスト時と同等でした。Podあたり約1/7 req/sが実力で、上限は「1/7 × Pod数」、2 Podで0.286 req/sです
- **制約2(Pod間): ランダム振り分け** — Serviceは空いているPodを選ばず、接続ごとにランダムに割り当てます。同時2では確率1/2で同じPodに寄り、片方がアイドルになるため、期待値は1.5倍止まりです。同時実行数を増やせば緩和できます(同時4で片方が空く確率は12.5%)

同時実行数を変えた実測です。encode計測の`embed-req.json`を使い回し、`xargs -P`で切り替えます。

:::details 計測スクリプト(同時実行スイープ)

```bash
bench() {
  local P=$1 N=$2
  local s=$(python3 -c 'import time; print(time.time())')
  seq $N | xargs -P $P -I{} curl -s -o /dev/null \
    -X POST "http://plamo-embedding.${TAILNET}/embed" \
    -H 'Content-Type: application/json' -d @embed-req.json
  local e=$(python3 -c 'import time; print(time.time())')
  python3 -c "print(f'P=$P N=$N wall={$e-$s:.1f}s throughput={$N/($e-$s):.3f} req/s')"
}
bench 1 4
bench 2 8
bench 4 12
```

:::

| 同時実行数 | リクエスト数 | 所要時間 | スループット | 並列なし比 |
| :-: | :-: | --: | --: | :-: |
| 1 (並列なし) | 4 | 28.0秒 | 0.143 req/s | 1.0倍 |
| 2 | 8 | 38.0秒 | 0.211 req/s | 1.48倍 |
| 4 | 12 | 46.1秒 | 0.260 req/s | 1.82倍 |

同時2の1.48倍は制約2の期待値(1.5倍)、同時4の1.82倍は制約1の上限(2倍)の約9割であり、実測は計算とおおむね整合します。ただし、この計測中にnode2側のPodがOOMKillで一時離脱していたことが後から判明しました(本編のインデックス化の章を参照)。そのため、特に同時2の1.48倍にはOOMKillの影響が含まれている可能性があり、参考値として扱う必要があります。

| 制約 | 解消策 | 効果 |
| --- | --- | --- |
| Pod内: CPUバウンド | Podを足す / GPUを使う | 天井そのものが上がる |
| Pod間: ランダム振り分け | 同時実行数を積む(同時4で十分) | 天井の約9割まで到達(天井は超えない) |

なお、スループットが最大になるのは同時4ですが、1件あたりの処理が最速なのは同時1です(7秒。同時4では1件約15秒)。一括投入では総スループットを優先するため、本編では同時4を採用しています。

![並列スイープ中のノード別CPU使用率。同時4でもピークはnode3が約63%、node2が約47%](https://res.cloudinary.com/dkerzyk09/image/upload/v1784334518/blog/2026-07-16-selfhosted-tidb-vector-plamo-embedding-1b-hybrid-search/da8alluv6zzb1e6s2vwl.png)
*同時実行スイープ中のノードCPU。同時1→2→4と上げてもピークは60%前後にとどまり、論理CPU全体ではなく、推論に使う8スレッドの演算能力がボトルネックになっていることが分かる*

### GPUならどれくらい速いか(DGX Sparkで試算)

CPU推論の実測(1チャンク7秒)があると、GPUに載せた場合の速度を桁レベルで見積もれます。1,024トークンの埋め込み生成に必要な計算量は、おおよそ「2 × パラメータ数 × トークン数 = 2 × 10⁹ × 1,024 ≒ 2 TFLOPs/チャンク」です。実測の7秒から逆算すると、Ryzen 7 7730Uの実効性能は約0.3 TFLOPSで、この見積もりと整合します。

同じ2 TFLOPsをDGX Spark(GB10、BF16の密行列演算性能でおおよそ100 TFLOPS級)で処理すると、理論値は20ms、実効値も30〜50ms程度と推定できます。GPUではバッチ処理も有効なため、スループットはさらに伸ばせます。

| | CPU(本構成、2ノード) | DGX Spark(推定) |
| --- | --- | --- |
| 1チャンク | 7秒 | 30〜50ms |
| 10万チャンク | 約4.5日 | 1〜2時間 |
| 100万チャンク | 約45日(非現実的) | 半日〜1日 |

本編で「非現実的」とした100万チャンクの一括投入が、GPU 1台で一晩の処理に変わる計算です。なお、GPUで利用する場合、`server.py`は`DEVICE = "cuda" if torch.cuda.is_available() else "cpu"`によりCUDAを選択します。

### 1記事1ベクトル vs チャンク方式

検索精度の章で使った言い換えクエリでの補足検証です。埋め込みの単位を「1記事1ベクトル」にするか「チャンクごと」にするかを比べます。PLaMoの入力上限は1,024トークンのため、1記事1ベクトル方式は実質「記事先頭の1,024トークンだけの埋め込み」になります。これは本編のテーブルの先頭チャンク(`chunk_index = 0`)そのものなので、`WHERE chunk_index = 0`で絞れば1記事1ベクトル方式の検索を再現できます。

クエリ「将棋で光速の寄せと呼ばれた棋士」で比べます。答えに当たる「光速の寄せ」への言及は、谷川浩司の記事(25チャンク、約2.5万トークン)の後半にしかありません。チャンク方式(本編の検索)では、top3を谷川浩司のチャンクが独占します。

```text
0.3038  谷川浩司 (chunk 21)
0.3228  谷川浩司 (chunk 10)
0.3328  谷川浩司 (chunk 17)   ← 「光速の寄せ」の記述を含むチャンク
```

同じクエリを先頭チャンクだけに絞って、1記事1ベクトル方式を再現します。

```bash
QVEC=$(curl -s -X POST http://plamo-embedding.${TAILNET}/embed \
  -H 'Content-Type: application/json' \
  -d '{"text":"将棋で光速の寄せと呼ばれた棋士","mode":"query"}' | jq -c '.vector')

mysql -t -h tidb.${TAILNET} -P 4000 -u root bench_wiki -e "
SELECT /*+ READ_FROM_STORAGE(TIFLASH[c]) */
       title, chunk_index,
       VEC_COSINE_DISTANCE(embedding, '${QVEC}') AS distance
  FROM wiki_embedding_chunks AS c
 WHERE chunk_index = 0
 ORDER BY VEC_COSINE_DISTANCE(embedding, '${QVEC}') + 0
 LIMIT 5;"
```

```text
+--------------+-------------+---------------------+
| title        | chunk_index | distance            |
+--------------+-------------+---------------------+
| 石井秀吉     |           0 | 0.36196060406573516 |
| 村上文祥     |           0 | 0.37181090608354583 |
| 本田邦久     |           0 |  0.3723709897252001 |
| 栃光正之     |           0 | 0.37830654592072643 |
| 佃正樹       |           0 |  0.3912052429456595 |
+--------------+-------------+---------------------+
```

top5はいずれも別人の記事で、谷川浩司(先頭チャンクのdistanceは0.4070)は**圏外**です。長い記事では先頭1,024トークンに後半の内容が反映されないため、1記事1ベクトル方式では「記事のどこかに書いてあること」を検索できません。チャンク方式では、これをdistance 0.3038のtop1で取得できます。

代償は行数です。チャンク方式では4,083記事が10,017行(約2.5倍)になり、レイテンシの章で見た検索コストとストレージをその分支払います。短い記事(1チャンクに収まるもの)では両方式の検索対象が同じになるため、チャンク方式の価値はコーパスの記事長の分布で決まります。
