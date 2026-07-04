---
title: "今週のきづきメモ(技術編)"
type: "note"
description: ""
thumbnail: ""
publish: true
tags:
  - "tech/aws"
  - "tech/技術メモ"
---

## LLM関連

### Windsurf DeepSeek v3使い放題

WindsurfからDeepSeek v3が使い放題になった。(マルチモーダルなし、Proモード限定)


## ECS関連

久しぶりに検討してすっかり忘れていたのでメモ

AWSサービスをNATゲートウェイ経由で通信させないためには、VPCエンドポイントが必要。
ただし、通信するすべてのAWSサービスに対してVPCエンドポイントを用意すればいいとは限らず、VPCエンドポイントの起動時間、データ処理量の2つを考慮し、損益分岐点を計算する必要がある。

ECRの接続にはECRとS3のVPC Endpointが必要なので注意！ただしイメージの取得は、S3経由になるので無料だと考えられる。

> ECRのVPCエンドポイントは、APIリクエストや認証、イメージメタデータの取得など、コンテナイメージ自体の大容量のデータ転送には関与しません。実際のイメージレイヤーのデータはAmazon S3から取得されるため、ECRのVPCエンドポイントを通るトラフィック量は非常に少なくなります。ただし、少量でもエンドポイント自体の利用料金が発生する可能性があるので、最新の料金情報は確認するのが良いでしょう。

とりあえずS3とDyanmoDBは無料なので思考停止で立てて良いと思う！

## 参考記事

* https://qiita.com/Dai_Kentaro/items/29e66962973f67d6e8bf
* https://zenn.dev/strix/articles/5c698c65817308
