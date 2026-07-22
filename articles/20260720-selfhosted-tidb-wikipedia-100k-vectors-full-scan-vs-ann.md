---
title: "TiDB Self-Managedで日本語版Wikipedia 4万記事（10万ベクトル）の全件走査とANN検索を比較する"
description: "セルフホストTiDBに日本語版Wikipedia 4万記事をチャンク化した10万ベクトルを投入し、TiKV/TiFlashの全件走査とHNSWのANN検索を実測比較。全件走査は線形劣化するという予測の答え合わせから、HNSW構築時間・インデックスサイズ・recall@10まで記録します"
publish: true
tags:
  - "tech/tidb"
  - "tech/tiflash"
  - "tech/kubernetes"
  - "tech/plamo-embedding-1b"
---

## はじめに

TiDBはベクトル型と距離関数に加えて、TiFlash上で動作するHNSWベクトルインデックスをサポートしています。同じTiDBクラスタ内で、TiKVまたはTiFlashによる全件走査と、HNSWインデックスを使ったANN検索を実行できます。

前回の記事では、同じ構成に日本語版Wikipediaの1万ベクトル（4,083記事）を投入し、環境構築から検索レイテンシ・再現率までを実測しました。本記事はその続編です。

https://shuntaka.dev/shuntaka/articles/20260716-selfhosted-tidb-vector-plamo-embedding-1b-hybrid-search

本記事では、日本語版Wikipediaの約4万記事をチャンク化した10万ベクトルをTiDBへ格納し、コサイン距離による上位10件の検索を比較します。埋め込みには日本語向けのPLaMo-Embedding-1Bを使用し、2,048次元のベクトルをKubernetesクラスタ内で生成します。

比較するのは、TiKVによる全件走査、TiFlashによる全件走査、TiFlashのHNSWインデックスを使ったANN検索の3パターンです。それぞれの実行計画と検索レイテンシを確認し、ANNについては全件走査の検索結果に対する再現率も計測します。10万ベクトルという規模で、ストレージエンジンとインデックスの違いが検索性能と検索結果にどのように現れるかを検証します。

実測の数字と結論だけを読みたい場合は、[計測結果](#計測結果)と[まとめ](#まとめ)へどうぞ。

## ベクトル検索について

### 全件走査によるkNN検索

コサイン距離を使って上位k件を厳密に求める全件走査では、クエリベクトルとDBに保存された各ベクトルとの距離を計算します。

今回のように検索対象が10万ベクトルある場合、SQLクエリ自体は1回ですが、その実行中に最大10万件分の距離計算が必要です。検索対象のベクトル数を`N`とすると、全件走査の計算量は`O(N)`であり、データ量や同時検索数が増えると、CPU負荷や検索レイテンシーのボトルネックになり得ます。


### ANN

ANN（Approximate Nearest Neighbor、近似最近傍探索）では、あらかじめ構築したインデックスを使って、クエリベクトルに近い可能性が高い候補を絞り込みます。すべてのベクトルとの距離を計算しないため検索を高速化できますが、全件走査で得られる本来の上位k件と結果が一致する保証はありません。

ANNを実現する方式は複数あり、HNSWはその一つです。ANN-Benchmarksでは、さまざまなANNのアルゴリズムや実装について、データセットごとに1秒あたりのクエリ数と再現率の関係を比較できます。インデックスサイズや構築時間も確認でき、検索速度だけで優劣を決められないことが分かります。

https://ann-benchmarks.com/index.html

TiDBのベクトルインデックスは、近いベクトル同士を多層のグラフで結ぶHNSW（Hierarchical Navigable Small World）をサポートしています。検索時はこのグラフをたどって有望な候補を探索します。インデックス作成後も、`ORDER BY VEC_COSINE_DISTANCE(...) LIMIT k` というSQL自体は全件走査と同じですが、条件が合えばオプティマイザがHNSWインデックスを利用します。

ANNにはインデックスの構築・保持コストがあり、TiDBではベクトルインデックスを利用するためにTiFlashレプリカが必要です。また、検索品質は正解となる全件走査の結果に対する再現率で評価します。そのため、今回の比較では検索時間だけでなく、ANNが全件走査の上位k件をどれだけ取得できたかも確認します。

## 前提

### アーキテクチャ

検証環境の全体構成を示します。ミニPC 3台で構成したKubernetesクラスタ上に、TiDB Operatorを使ってTiDB v8.5.7（PD / TiKV / TiDB / TiFlash）を構築しています。GPUは搭載していません。

![検証環境の全体構成図](https://res.cloudinary.com/dkerzyk09/image/upload/v1784334412/blog/2026-07-16-selfhosted-tidb-vector-plamo-embedding-1b-hybrid-search/ah18icf4fmbaexxlrjog.png)
*前記事と同じ構成。本記事で扱うのは右側のKubernetes部分で、左側のWebシステムは検証の背景*

:::message
本構成で利用しているTailscaleのPersonalプランは、個人による非商用利用が対象です。構成図左側のWebシステムは、筆者が個人で非商用運営しているサービスであり、この前提でPersonalプランを利用しています。業務・商用目的で同様の構成を採用する場合はPersonalプランの対象外となるため、用途に合った有料プランを検討してください。最新の適用条件は[Tailscaleの料金ページ](https://tailscale.com/pricing)を確認してください。
:::

| ノード | CPU | メモリ | 本記事での主な役割 |
| --- | --- | --- | --- |
| node1 | Ryzen 7 7730U（8コア / 16スレッド） | 32GB | TiFlash、データ投入スクリプトの実行 |
| node2 | Ryzen 7 7730U（8コア / 16スレッド） | 32GB | PLaMo推論サービス |
| node3 | Ryzen 7 7730U（8コア / 16スレッド） | 32GB | PLaMo推論サービス |

Wikipediaデータの投入スクリプトは、常時稼働しているnode1上で実行します。TiDBとPLaMo推論サービスへの接続は前記事と同様に、TailscaleのMagicDNS名（`tidb.<tailnet>` / `plamo-embedding.<tailnet>`）を使います。node1も手元PCもtailnetに参加しているため、投入（node1上）も確認・計測（手元PC）も同じ接続先で実行できます。

埋め込み生成のCPU推論とHNSWインデックスの構築は、どちらもCPU負荷の高い処理です。両者が同じノード上で競合しないように、TiFlashはnode1、PLaMo推論サービスはnode2とnode3へ配置します。

参考として、TiDB公式の[ハードウェア要件](https://docs.pingcap.com/tidb/stable/hardware-and-software-requirements/)と本環境の比較を示します。公式要件には開発・テスト環境向けの最低ラインと本番環境向けの2段階があり、本環境はその最低ラインも下回ります。

| コンポーネント | 最低要件（開発・テスト） | 本番環境 | 本環境 |
| --- | --- | --- | --- |
| PD | 4コア / 8GB ×1 | 8コア / 16GB ×3 | 3ノードに同居 ×3 |
| TiDB | 8コア / 16GB ×1 | 16コア / 48GB ×2 | 3ノードに同居 ×3 |
| TiKV | 8コア / 32GB ×3 | 16コア / 64GB ×3 | 3ノードに同居 ×3（メモリ上限12GiB） |
| TiFlash | 32コア / 64GB ×1 | 48コア / 128GB ×2 | node1のみ ×1（メモリ上限8GiB） |

公式要件がコンポーネントごとの専用サーバーを前提とするのに対して、本環境は8コア16スレッド / 32GBのノード3台へ全コンポーネントを同居させ、node2とnode3ではPLaMo推論サービスも動かしています。特にTiFlashは、最低要件の32コア / 64GBに対してノード全体でも8コア16スレッド / 32GB、Podのメモリ上限は8GiBと大きく乖離しています。

:::message
本環境は、TiDBが公式に定める[ハードウェア要件](https://docs.pingcap.com/tidb/stable/hardware-and-software-requirements/)のうち、開発・テスト環境向けの最低要件も下回る構成です(特にTiFlashの最低要件は32コア/64GB)。本記事のレイテンシやスループットの絶対値は、この環境固有の参考値として見てください。主眼は、同一環境内での各検索経路の相対比較と、セルフホスト構成の実用性確認にあります。
:::

### 比較対象の検索方法

今回は、同じベクトルデータに対するコサイン距離検索を、データの読み取り元や距離計算の実行場所、インデックスの有無が異なる次の3パターンで比較します。

| パターン | 検索方法 | 読み取り・検索処理 | 距離計算 |
| ---: | --- | --- | --- |
| 1 | 全件走査によるkNN検索 | TiKV（行指向） | 検索対象の全ベクトルと比較（厳密検索） |
| 2 | (同上) | TiFlash（列指向） | (同上) |
| 3 | HNSWによるANN検索 | TiFlash + HNSWインデックス | グラフ探索で候補を絞って比較（近似検索） |

パターン1と2では、距離計算とTopNをそれぞれTiKVとTiFlashへプッシュダウンし、同じ全件走査を行指向と列指向のストレージエンジンで実行した場合の検索時間を比較します。パターン3では、HNSWインデックスによって距離計算の対象を絞った場合の検索時間と再現率を確認します。

各検索文は事前にベクトル化し、TiDBへSQLを発行してから検索結果が返るまでの時間を計測します。

## 検証用ベクトルデータの生成と投入

### 検証データの準備

検証データには、日本語WikipediaのCirrusSearchダンプを使用します。`[[リンク]]`や`{{テンプレート}}`のようなWikipedia独自のマークアップが取り除かれた、プレーンテキストの本文が`text`フィールドに入っているため、そのままチャンク分割に使えます。

1記事はインデックス行と記事本体の2行のJSON Linesで表現されます。先頭レコードの抜粋を示します。

```json:jawiki_content-20260712-00000.json（展開後・抜粋）
{"index": {"_id": 2332351}}
{
  "page_id": 2332351,
  "title": "山陰合同銀行根雨支店",
  "text": "山陰合同銀行 > 山陰合同銀行根雨支店 山陰合同銀行根雨支店（さんいんごうどうぎんこうねうしてん）は、山陰合同銀行の鳥取県日野郡日野町にある支店。1929年（昭和4年）に雲陽実業銀行根雨支店として建築された建物。...",
  "heading": ["概要", "歴史", "脚注", "外部リンク"],
  "category": ["日本の銀行建築", "西洋館", "鳥取県日野町の建築物", ...]
}
```

投入に使うのは`page_id`・`title`・`text`の3つで、そのほかに`opening_text`や`outgoing_link`など検索インデックス用のフィールドが20以上あります。

node1で2026年7月12日版の`jawiki_content`先頭シャードをダウンロードします。

```bash
ssh node1
mkdir -p ~/work/20260717
cd ~/work/20260717

curl -LO "https://dumps.wikimedia.org/other/cirrus_search_index/20260712/index_name%3Djawiki_content/jawiki_content-20260712-00000.json.bz2"
```

データセットの規模と、投入する10万ベクトルのサイズ感を次にまとめます。

| 項目 | 値 |
| --- | --- |
| 先頭シャード | bz2圧縮で約640MB（展開すると約4.8GB）・107,832記事 |
| `jawiki_content`全体 | 14シャード・圧縮で合計約9GB（単純計算で約150万記事） |
| 今回の投入対象 | 10万ベクトル・約4万記事（先頭シャードの4割弱） |
| 本文テキストの合計 | 約280MB（1ベクトルあたり平均約2,800バイト・約830トークン） |
| ベクトル本体の合計 | 約820MB（10万 × 2,048次元 × 4バイト） |

埋め込みは元の本文テキストの約3倍のサイズになり、ストレージの主役は本文ではなくベクトルとHNSWインデックスになります。

:::message alert
日本語版Wikipediaの本文はCC BY-SA 4.0とGFDLのデュアルライセンスで公開されています。ダンプの取得・加工やベンチマーク結果の公表はライセンス上問題ありませんが、本文の抜粋を転載する場合は出典の表示が必要です。本記事で検索結果として本文を掲載する箇所は、該当するWikipedia記事を出典として明記します。
:::

### PLaMo APIと投入スクリプト

投入で使うコードは次の2つです。

| コンポーネント | ファイル | 実行場所 |
| --- | --- | --- |
| PLaMo API | `server.py`・`chunking.py` | node2 / node3のPod（常駐） |
| 投入スクリプト | `ingest_wiki.py` | node1 |

PLaMo APIは、PLaMo-Embedding-1Bをロードして常駐するFastAPIサーバーです。トークナイザによるチャンク分割を`POST /chunks`、埋め込み生成を`POST /embed`として提供します。`chunking.py`は見出し構造を保ってMarkdownを分割し、タイトル・見出しをメタデータとして各チャンクの先頭へ付与します。

:::message
同じPLaMO-Embedding-1Bは[Cloudflare Workers AI](https://developers.cloudflare.com/workers-ai/models/plamo-embedding-1b/)でも利用できます。本構成のCPU推論では10万ベクトルの生成に約96時間かかるため、データを外部へ送信できない場合や、セルフホスト自体を検証したい場合を除けば、Workers AIを使う方が現実的です。本記事ではセルフホスト環境の性能を確認するため、あえてCPUで生成します。
:::

:::details PLaMo APIの全文（server.py / chunking.py）
```python:server.py
# cspell:ignore healthz
"""PLaMo Embedding 1B の HTTP wrapper.

`POST /embed` に {"text": "...", "mode": "query"|"document"} を投げると、
2048 次元の float 配列を返す。`POST /chunks` は同じtokenizerを使い、Markdownを
学習時のcontext長に合わせて分割する。tidb-embedder (backfill) と blog-api (検索) が
同じServiceを利用する。
"""
import logging
import os

import torch
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field, model_validator
from transformers import AutoModel, AutoTokenizer

from chunking import CHUNKING_VERSION, chunk_document

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")
log = logging.getLogger(__name__)

MODEL_ID = os.environ.get("MODEL_ID", "pfnet/plamo-embedding-1b")
DEVICE = "cuda" if torch.cuda.is_available() else "cpu"

log.info("loading model=%s device=%s", MODEL_ID, DEVICE)
tokenizer = AutoTokenizer.from_pretrained(MODEL_ID, trust_remote_code=True)
model = AutoModel.from_pretrained(MODEL_ID, trust_remote_code=True).to(DEVICE).eval()
log.info("model loaded")

app = FastAPI()


class EmbedRequest(BaseModel):
    text: str
    mode: str  # "query" | "document"


class EmbedResponse(BaseModel):
    vector: list[float]
    dim: int


class ChunksRequest(BaseModel):
    title: str
    description: str = ""
    content: str
    max_tokens: int = Field(default=1024, ge=64, le=4096)
    overlap_tokens: int = Field(default=128, ge=0)

    @model_validator(mode="after")
    def validate_overlap(self) -> "ChunksRequest":
        if self.overlap_tokens >= self.max_tokens:
            raise ValueError("overlap_tokens must be less than max_tokens")
        return self


class ChunkResponseItem(BaseModel):
    index: int
    heading: str | None
    content: str
    embedding_text: str
    token_count: int


class ChunksResponse(BaseModel):
    version: str
    max_tokens: int
    overlap_tokens: int
    chunks: list[ChunkResponseItem]


@app.post("/embed", response_model=EmbedResponse)
def embed(req: EmbedRequest) -> EmbedResponse:
    if req.mode not in ("query", "document"):
        raise HTTPException(status_code=400, detail="mode must be 'query' or 'document'")
    with torch.inference_mode():
        if req.mode == "query":
            vec = model.encode_query(req.text, tokenizer)
        else:
            vec = model.encode_document(req.text, tokenizer)
    # encode_* は (1, hidden_size) を返す。squeeze して 1 次元にする
    vec_list = vec.squeeze(0).cpu().float().tolist()
    return EmbedResponse(vector=vec_list, dim=len(vec_list))


@app.post("/chunks", response_model=ChunksResponse)
def chunks(req: ChunksRequest) -> ChunksResponse:
    items = chunk_document(
        tokenizer,
        title=req.title,
        description=req.description,
        content=req.content,
        max_tokens=req.max_tokens,
        overlap_tokens=req.overlap_tokens,
    )
    return ChunksResponse(
        version=CHUNKING_VERSION,
        max_tokens=req.max_tokens,
        overlap_tokens=req.overlap_tokens,
        chunks=[ChunkResponseItem(**item.__dict__) for item in items],
    )


@app.get("/healthz")
def healthz() -> dict[str, str]:
    return {"status": "ok"}
```

```python:chunking.py
"""PLaMO tokenizer に合わせた Markdown document chunking."""

from dataclasses import dataclass
import re
from typing import Protocol


CHUNKING_VERSION = "plamo-markdown-1024-v1"
METADATA_TOKEN_BUDGET = 256


class Tokenizer(Protocol):
    def encode(self, text: str, add_special_tokens: bool = False) -> list[int]: ...

    def decode(self, token_ids: list[int], skip_special_tokens: bool = True) -> str: ...


@dataclass(frozen=True)
class MarkdownSection:
    heading: str | None
    content: str


@dataclass(frozen=True)
class DocumentChunk:
    index: int
    heading: str | None
    content: str
    embedding_text: str
    token_count: int


_HEADING_PATTERN = re.compile(r"^(#{1,6})[ \t]+(.+?)[ \t]*$")
_CLOSING_HASHES_PATTERN = re.compile(r"[ \t]+#+[ \t]*$")
_FENCE_PATTERN = re.compile(r"^[ \t]*(`{3,}|~{3,})")


def split_markdown_sections(content: str) -> list[MarkdownSection]:
    """見出し階層を保ちながら Markdown を section に分割する。"""

    sections: list[MarkdownSection] = []
    headings: list[str] = []
    lines: list[str] = []
    current_heading: str | None = None
    fence_marker: str | None = None

    def flush() -> None:
        nonlocal lines
        body = "\n".join(lines).strip()
        if body:
            sections.append(MarkdownSection(heading=current_heading, content=body))
        lines = []

    for line in content.replace("\r\n", "\n").replace("\r", "\n").split("\n"):
        fence = _FENCE_PATTERN.match(line)
        if fence:
            marker = fence.group(1)[0]
            if fence_marker is None:
                fence_marker = marker
            elif fence_marker == marker:
                fence_marker = None
            lines.append(line)
            continue

        heading = _HEADING_PATTERN.match(line) if fence_marker is None else None
        if heading:
            flush()
            level = len(heading.group(1))
            title = _CLOSING_HASHES_PATTERN.sub("", heading.group(2)).strip()
            headings = headings[: level - 1]
            while len(headings) < level - 1:
                headings.append("")
            headings.append(title)
            current_heading = " > ".join(value for value in headings if value)
            continue

        lines.append(line)

    flush()
    return sections


def _token_ids(tokenizer: Tokenizer, text: str) -> list[int]:
    return list(tokenizer.encode(text, add_special_tokens=False))


def _token_count(tokenizer: Tokenizer, text: str) -> int:
    return len(_token_ids(tokenizer, text))


def _metadata_prefix(
    tokenizer: Tokenizer,
    title: str,
    description: str,
    heading: str | None,
    max_tokens: int,
) -> str:
    fields = [f"タイトル: {title.strip()}"]
    if heading:
        fields.append(f"見出し: {heading.strip()}")
    if description.strip():
        fields.append(f"概要: {description.strip()}")

    header = "\n".join(fields)
    suffix = "\n本文:\n"
    # metadata が長くても本文用の token budget を確保する。タイトル、見出し、概要の
    # 順なので、切り詰めが必要な場合も検索に重要な情報を優先できる。
    budget = min(METADATA_TOKEN_BUDGET, max_tokens // 2)
    suffix_tokens = _token_ids(tokenizer, suffix)
    header_tokens = _token_ids(tokenizer, header)
    header_budget = max(1, budget - len(suffix_tokens))
    if len(header_tokens) > header_budget:
        header = tokenizer.decode(
            header_tokens[:header_budget], skip_special_tokens=True
        ).strip()
    return f"{header}{suffix}"


def chunk_document(
    tokenizer: Tokenizer,
    *,
    title: str,
    description: str,
    content: str,
    max_tokens: int = 1024,
    overlap_tokens: int = 128,
) -> list[DocumentChunk]:
    """Markdown section を PLaMO の token 数で window 分割する。"""

    if not 64 <= max_tokens <= 4096:
        raise ValueError("max_tokens must be between 64 and 4096")
    if not 0 <= overlap_tokens < max_tokens:
        raise ValueError("overlap_tokens must be between 0 and max_tokens - 1")

    sections = split_markdown_sections(content)
    if not sections:
        sections = [MarkdownSection(heading=None, content="")]

    chunks: list[DocumentChunk] = []
    for section in sections:
        prefix = _metadata_prefix(
            tokenizer, title, description, section.heading, max_tokens
        )
        prefix_count = _token_count(tokenizer, prefix)
        body_budget = max_tokens - prefix_count
        if body_budget <= 0:
            raise ValueError("metadata leaves no token budget for content")

        body_tokens = _token_ids(tokenizer, section.content)
        if not body_tokens:
            embedding_text = prefix.rstrip()
            chunks.append(
                DocumentChunk(
                    index=len(chunks),
                    heading=section.heading,
                    content="",
                    embedding_text=embedding_text,
                    token_count=_token_count(tokenizer, embedding_text),
                )
            )
            continue

        start = 0
        while start < len(body_tokens):
            end = min(start + body_budget, len(body_tokens))
            window = body_tokens[start:end]
            body = tokenizer.decode(window, skip_special_tokens=True).strip()
            embedding_text = f"{prefix}{body}"
            token_count = _token_count(tokenizer, embedding_text)

            # SentencePiece は prefix と本文を結合した境界で再tokenize結果がわずかに
            # 変わり得る。最終入力を数え直し、1024 tokenを確実に超えないよう縮める。
            while token_count > max_tokens and len(window) > 1:
                overflow = token_count - max_tokens
                window = window[: max(1, len(window) - overflow - 1)]
                end = start + len(window)
                body = tokenizer.decode(window, skip_special_tokens=True).strip()
                embedding_text = f"{prefix}{body}"
                token_count = _token_count(tokenizer, embedding_text)

            chunks.append(
                DocumentChunk(
                    index=len(chunks),
                    heading=section.heading,
                    content=body,
                    embedding_text=embedding_text,
                    token_count=token_count,
                )
            )
            if end >= len(body_tokens):
                break
            effective_overlap = min(overlap_tokens, len(window) - 1)
            start = end - effective_overlap

    return chunks
```
:::

PLaMo APIのコンテナイメージと、node2 / node3へ1 Podずつ配置するKubernetesマニフェストは次のとおりです。モデルはビルド時にイメージへ焼き込み、メモリアロケータはjemallocへ差し替えています（経緯は後述）。

:::details Dockerfileとマニフェスト（Dockerfile / deployment.yaml）
```dockerfile:Dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates \
      libjemalloc2 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# CPU版PyTorch(クラスタにGPUなし)
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

# モデル(約2.3GB)はビルド時に取得してイメージへ焼き込み、実行時はHF cacheからロードする
ARG MODEL_ID=pfnet/plamo-embedding-1b
ENV MODEL_ID=${MODEL_ID}
RUN python -c "from huggingface_hub import snapshot_download; \
  snapshot_download(repo_id='${MODEL_ID}')"

# メモリアロケータをjemallocへ差し替える(経緯は後述)
ENV LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2
ENV MALLOC_CONF=background_thread:true,dirty_decay_ms:10000

EXPOSE 8080
CMD ["uvicorn", "server:app", "--host", "0.0.0.0", "--port", "8080"]
```

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
          image: ghcr.io/shuntaka9576/plamo-embedding:latest
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
              # OOMKillされた(2026-07-17)ため8Giへ引き上げ
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

投入スクリプトの`ingest_wiki.py`は、ダンプを読み込んでPLaMo APIを呼び出し、TiDBへINSERTします。全文は次のとおりです。

:::details ingest_wiki.pyの全文
```python:ingest_wiki.py
# /// script
# requires-python = ">=3.11"
# dependencies = ["pymysql"]
# ///
"""日本語Wikipedia(cirrus_search_indexダンプ)をチャンク化してTiDBへ投入する。

シャード(.json.bz2)をストリームで読み、PLaMoサービスの /chunks でチャンク化、
/embed (mode=document) で2048次元ベクトルを生成して bench_wiki.wiki_embedding_chunks
へINSERTする。--target-chunks に達したら終了する。

再開可能: 投入済みのpage_idはスキップし、既存チャンク数をターゲットに算入する。
INSERTはページ単位のトランザクションなので、中断してもページ単位で整合する。

Usage:
  export TAILNET=$(tailscale status --json | jq -r '.MagicDNSSuffix')
  uv run ingest_wiki.py jawiki_content-20260712-00000.json.bz2 \
    --target-chunks 10000 --concurrency 4
  uv run ingest_wiki.py <shard> --dry-run   # チャンク数の見積もりだけ(embed/INSERTなし)
"""

import argparse
import bz2
import gzip
import json
import os
import sys
import time
import urllib.request
from concurrent.futures import ThreadPoolExecutor

import pymysql


def log(msg: str) -> None:
    print(f"[{time.strftime('%H:%M:%S')}] {msg}", flush=True)


def http_post_json(url: str, payload: dict, timeout: int, retries: int = 20) -> dict:
    body = json.dumps(payload).encode()
    last_err: Exception | None = None
    for attempt in range(1, retries + 1):
        try:
            req = urllib.request.Request(
                url, data=body, headers={"Content-Type": "application/json"}
            )
            with urllib.request.urlopen(req, timeout=timeout) as resp:
                return json.load(resp)
        except Exception as e:  # noqa: BLE001
            last_err = e
            log(f"HTTP retry {attempt}/{retries} {url}: {e}")
            # Pod再起動時のモデルロード(最大10分)を跨げるよう、計16分程度まで粘る
            time.sleep(min(5 * attempt, 120))
    raise RuntimeError(f"HTTP failed after {retries} retries: {url}: {last_err}")


def open_dump(path: str):
    if path.endswith(".bz2"):
        return bz2.open(path, "rt", encoding="utf-8")
    if path.endswith(".gz"):
        return gzip.open(path, "rt", encoding="utf-8")
    return open(path, encoding="utf-8")


def iter_pages(paths: list[str], min_chars: int):
    """ダンプのNDJSONからドキュメント行だけを取り出す。

    形式: {"index": {"_id": N}} 行と {"page_id": N, "title": ..., "text": ...} 行の2行1組。
    ドキュメント行はpage_idを含むため、index行は読み飛ばすだけでよい。
    """
    for path in paths:
        log(f"reading {path}")
        with open_dump(path) as f:
            for line in f:
                try:
                    doc = json.loads(line)
                except json.JSONDecodeError:
                    continue
                if "index" in doc and "page_id" not in doc:
                    continue
                page_id = doc.get("page_id")
                title = doc.get("title")
                text = doc.get("text")
                if not page_id or not title or not text:
                    continue
                if doc.get("namespace", 0) != 0:
                    continue
                if len(text) < min_chars:
                    continue
                yield {"page_id": page_id, "title": title, "text": text}


def main() -> int:
    tailnet = os.environ.get("TAILNET", "")
    ap = argparse.ArgumentParser(description=__doc__)
    ap.add_argument("files", nargs="+", help="ダンプシャード (.json.bz2 / .json.gz)")
    ap.add_argument(
        "--embed-endpoint",
        default=os.environ.get(
            "PLAMO_EMBED_ENDPOINT",
            f"http://plamo-embedding.{tailnet}" if tailnet else "",
        ),
    )
    ap.add_argument("--tidb-host", default=f"tidb.{tailnet}" if tailnet else "")
    ap.add_argument("--tidb-port", type=int, default=4000)
    ap.add_argument("--tidb-user", default="root")
    ap.add_argument("--database", default="bench_wiki")
    ap.add_argument("--table", default="wiki_embedding_chunks")
    ap.add_argument("--target-chunks", type=int, default=10_000)
    ap.add_argument("--concurrency", type=int, default=4)
    ap.add_argument("--max-tokens", type=int, default=1024)
    ap.add_argument("--overlap-tokens", type=int, default=128)
    ap.add_argument("--min-chars", type=int, default=200)
    ap.add_argument("--dry-run", action="store_true", help="embed/INSERTせず見積もりのみ")
    args = ap.parse_args()

    if not args.embed_endpoint:
        ap.error("--embed-endpoint (または TAILNET / PLAMO_EMBED_ENDPOINT) が必要")

    conn = None
    done_pages: set[int] = set()
    inserted = 0
    if not args.dry_run:
        if not args.tidb_host:
            ap.error("--tidb-host (または TAILNET) が必要")
        conn = pymysql.connect(
            host=args.tidb_host,
            port=args.tidb_port,
            user=args.tidb_user,
            database=args.database,
            charset="utf8mb4",
            autocommit=False,
        )
        with conn.cursor() as cur:
            cur.execute(f"SELECT DISTINCT page_id FROM {args.table}")
            done_pages = {row[0] for row in cur.fetchall()}
            cur.execute(f"SELECT COUNT(*) FROM {args.table}")
            inserted = cur.fetchone()[0]
        log(f"resume: {len(done_pages)} pages / {inserted} chunks already in DB")

    executor = ThreadPoolExecutor(max_workers=args.concurrency)

    def embed(text: str) -> list[float]:
        resp = http_post_json(
            f"{args.embed_endpoint}/embed",
            {"text": text, "mode": "document"},
            timeout=120,
        )
        return resp["vector"]

    def flush(pages: list[dict]) -> int:
        """保留ページのチャンクを並列embedし、ページ単位のトランザクションでINSERTする。"""
        jobs = [c for p in pages for c in p["chunks"]]
        vectors = list(executor.map(lambda c: embed(c["embedding_text"]), jobs))
        for c, v in zip(jobs, vectors):
            c["vector"] = v
        # 長時間実行中にTCPが切れていた場合はここで張り直す
        conn.ping(reconnect=True)
        n = 0
        for p in pages:
            try:
                with conn.cursor() as cur:
                    cur.executemany(
                        f"INSERT INTO {args.table}"
                        " (page_id, title, chunk_index, content, token_count, embedding)"
                        " VALUES (%s, %s, %s, %s, %s, %s)",
                        [
                            (
                                p["page_id"],
                                p["title"][:512],
                                c["index"],
                                c["content"],
                                c["token_count"],
                                json.dumps(c["vector"], separators=(",", ":")),
                            )
                            for c in p["chunks"]
                        ],
                    )
                conn.commit()
            except pymysql.err.IntegrityError:
                # コミット成功後に応答だけ失われた再実行ケース。UNIQUE制約で検出し投入済みとして扱う
                conn.rollback()
            else:
                n += len(p["chunks"])
        return n

    t0 = time.time()
    session_start = inserted  # 今回セッション開始時点の投入済み件数(rate計算用)
    planned = inserted  # 投入済み + 保留分
    pending: list[dict] = []
    pending_chunks = 0
    total_pages = 0

    try:
        for page in iter_pages(args.files, args.min_chars):
            if planned >= args.target_chunks:
                break
            if page["page_id"] in done_pages:
                continue

            chunk_resp = http_post_json(
                f"{args.embed_endpoint}/chunks",
                {
                    "title": page["title"],
                    "description": "",
                    "content": page["text"],
                    "max_tokens": args.max_tokens,
                    "overlap_tokens": args.overlap_tokens,
                },
                timeout=300,
            )
            chunks = chunk_resp["chunks"]
            if not chunks:
                continue

            total_pages += 1
            planned += len(chunks)

            if args.dry_run:
                inserted = planned
                if total_pages % 200 == 0:
                    log(f"dry-run: {total_pages} pages -> {planned} chunks")
                continue

            page["chunks"] = chunks
            pending.append(page)
            pending_chunks += len(chunks)

            if pending_chunks >= args.concurrency * 2:
                inserted += flush(pending)
                pending, pending_chunks = [], 0
                rate = (inserted - session_start) / max(time.time() - t0, 1)
                remain = max(args.target_chunks - inserted, 0)
                eta_h = remain / rate / 3600 if rate > 0 else float("inf")
                log(
                    f"{inserted}/{args.target_chunks} chunks"
                    f" ({total_pages} pages, {rate:.2f} chunks/s, ETA {eta_h:.1f}h)"
                )

        if pending and not args.dry_run:
            inserted += flush(pending)
    except KeyboardInterrupt:
        log("interrupted: 投入済みページまでは保存されています(再実行で再開)")
        return 130
    finally:
        executor.shutdown(wait=False, cancel_futures=True)
        if conn:
            conn.close()

    if args.dry_run:
        log(f"dry-run done: {total_pages} pages -> {planned} chunks")
    else:
        log(f"done: {inserted} chunks ({total_pages} pages this run)")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```
:::

処理の流れと、ソース上の対応箇所は次のとおりです。

1. `iter_pages`: ダンプをストリームで読み、インデックス行を読み飛ばして`page_id`・`title`・`text`を取り出す。200文字未満の記事は除外する
2. `main`のループ: PLaMoサービスの`POST /chunks`で、本文をPLaMo-Embedding-1Bのトークナイザで最大1,024トークン・128トークン重複のチャンクに分割する
3. `embed`: `POST /embed`へ`mode=document`で送信し、チャンクごとに2,048次元のベクトルを生成する（同時4並列）
4. `flush`: ページ単位のトランザクションで、1チャンクを1行として`wiki_embedding_chunks`テーブルへINSERTする

今回は、同じ条件で10万ベクトルまで投入します。投入スクリプトは登録済みの`page_id`をスキップするため、中断した場合も投入済みページの次から再開できます。

### 10万ベクトルを投入する

node1もtailnetに参加しているため、接続先（TiDBとPLaMo推論サービス）は前記事と同様にTailscaleのMagicDNS名で組み立てます。投入スクリプトは`TAILNET`環境変数から接続先を解決します。

```bash
export TAILNET=$(tailscale status --json | jq -r '.MagicDNSSuffix')
```

PLaMO推論サービスのヘルスチェックに加えて、実際に埋め込みを生成できることを確認します。レスポンスの`dim`が`2048`であれば、PLaMO-Embedding-1Bによる推論まで正常に動作しています。

```bash
curl -fsS "http://plamo-embedding.${TAILNET}/healthz" | jq

curl -fsS -X POST "http://plamo-embedding.${TAILNET}/embed" \
  -H 'Content-Type: application/json' \
  -d '{"text":"日本の城の石垣の構造","mode":"document"}' \
  | jq '{dim, vector_head: .vector[:5]}'
```

動作を確認できたら、投入を開始します。

```bash
nohup uv run ingest_wiki.py jawiki_content-20260712-00000.json.bz2 \
  --target-chunks 100000 \
  > ingest-100k.log 2>&1 &
```

接続先のほか、DB名（`bench_wiki`）・テーブル名（`wiki_embedding_chunks`）・同時実行数（4）はスクリプトのデフォルト値をそのまま使います。

`--target-chunks`には投入済みの行も含まれます。ページ単位でINSERTするため、最終的なベクトル数は10万を少し超える可能性があります。これまでの実測値である0.29ベクトル/秒を維持した場合、10万ベクトルの投入には約96時間かかる見込みです。

### その他の投入時の注意事項

PLaMO-Embedding-1Bはnode2とnode3にセルフホストしています。glibc mallocを使った1万ベクトルの投入では、解放済みヒープがOSへ返されず、8GiB制限のPodで最大7.81GiBまで使用しました。

そこで、アプリケーションコードは変更せず、Dockerfileでjemallocへ差し替えました。

```diff dockerfile
 FROM python:3.12-slim
 RUN apt-get update && apt-get install -y --no-install-recommends \
       ca-certificates \
+      libjemalloc2 \
     && rm -rf /var/lib/apt/lists/*
+ENV LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2
+ENV MALLOC_CONF=background_thread:true,dirty_decay_ms:10000
```

`LD_PRELOAD`でPyTorchなどが使うメモリアロケータを差し替え、未使用ページをバックグラウンドでpurgeします。`dirty_decay_ms:10000`は、dirty pageを約10秒の減衰期間でpurgeまたは再利用する設定です。詳細は[jemallocの公式ドキュメント](https://jemalloc.net/jemalloc.3.html)を参照してください。

| 指標 | glibc malloc（1万ベクトル投入） | jemalloc（10万ベクトル投入） |
| --- | ---: | ---: |
| モデルロード後のベースライン | 5.45〜5.72GiB | 約4.3GiB |
| 投入中の`memory.peak` | 7.31 / 7.81GiB | 5.16 / 5.45GiB |
| 8GiBのメモリ上限までの余裕 | 200MiB未満 | 約2.5GiB |
| 投入完了後のアイドルRSS | ピーク近くに滞留 | 4.27 / 4.28GiB（ベースラインへ復帰） |

![PLaMo推論Podのメモリ使用量をglibc mallocとjemallocで比較](https://res.cloudinary.com/dkerzyk09/image/upload/v1784677449/blog/2026-07-20-selfhosted-tidb-wikipedia-100k-vectors-full-scan-vs-ann/uydv3fit3kg82ziobnbb.png)
*PLaMO推論Podのメモリ使用量の推移。緑・黄がglibc malloc、赤・紫がjemalloc*

![glibc mallocとjemallocのRSS推移の違い](https://res.cloudinary.com/dkerzyk09/image/upload/v1784677452/blog/2026-07-20-selfhosted-tidb-wikipedia-100k-vectors-full-scan-vs-ann/pe7ncpf2rwus9xfncmkf.png)
*実測値をもとに、glibc mallocとjemallocのメモリ推移を模式化した図*

Pod世代をまたぐ比較ですが、同じ2Pod・同時実行数4・メモリ上限8GiBの構成で、投入中のピークを約2.4GiB低減し、投入完了後のRSSはベースラインへ戻りました。

投入は約89時間（3日と17時間）で完了しました。この実行では89,983ベクトルを実効0.281ベクトル/s（同時4並列）で投入し、期間中のPodの再起動（OOMKill）とHTTPリトライはどちらも0回でした。

### 投入結果を確認する

投入が完了したら、ベクトル数と記事数を確認します。以降の確認・計測コマンドは、手元PCからTailscaleのMagicDNS名（`tidb.<tailnet>`）でTiDBへ接続して実行します。

```bash
export TAILNET=$(tailscale status --json | jq -r '.MagicDNSSuffix')

mysql -h "tidb.${TAILNET}" -P 4000 -u root bench_wiki -e "
SELECT COUNT(*) AS vectors,
       COUNT(DISTINCT page_id) AS articles,
       SUM(token_count) AS tokens,
       ROUND(AVG(token_count), 1) AS avg_tokens_per_vector
  FROM wiki_embedding_chunks;"
```

```bash:実行結果
+---------+----------+----------+-----------------------+
| vectors | articles | tokens   | avg_tokens_per_vector |
+---------+----------+----------+-----------------------+
|  100000 |    39480 | 82564327 |                 825.6 |
+---------+----------+----------+-----------------------+
```

TiFlashレプリカの状態も確認します。

```bash
mysql -h "tidb.${TAILNET}" -P 4000 -u root -e "
SELECT TABLE_SCHEMA, TABLE_NAME, REPLICA_COUNT, AVAILABLE, PROGRESS
  FROM INFORMATION_SCHEMA.TIFLASH_REPLICA
 WHERE TABLE_SCHEMA = 'bench_wiki'
   AND TABLE_NAME = 'wiki_embedding_chunks';"
```

```bash:実行結果
+--------------+-----------------------+---------------+-----------+----------+
| TABLE_SCHEMA | TABLE_NAME            | REPLICA_COUNT | AVAILABLE | PROGRESS |
+--------------+-----------------------+---------------+-----------+----------+
| bench_wiki   | wiki_embedding_chunks |             1 |         1 |        1 |
+--------------+-----------------------+---------------+-----------+----------+
```

`AVAILABLE = 1`かつ`PROGRESS = 1`になり、TiKVとTiFlashから取得した行数が一致してからインデックスを作成します。

```bash
mysql -h "tidb.${TAILNET}" -P 4000 -u root bench_wiki -e "
SELECT /*+ READ_FROM_STORAGE(TIKV[c]) */ COUNT(*) AS tikv_rows
  FROM wiki_embedding_chunks AS c;
SELECT /*+ READ_FROM_STORAGE(TIFLASH[c]) */ COUNT(*) AS tiflash_rows
  FROM wiki_embedding_chunks AS c;"
```

```bash:実行結果
+-----------+
| tikv_rows |
+-----------+
|    100000 |
+-----------+
+--------------+
| tiflash_rows |
+--------------+
|       100000 |
+--------------+
````

## HNSWインデックスの作成

全ベクトルがTiFlashへ反映された後、データをStable層へまとめてからHNSWインデックスを作成します。COMPACTとインデックス作成は別々に実行し、それぞれの所要時間を記録します。

```bash
time mysql -h "tidb.${TAILNET}" -P 4000 -u root bench_wiki -e "
ALTER TABLE wiki_embedding_chunks COMPACT;"
```

COMPACTは約2.9秒で完了しました。

```bash:実行結果
mysql -h "tidb.${TAILNET}" -P 4000 -u root bench_wiki -e   0.01s user 0.01s system 0% cpu 2.857 total
```

続いてHNSWインデックスを作成します。

```bash
time mysql -h "tidb.${TAILNET}" -P 4000 -u root bench_wiki -e "
CREATE VECTOR INDEX idx_wiki_embedding_chunks_embedding
  ON wiki_embedding_chunks ((VEC_COSINE_DISTANCE(embedding))) USING HNSW;"
```

DDL全体は71.2秒で完了しました。1万ベクトル時の7.9秒に対して約9倍で、データ量10倍に対してほぼ線形です。

```bash:実行結果
mysql -h "tidb.${TAILNET}" -P 4000 -u root bench_wiki -e   0.01s user 0.01s system 0% cpu 1:11.15 total
```

ビルド本体の時間はTiFlashのログから確認できます。TiFlashはログを標準出力ではなく`/data0/logs`配下のファイルへ出力するため、Pod内のログファイルをgrepします。

```bash
kubectl -n tidb-cluster exec basic-tiflash-0 -c tiflash -- \
  sh -c 'grep -h EnsureStableLocalIndex /data0/logs/*.log' | tail -4
```

Stable層の2つのDMFileに対するビルドが並列に走り、ビルド本体は70.7秒でした。DDL全体の時間はほぼすべてがビルドです。

```text:TiFlashログ（抜粋）
[2026/07/21 23:20:29.660 +00:00] ["EnsureStableLocalIndex - Begin building index, dm_files=[dmf_40(v=0)]"]
[2026/07/21 23:20:29.660 +00:00] ["EnsureStableLocalIndex - Begin building index, dm_files=[dmf_41(v=0)]"]
[2026/07/21 23:21:36.589 +00:00] ["EnsureStableLocalIndex - Finish building index, dm_files=[dmf_40(v=0)]"]
[2026/07/21 23:21:40.327 +00:00] ["EnsureStableLocalIndex - Finish building index, dm_files=[dmf_41(v=0)]"]
```

ビルドの進捗は`INFORMATION_SCHEMA.TIFLASH_INDEXES`で確認できます。

```bash
mysql -h "tidb.${TAILNET}" -P 4000 -u root -e "
SELECT TIDB_DATABASE, TIDB_TABLE, INDEX_NAME,
       ROWS_STABLE_INDEXED, ROWS_STABLE_NOT_INDEXED,
       ROWS_DELTA_INDEXED, ROWS_DELTA_NOT_INDEXED,
       ERROR_MESSAGE
  FROM INFORMATION_SCHEMA.TIFLASH_INDEXES
 WHERE TIDB_DATABASE = 'bench_wiki'
   AND TIDB_TABLE = 'wiki_embedding_chunks';"
```

`ROWS_STABLE_NOT_INDEXED`と`ROWS_DELTA_NOT_INDEXED`がともに0で、`ERROR_MESSAGE`が空であれば、すべてのベクトルに対するインデックス構築が完了しています。

```bash:実行結果
+---------------+-----------------------+-------------------------------------+---------------------+-------------------------+--------------------+------------------------+---------------+
| TIDB_DATABASE | TIDB_TABLE            | INDEX_NAME                          | ROWS_STABLE_INDEXED | ROWS_STABLE_NOT_INDEXED | ROWS_DELTA_INDEXED | ROWS_DELTA_NOT_INDEXED | ERROR_MESSAGE |
+---------------+-----------------------+-------------------------------------+---------------------+-------------------------+--------------------+------------------------+---------------+
| bench_wiki    | wiki_embedding_chunks | idx_wiki_embedding_chunks_embedding |              100000 |                       0 |                  0 |                      0 |               |
+---------------+-----------------------+-------------------------------------+---------------------+-------------------------+--------------------+------------------------+---------------+
```

インデックスファイルのサイズも確認します。パスの`t_438`はテーブルIDです。

```bash
kubectl -n tidb-cluster exec basic-tiflash-0 -c tiflash -- \
  sh -c 'ls -la /data0/db/data/t_438/stable/dmf_*/idx_*.vector'
```

```bash:実行結果
-rw-r--r-- 1 root root 416675040 Jul 21 23:21 /data0/db/data/t_438/stable/dmf_40/idx_4.vector
-rw-r--r-- 1 root root 417026752 Jul 21 23:21 /data0/db/data/t_438/stable/dmf_41/idx_4.vector
```

2ファイル合計で約834MBです。1万ベクトル時の79.6MiB（約83.5MB）に対して約10倍と、こちらもデータ量に対して線形で、ベクトル本体（約820MB）とほぼ同サイズです。ビルドを含むTiFlash Podの累積`memory.peak`は5.98GiBで、8GiBの上限内に収まりました。

## 検索クエリ

検索に使うクエリベクトルは、ドキュメント投入時とは異なり`mode=query`で生成します。

```bash
QVEC=$(curl -s -X POST "http://plamo-embedding.${TAILNET}/embed" \
  -H 'Content-Type: application/json' \
  -d '{"text":"日本の城の石垣の構造","mode":"query"}' \
  | jq -c '.vector')
```

### パターン1: TiKVでの全件走査

`READ_FROM_STORAGE`ヒントで、行指向ストレージのTiKVを指定します。

```sql
SELECT /*+ READ_FROM_STORAGE(TIKV[c]) */
       page_id, title,
       VEC_COSINE_DISTANCE(embedding, '${QVEC}') AS distance
  FROM wiki_embedding_chunks AS c
 ORDER BY VEC_COSINE_DISTANCE(embedding, '${QVEC}')
 LIMIT 10;
```

`EXPLAIN ANALYZE`では、`TopN`と`TableFullScan`が`cop[tikv]`で実行されていることを確認します。

### パターン2: TiFlashでの全件走査

TiFlashを指定したうえで、`ORDER BY`式の末尾に`+ 0`を付けます。距離の順序は変えずにHNSWインデックスの適用条件から外し、列指向ストレージのTiFlashで全件走査を実行します。

```sql
SELECT /*+ READ_FROM_STORAGE(TIFLASH[c]) */
       page_id, title,
       VEC_COSINE_DISTANCE(embedding, '${QVEC}') AS distance
  FROM wiki_embedding_chunks AS c
 ORDER BY VEC_COSINE_DISTANCE(embedding, '${QVEC}') + 0
 LIMIT 10;
```

`EXPLAIN ANALYZE`では`mpp[tiflash]`で実行され、`annIndex`が表示されないことを確認します。この結果を、ANNの再現率を計算する際の正解集合として使用します。

### パターン3: HNSWインデックスによるANN検索

パターン2から`+ 0`を外すと、TiFlashのHNSWインデックスが利用されます。

```sql
SELECT /*+ READ_FROM_STORAGE(TIFLASH[c]) */
       page_id, title,
       VEC_COSINE_DISTANCE(embedding, '${QVEC}') AS distance
  FROM wiki_embedding_chunks AS c
 ORDER BY VEC_COSINE_DISTANCE(embedding, '${QVEC}')
 LIMIT 10;
```

HNSWインデックスが使われている場合、`EXPLAIN ANALYZE`の`TableFullScan`に`annIndex:COSINE(..., limit:10)`が表示されます。インデックスを利用していてもexecutor名は`TableFullScan`のままである点に注意してください。

## 計測方法

意味の異なる20個の検索文を用意し、各クエリについて3パターンを1回ずつ実行します。PyMySQLの持続接続を使用し、接続確立やコマンド起動の時間は計測から除外します。

同じクエリを繰り返すと、TiDBのcoprocessor cacheによってTiKV経路だけが不自然に高速化されます。そのため、同一クエリの繰り返しでは試行回数を増やしません。

各パターンの検索レイテンシを集計し、TiFlash全件走査の上位10件を正解集合として、ANNが取得できた件数から再現率を計算します。

計測には次の`bench_search.py`を使用します。20個の検索文もこのスクリプトに含まれています。

:::details bench_search.pyの全文
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

```bash
uv run bench_search.py
```

```bash:実行結果
recall@10=0.80 日本の城の石垣の構造
recall@10=1.00 相対性理論における時間の遅れ
recall@10=0.70 江戸時代の参勤交代の制度
recall@10=1.00 ワクチンで免疫がつく仕組み
recall@10=0.70 新幹線が開業するまでの歴史
recall@10=1.00 火山の噴火が起きる仕組み
recall@10=0.90 平安時代の貴族の暮らし
recall@10=1.00 サッカーのオフサイドのルール
recall@10=0.70 株式会社と合同会社の違い
recall@10=1.00 太陽系の惑星の並び順
recall@10=1.00 発酵食品と微生物の関係
recall@10=1.00 CPUが計算を実行する仕組み
recall@10=1.00 織田信長と本能寺の変
recall@10=1.00 地震の震度とマグニチュードの違い
recall@10=1.00 世界遺産に登録される条件
recall@10=0.90 オリンピックの開催地の決め方
recall@10=0.90 日本酒の醸造工程
recall@10=0.90 遺伝子の突然変異と進化
recall@10=0.80 気候変動が農業に与える影響
recall@10=1.00 印象派の画家と代表作
mean recall@10=0.92 (n=20)
query_embed    min= 730.7ms p50= 762.8ms p95= 808.0ms max= 808.1ms
tikv_full      min= 323.5ms p50= 341.5ms p95= 428.3ms max= 458.4ms
tiflash_exact  min= 209.5ms p50= 223.3ms p95= 263.8ms max= 306.7ms
tiflash_ann    min=  32.0ms p50=  33.6ms p95= 114.2ms max= 284.7ms
```

## 計測結果

前記事の1万ベクトル時の実測と比較します。

| 経路 | 1万ベクトル（前記事） | 10万ベクトル（p50） | 比 |
| --- | ---: | ---: | ---: |
| TiKV全件走査 | 187.7ms | 341.5ms | 約1.8倍 |
| TiFlash全件走査 | 73.4ms | 223.3ms | 約3.0倍 |
| ANN（HNSW） | 21.7ms | 33.6ms | 約1.5倍 |
| recall@10（平均） | 0.83 | 0.92 | — |

![1万から10万ベクトルへのp50レイテンシの伸びと線形外挿の比較](https://res.cloudinary.com/dkerzyk09/image/upload/v1784677454/blog/2026-07-20-selfhosted-tidb-wikipedia-100k-vectors-full-scan-vs-ann/vpytklw0zazsuhfifzw0.png)
*実測のp50レイテンシ（実線）と、レイテンシがデータ量に比例して10倍になると仮定した線形外挿（点線）。全件走査の2経路とも外挿を大きく下回った*

最も意外だったのは全件走査です。1万ベクトル時の実測を単純に線形外挿すると、データ量10倍でTiKVは約1.9秒、TiFlashは約0.7秒になる計算でしたが、実測はTiKVで約1.8倍、TiFlashで約3.0倍に留まりました。距離計算の総量は確かに10倍になっているものの、データがTiKVでは複数リージョン、TiFlashでは複数のDMFileへ分割され、並列に処理されるため、単発クエリのレイテンシは線形には伸びなかったと考えられます。ただしこれは並列化でCPU時間を壁時計時間へ押し込んでいるだけなので、総仕事量`O(N)`は変わらず、同時検索数が増えた場合のスループットには依然として効いてくるはずです。

ANNはp50で33.6msとほぼ横ばいでした。p95が114.2msと少し尾を引いていますが、それでも全件走査のp50より高速です。recall@10の平均は0.83から0.92へ向上しました。データが増えて検索が難しくなるという事前の想定とは逆の結果で、この規模ではHNSWの近似が全件走査の上位10件を9割以上再現しています。

E2Eの内訳では、クエリの埋め込み生成が約763ms（p50）と依然として支配的です。検索自体をANNで33.6msまで縮めても、体感を決めるのはCPU推論の埋め込み生成側にあります。

## まとめ

### 実測値と結論

前記事の1万ベクトルから10倍の10万ベクトル（39,480記事、約8,256万トークン）へスケールし、投入・HNSWインデックス構築・3経路検索を実測しました。本記事で得た実測値の一覧です。

| 項目 | 実測値 |
| --- | --- |
| 埋め込み生成（投入） | 89,983ベクトル / 89時間0分（0.281ベクトル/s、同時4）。OOMKill・リトライ0回 |
| PLaMo Podメモリ（jemalloc化後） | `memory.peak` 5.16 / 5.45GiB（glibc時代は7.31 / 7.81GiB）。投入完了後はベースラインへ復帰 |
| COMPACT | 約2.9秒 |
| HNSWインデックス構築 | DDL全体71.2秒（ビルド本体70.7秒）。1万時7.9秒の約9倍でほぼ線形 |
| HNSWインデックスサイズ | 約834MB。1万時79.6MiBの約10倍で、ベクトル本体とほぼ同サイズ |
| 検索レイテンシ（p50） | ANN 33.6ms < TiFlash全件走査 223.3ms < TiKV全件走査 341.5ms |
| recall@10（ANN） | 0.92（1万時の0.83から向上） |
| 検索E2E（クエリ埋め込み込み） | 約0.8秒（支配項はCPU推論の763ms） |

一番の発見は、前記事の「全件スキャンのレイテンシが線形に伸びる」という見込みが外れたことです。データ量10倍に対して単発クエリのレイテンシはTiKVで約1.8倍、TiFlashで約3.0倍に留まりました。リージョンやDMFileへの分割と並列処理が伸びを吸収するためで、単発レイテンシだけを見るなら10万ベクトルでも全件走査は223〜342msと許容圏に残ります。ただし距離計算の総量`O(N)`は変わらないため、同時検索が増えればCPUが飽和し、この「見かけの横ばい」は成立しなくなるはずです。

その上で、ANNの優位は明確になりました。検索は33.6msと全件走査の1/7〜1/10で、recall@10は0.92へむしろ向上し、構築コストは71.2秒と834MBに収まります。1万時点の結論は「全件スキャンで十分」でしたが、10万ではHNSWインデックスを張らない理由が薄くなります。

一方で検索の体感を決めるのは依然としてクエリ埋め込みのCPU推論（約763ms）です。検索経路の最適化はここから先、E2Eにはほぼ効きません。

### 今後

次の10倍（100万ベクトル）は、今回と同じやり方が通用しない領域です。投入はCPU推論0.281ベクトル/sの単純計算で約37日となり現実的でないため、埋め込み生成だけWorkers AIへ寄せる折衷構成（前記事のまとめ参照）が前提になります。今回の10万ベクトル分（約8,256万トークン）でもWorkers AIなら約$1.6で、89時間との対比はコスト面でも決定的です。HNSWは線形なら構築約12分・インデックス約8GBと、TiFlash Podのメモリ上限8GiBに並ぶ規模になるため、メモリ設計が最初の論点になります。あわせて、単発では見えなかった同時実行スループットの劣化も、この規模で計測する予定です。
