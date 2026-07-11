---
title: "Cloudflare PagesにApexドメインを設定しようとして嵌った話"
description: "寝たら解決しました!!"
publish: true
tags:
  - "tech/cloudflare"
  - "tech/dns"
---

# はじめに
Cloudflare Pagesのクソ嵌り事案の備忘録です。嵌るのは多分私くらいなので、参考までに。

# やりたいこと
* Cloudflare PagesでホスティングしているサイトにApexドメイン(※)を適用したい

※ 本記事でApexドメインはwwwなしの、`example.com`ようなドメインを指すこととします

# 前提
* Google Domainsでドメインを取得済
* Google Domainsの管理コンソールからネームサーバーをCloudflareに変更済
* Cloudflare Pagesにアプリをホスティング済
* Cloudflare DNSにドメインを登録済
* Cloudflare PagesのCustom domainsにApexドメインを登録(<-クソ嵌り)

# 事象
:::message alert
実際に起きた事象を後日忠実に再現した内容です
:::

Cloudflare PagesでCustom domainsの設定
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1628293814/blog/01fcexkj03t5wr0hkjt37t7dcz/ybh8ef0ihp8bt305c6tm.png)

Cloudflare PagesでApexドメインの設定
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1628293932/blog/01fcexkj03t5wr0hkjt37t7dcz/ksvbp3h0ilzdnni1uenn.png)

「Get started with Cloudflare DNS」と表示。Cloudflare DNS設定へ遷移します。
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1628294099/blog/01fcexkj03t5wr0hkjt37t7dcz/p2z7mjr4jhkpgkbvghy1.png)


前述の通りCloudflare DNSにドメイン設定を行っている状態で上記メッセージが出ます。「Begin DNS transfer」を押しても、既にそのドメインは登録済とされ、エラーが出ます。しばらくこの状態が続きました。

サブドメインは登録出来たため、DNSの反映待ちということはないと推測していました。

# 解決した話
諦めてその日は眠り翌朝設定を確認したところ設定が出来るようになっていました。幾つかコンソール上で変化があった点をメモします。

Overview画面の変化
**変更前**
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1628294662/blog/01fcexkj03t5wr0hkjt37t7dcz/xlilm0cfhqo354xirjyh.png)


**変更後**
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1628294752/blog/01fcexkj03t5wr0hkjt37t7dcz/nakrg1z3a2oae9aynjkx.png)


再度Apexレコードの登録を試します。前項の「Cloudflare PagesでApexドメインの設定」後の画面に変化があります
Cloudflare DNSへレコードが自動で登録出来るようになっています。「Activate domain」押下して、設定は完了です
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1628294905/blog/01fcexkj03t5wr0hkjt37t7dcz/srmgeooblk385uc0m8fs.png)

Custom domainsの設定を確認しても「Activate」状態になっており、実際Cloudflare Pagesでホスティングしたサイトが閲覧できました
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1628295132/blog/01fcexkj03t5wr0hkjt37t7dcz/lls40jzk7frt0b7clk4v.png)

また、上記画像の赤枠部が設定可能かどうかの判断材料になると思っています。Custom domainで、Apexドメインが設定できないとき、切り分けとして適当なサブドメインを設定した際に赤枠のオプションは存在しませんでした。

推測ですが、Cloudflare PagesにClodflare DNSの設定が反映されておらず、Cloudflare DNS設定へ誘導されたと考えています。

# 最後に

果報寝て待て

# 参考

[Getting Started with Cloudflare Pages, Google domains and Gmail](https://dev.to/julianjurai/getting-started-with-cloudflare-pages-google-domains-and-gmail-fhf)


