---
title: "Cloudflare Registrarからムームーへドメイン移管してみる"
description: "Cloudflare Registrarへ移管するパターンはよくありそう。今回は逆です。Cloudflareはfreeだとネームサーバー設定の制約があるのでムームーに移管しました"
publish: true
tags:
  - "tech/cloudflare"
  - "tech/aws"
  - "tech/開発環境"
---

## はじめに

Cloudflare Registrarへ移管するパターンはよくありそう。今回は逆です。Cloudflareはfreeだとネームサーバー設定が変更できないので移管しました。卸価格だから、このサービスモデルを維持する条件として、Cloudflareの提供するDNS（ネームサーバー）を利用させたいんだと思う。サブドメインのみを権限委譲（DNS Delegation）する方法はあるので、取っちゃった人はこれで良いと思う。

多くの人はRoute53で自分のサービス公開せず、Cloudflareで完結しそうなのでこういう機会はなさそう。

とはいえ今回作業の時刻も書いたので、ドメイン移管にどれくらい時間かかるかという観点でみてもいいかも。

## 作業

(14:33) 作業。Cloudflare側でDomainをUnlockする。するとコードが出てくる。
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1768717120/blog/20260118-transfer-from-cloudflare/xoemotxksiit6wvxncwk.png)


(14:38) 作業。ムームー側で手続きを走らせる。ここで先ほどのコードとドメインを入力する。
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1768717250/blog/20260118-transfer-from-cloudflare/sc581zra4rvqxp0oecy2.png)

(14:40) すぐに受付メールが来る
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1768717430/blog/20260118-transfer-from-cloudflare/levubq8nqrvzesfnhq2r.png)

(15:00) 気づいたらすでにムームー側のダッシュボードから確認できるようになっている。
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1768717596/blog/20260118-transfer-from-cloudflare/wgdkftqyfkftzqsesvzc.png)

ここから少し待つことになった。多分Cloudflare側の応答待ちなんだと思う。[参考 | ムームードメイン移管完了までの流れ](https://support.muumuu-domain.com/hc/ja/articles/360046455934-%E7%A7%BB%E7%AE%A1%E5%AE%8C%E4%BA%86%E3%81%BE%E3%81%A7%E3%81%AE%E6%B5%81%E3%82%8C)の項6に当たる。

> 6. 『現在お客様のドメインを管理している会社』が『弊社レジストラ』に移管の連絡を行う。

(15:58) Cloudflareからメールを確認
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1768719462/blog/20260118-transfer-from-cloudflare/rkksrsqpwvf5ec1xipgu.png)

(16:01) 作業。Cloudflareのマネコンでドメイン移管の承認をする。
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1768719642/blog/20260118-transfer-from-cloudflare/n8p1eyby0s6y3gvuhas4.png)

> 移管をすぐに進める場合は、承認（リリース）のお手続きをお願いします。手動での承認が行われない場合でも、5日後に自動的に移管が完了します。

前倒しで出来る的なことが書いてある
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1768719758/blog/20260118-transfer-from-cloudflare/wyganekejidy7qcf9avo.png)

16ドルでCloudflareに戻って来れるよ的なことが書いてあり、抜け目ないね。。😂
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1768719816/blog/20260118-transfer-from-cloudflare/hewekw4ltcxt2dane3z3.png)

(16:02) 完了するとCloudflare側から移管手続きを開始するよメールが来る。Cloudflare側の作業が終われば、ムームー公式の移管の流れ項6が開始される感じかなぁ
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1768720015/blog/20260118-transfer-from-cloudflare/tkdofu3oioq1ysomzdhg.png)

(16:26) レジストリの承認待ちの状態にステータスが変更。ムームー公式の移管の流れ項6が完了した気配を感じる。
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1768721196/yrf6ocfg3pcnvwsdrcuw.png)

(16:40) 移管進行状況が空っぽになった。
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1768722044/jhmw3o4cijnn6itgttkk.png)

(16:50) 移管進行状況が「移管まもなく完了します。」に変化。
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1768722680/hgc13h2p8w0nfbwecc2x.png)

(17:28) まもなく完了からが長く終わらない。。随時追記予定。。😭😭😭

(18:20) 16時ちょうどにCloudflareのマネコンから移管の承認をして約2時間半弱でムームー側から私へ移管完了通知が来ました。
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1768728207/t7u539jd0sgux2phqhmx.png)

更新するボタンを押下して、移管費用を決済したら完了です🥳🥳🥳自分はクレカ決済にしているので、おさいぽからクレカに選択切り替えて決済しました。

![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1768728207/osot4rztvxbovwa09zvn.png)

ほぼ作業系は通知来てからすぐ対応したので、14時33分に始めて18時20分に終わったので大体4時間弱で全ての作業が完了しました🎉 1つの目安にしてください！


## さいごに

ドメイン買うなら自分はここら辺気にすることにする

* 初期費用
* 更新費用
* Whois代行できるか
* サービス維持調整費を考慮(ムームーはあるので、GMO系はあるんじゃないかな。さくらはないっぽいけど高め。トータルどうかはわからない。)
* Route53使うならネームサーバー設定を変更できるか(new!)

初期費用安くても、更新費高いケースもあるので注意!

https://x.com/shuntaka_jp/status/2012765878576902307

CleanShot Xは画像名に時刻埋め込んでくれるので、作業と時刻の付き合わせが出来て便利でした⭐️
