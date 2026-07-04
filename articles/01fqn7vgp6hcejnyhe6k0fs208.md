---
title: "2021年の振り返り"
type: "tech"
description: "お世話になった皆様、感謝です！本記事では、2021年の活動を軽く振り返ります。"
thumbnail: "https://res.cloudinary.com/dkerzyk09/image/upload/v1640324333/blog/01fqn7vgp6hcejnyhe6k0fs208/j4og57fak74bzlz2vuqb.png"
publish: true
tags:
  - "misc/振り返り"
  - "tech/neovim"
  - "tech/iot"
---


# はじめに

今年も残すところ7日です。私は今日で仕事納めかつ、年末は引越し準備でブログをまったり書いている時間がなく、いつもより早めに書くことにしました！

本記事では、2021年の活動を軽く振り返ります。

# 今年の活動(技術)

## 個人ブログ
2021年上半期はVercelとNext.js、サーバーサイドはサーバーレス構成でISRする自作ブログの改善をしていました。

改善内容としては下記の項目で、某投稿サイトで使いやすかった機能を参考に実装しました。

* マークダウンパーサー、CSSの改善
* Neovimでのリアルタイムプレビューできる[オレオレプラグイン](https://github.com/hozi-dev/preview-hozi-dev.nvim)の実装
* ダークモードの実装
* プライベートな記事を認証付き(Firebase Authentication)で見られる仕組みの導入
* GitHub Appsを使ったリポジトリとブログの同期システム 

実際に書いたコードは、[公開](https://github.com/hozi-dev)しているので、気になる人がいるわけないと思いますが、参考になればと思います。ここら辺の話の詳細は[Next.jsとサーバレスAPIで自分のブログを作った話](https://shuntaka.dev/shuntaka/articles/01f07hctzhjcwtdq4h6ew9stk8)に書いたので、気になる方は見てみてください。

PV数は大体月500PVくらいです。まだまだですが、自分の備忘録として書くというスタンスでこれからも続けます。
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1640317706/blog/01fqn7vgp6hcejnyhe6k0fs208/x7exp0dr4mz9cn1lv57f.png)

期間は2021/01/01 - 2021/12/23の記事別のPV内訳は、下記の通り。力を入れて書いた記事ほどPVが高くて納得しました。。

|記事タイトル|PV数|
|---|---|
|[Next.jsとサーバレスAPIで自分のブログを作った話](https://shuntaka.dev/shuntaka/articles/01f07hctzhjcwtdq4h6ew9stk8)|697|
|[GitHubとの連携手段(OAuth Apps, GitHub Apps)を整理する](https://shuntaka.dev/shuntaka/articles/01ezm5k2rt1jm6zbsewm33r0xw)|526|
|[AWS IoTで独自CA証明書を使ってMQTT通信を試してみる](https://shuntaka.dev/shuntaka/articles/01f4gnep6herhgy449er48g9c0)|422|
|[OpenSSLでJWTの作成、検証ロジックを確認する](https://shuntaka.dev/shuntaka/articles/01f4xw3pwm7tcrdswyzqsft5zs)|282|
|[npmモジュールをモノレポで管理し、煩雑なリリース業務を効率化する](https://shuntaka.dev/shuntaka/articles/01evbw029qzxavp20erstgvm5r)|269|
|[Vim/Neovimプラグイン開発サイクルをまとめてみる](https://shuntaka.dev/shuntaka/articles/01f0n3x0afc5wt54qeaz77zvw4)|257|
|[App EngineとCloud CDNでstale-while-revalidate可能な環境を構築する](https://shuntaka.dev/shuntaka/articles/01f2wwqs2jcdgc7fh8bmhnewk6)|253|
|[next/linkのpassHref属性がどのような場合に必要か整理、検証してみる](https://shuntaka.dev/shuntaka/articles/01f3qsp7vz8dtetg5cq27ercna)|217|
|[lernaでモノレポで管理したパッケージをnpmに公開する際の注意点](https://shuntaka.dev/shuntaka/articles/01ev3p1knggn1wwsg0n0e98915)|207|


## その他
ほぼないので、ブログところで水増ししました。

強いてあげるとするなら[Neovimのinit.lua](https://github.com/shuntaka9576/dotfiles/blob/master/lua/init.lua)化とプラグインをlua製に総入れ替えしました。luaでDeepLを使った翻訳プラグインを作ったりしましたが、やりかけで年内公開まで至らずという感じです。。Neovimのluaプラグインはかなりエコシステムが充実しているように感じました！

## 仕事
2020年に引き続きIoTサービスの設計、実装を行っていました。大きく変化した点としてはAWS(クラウド)側だけでなく、ハード側のアプリを書いている比重が多かった。(ラズパイとPythonなので組み込みではないです)

少し耐性がついた結果、M5Stack Core2 for AWSを購入して色々記事を書きました。

* [NeovimでM5Stack Core2ファームウェア開発を快適に行う環境を作った](https://shuntaka.dev/shuntaka/articles/01ffwa0x782te58803721b1czg)
* [ESP-IDFの導入と使い心地を試す](https://shuntaka.dev/shuntaka/articles/01fg0ayqeqbf4rbfzc7gev1t0k)
* [[M5Stack Core2] PlatformIOでesp-aws-iotのサンプルsubscribe_publishを動かしてみる](https://shuntaka.dev/shuntaka/articles/01fgdc0bawyb6d34gs54vxgpg9)

今年は、個人で18記事と会社で3記事の合計21記事書きました。

# 技術以外

## 運動
5月,6月は、平日1万/休日1万5千歩くらい歩いていました。おかげで10kgくらい痩せました。登山グッズも買ったのですが、1回大山行ったきりです。。最近はサボり気味ですが、また太ってきたら再開しよーかなーと思っています。

# 2022年向けて
ざっくりですが、2022年は下記をテーマにやっていきたいなと思います。

* 本ブログは他媒体と比較しても一番使いやすいので、引き続き記事を書いていく
* 技術の素振り場所を増やす
  * 個人ブログとは別にWebアプリを1つを作り、運用する
    * WASM, Deno, CDN関連Saas(Cloudflare Workers, Workers KVとか)あたりが組み込めるテーマがいいな..
* M5Stack(ESP32, Arduino)を使って何かアプリを作る(ｽﾀｯｸﾁｬﾝとか..)
* 本ブログの採用技術の見直し、素振り
  * 例えば、VercelをAmplifyにする、DynamoDB, Firebase Authenticationをsupabaseにする等

# 最後に

ちなみにサムネイルは大さん橋埠頭から撮った写真です。実家から歩いて90分くらいのところにあり、引っ越すともうこの景色は歩いて見れなくなるなーという気持ちで今年の振り返りのサムネイルにしました。どうでもいいですね。

ということで、1年間お世話になりましたー。良いお年を〜。
