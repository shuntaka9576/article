---
title: "CloudinaryのFreeプランのクレジットが枯渇した話"
description: ""
publish: true
tags:
  - "tech/cloudinary"
  - "tech/aws/s3"
---


会社のブログ記事の画像を個人のCloudinaryにアップロードしてしまいクレジット枯渇、一旦$99課金して静観することにした。経緯は個人のブログに上げる予定だったのを急遽会社のブログに上げることにしたから。記事は画像数も画像の容量も大きく、かなりネットワーク帯域幅を使用してしまった。

圧倒的超過🥺
![image](https://github.com/shuntaka9576/public-image/assets/12817245/d598a843-7e43-450b-8f62-3a74acd54ee2)


ネットワーク帯域幅が、245.37 GB🥺 Creditの表記は課金後の値。
![image](https://github.com/shuntaka9576/public-image/assets/12817245/304a6636-d87d-40df-9f4b-8b9ed5cbd66e)



## 暫定対応

* $99のPlusプランに加入(これでもクレジットは超過している、6月まで一部画像が見れなくなることは許容)

![image](https://github.com/shuntaka9576/public-image/assets/12817245/00fa4efc-880c-43db-9ba7-ae7f61d7773e)


* PlusプランのS3バックアップ機能を使って画像を退避(すぐSyncされて流石だなと思った)
* 問題になった記事の画像の配信元を変更


## まとめ

CloudinaryはOGPタイトル埋め込みの変換だけで使うのがいいな、、




