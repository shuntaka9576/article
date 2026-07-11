---
title: "Next.jsとサーバレスAPIで自分のブログを作った話"
description: "昨年から様々なSaaSを活用して、自作のブログを改善しており、本記事では裏側の構成について解説する"
publish: true
tags:
  - "tech/next.js"
  - "tech/aws/lambda"
  - "tech/aws/dynamodb"
  - "tech/github"
---

# はじめに
昨年2020年の年末から個人のブログを定期的に改善しており、ある程度快適なアウトプット環境ができたので紹介する。

ざっくり書くと、データストアにDynamoDBを利用して、サーバレスAPIをAWS上に、Next.jsアプリをVercel上でホスティングしている。サーバレスAPIは貧者の技術で、開発環境/本番環境を作って休日中ずっと開発してても大体月3ドルくらいですんだ。

Next.jsでブログ作ってみました記事は、だいたいNext.jsアプリと同じリポジトリにマークダウンをコミットしてSSGするようなのばかりで、記事更新のたびにデプロイが必要...。Next.jsとVercelを使うなら、ISR使ってみたいし、独自的な機能をブログに付けたいこともあり、サーバサイドはREST APIにしつつ、コストを抑えるためサーバレス構成にした。

それなりに需要があるんじゃないかという目論見と、自分の備忘のためシステム構成を記事にすることにした。

# 要件
主に下記の機能を実現している。それぞれ詳しく解説できないので、気になる機能があったら気軽に[こちら](https://github.com/hozi-dev/article/issues)でissueとして起票してほしい。

* 認証
* マークダウンを使った記事の記述
* OGP画像の自動生成
* サムネイルの設定
* GitHubのリポジトリにpushしたら、記事更新可能
* サイトから認証を経て、リポジトリ連携設定可能
* サイトから認証を経て、リポジトリ連携解除可能
* 個人以外でもユーザー登録したら、普通に使える

# 構成概要
利用しているサービスは下記の図の通り。

![構成](https://res.cloudinary.com/dkerzyk09/image/upload/v1615111924/blog/01ezsr2jdx19bg00pgwt1rnsk6/hqdqjtntcjed43d0nnjp.webp)

# フロントエンド

## ページ全体構成
フロントエンドはNext.js。ホスティングにVercelを利用。ページ構成は下記。

|ページ名|パス|認証|API fetch|
|---|---|---|---|
|loginページ|/login||
|トップページ|/||ISR|
|記事詳細ページ|/[userName]/articles/[slug]||ISR|
|ダッシュボードページ|/dashbord|◯|CSR|
|GitHub Appsコールバックページ|/github/app/callback|◯|CSR|

## トップページ
![トップページ](https://res.cloudinary.com/dkerzyk09/image/upload/v1615082644/blog/01ezsr2jdx19bg00pgwt1rnsk6/vontcifux30p4hlitkpj.webp)
バックエンドの記事一覧取得APIをVercelサーバサイドサイドでfetchして、ISRで配信している。

デザインは定番の1〜3カラムカード表示で、レスポンシブ対応している。サムネイルを設定しないとOGP用に自動生成された画像が表示される。画像はカードのタイトルと被っているので正直微妙。テンプレートが設定できたり、Zennみたいに絵文字が設定できるように改善したいなーと感じている。

## 記事詳細ページ
![記事詳細ページ](https://res.cloudinary.com/dkerzyk09/image/upload/v1615083192/blog/01ezsr2jdx19bg00pgwt1rnsk6/uspkfahiltj22mbzqzid.webp)

トップページ同様、Vercelサーバサイドでfetchして、ISRで配信している。

[zenn-dev/zenn-editor](https://github.com/zenn-dev/zenn-editor)のやり方を参考に、詳細ページで利用するマークダウンパーサーとCSSを[hozi-dev/hozi-dev-packages](https://github.com/hozi-dev/hozi-dev-packages)にまとめた。

lernaを使ったモノレポ構成で管理している。課題は、npmにpublishしないとパッケージを使う側と結合テスト出来ない点。モノレポにしなければnpmの場合、雑にリポジトリ名とブランチ名指定でパッケージインストール可能なので、悩みどころ。

lerna関連では、過去に記事を書いたのでこちらも参考にすると良いかもしれない。
* [lernaでモノレポで管理したパッケージをnpmに公開する際の注意点](https://shuntaka.dev/shuntaka/articles/01ev3p1knggn1wwsg0n0e98915)
* [npmモジュールをモノレポで管理し、煩雑なリリース業務を効率化する](https://shuntaka.dev/shuntaka/articles/01evbw029qzxavp20erstgvm5r)

## ダッシュボードページ
管理ページで、著者しかみれないページなので、実質ユーザーには縁のないページ。(Firebas Authenticationでサインアップすれば同じことができるようにはしています)

リポジトリの連携、連携解除設定するのに利用。ログインからリポジトリ連携、解除は下記のgifのような流れとなる。(ボタンを押したのか分かり辛かったり、デザインやっつけですが...)

![dashbord](https://res.cloudinary.com/dkerzyk09/image/upload/v1615084489/blog/01ezsr2jdx19bg00pgwt1rnsk6/zbeonycrwk2ntm9zvgtp.gif)

ダッシュボードページでは、ユーザー情報を取得したり、連携しているリポジトリの情報を取得するAPIをクライアントサイドでfetchしています。

# 活用している主なSaaS
活用しているSassに関して簡単に紹介する

## Firebase Authentication
IDaaS。認証で利用。React Contextでユーザー情報やidTokenを各ページで取得して、認証が必要なAPIを叩いている。Firebaseはライブラリがしっかりしていて、IDaaSとしてうまく処理が隠蔽されているなと感じた。

## Cloudinary
画像処理SaaS。OGPタイトル埋め込みで自動生成する際には、定番(?)なサービス。下記ような形で、予め台紙となる画像をCloudinaryに保存しておけば、パスパラメータを使ってタイトルを埋め込める。

https://res.cloudinary.com/dkerzyk09/image/upload/s--d5U3lE5_--/c_fit,co_rgb:525457,l_text:notesansjpmid.otf_48_bold:GitHub%E3%81%A8%E3%81%AE%E9%80%A3%E6%90%BA%E6%89%8B%E6%AE%B5(OAuth%20Apps%252C%20GitHub%20Apps)%E3%82%92%E6%95%B4%E7%90%86%E3%81%99%E3%82%8B,w_600/v1/blog/og/ogp.webp

URLに着目してもらえるとわかる通り、`s--d5U3lE5_--`のような文字列がついておりこちらが署名になっている。パスパラーメーターの内容に対して署名付きURLが発行されているので、任意のユーザーに画像変換を悪用されない。

署名つきURLの発行は、後述するサーバレスAPI側のLambdaで発行している。Lambda側で署名付きURLを発行している処理は下記のような感じ。

```ts
import * as Cloudinary from 'cloudinary';

export const createSetedTitleSiginedUrl = (
  publicId: string,
  title: string,
  ext: string,
): string => {
  Cloudinary.v2.config({
    cloud_name: CLOUD_NAME,
    api_key: API_KEY,
    api_secret: API_SECRET, // KMS等のキーマネージャー側に保存するのが良い
  });

  const siginedUrl = Cloudinary.v2.url(`${publicId}.${ext}`, {
    sign_url: true,
    secure: true,
    type: 'upload',
    transformation: {
      secure: true,
      sign_url: true, // 署名付きにする
      transformation: [
        {
          overlay: {
            font_family: 'notesansjpmid.otf', // フォントを指定
            font_size: 48,
            font_weight: 'bold',
            text: title, // 画像に設定する文字列を指定
          },
          color: '#525457',
          width: 600,
          crop: 'fit',
        },
      ],
    },
  });

  return siginedUrl;
};
```

## GitHub Apps
リポジトリに記事をpushしたら、記事が格納されているDynamoDBを更新している。その仕組み実現するために、GiHub Appsを活用している。GitHub Appsはエンドユーザーがインストール時にGitHub Appsに対して認可(権限とその権限が使えるリポジトリ)を与えること出来る。その認可を使ってGitHub Appsは様々なサービスを提供可能。本ブログでは例えば下記のことをGitHub Appsで行っている。
* リポジトリpush時にこちらの設定したサーバーにwebhookしてもらう(このwebhookを契機にDBを更新)
* 認可のあるリポジトリを参照

# バックエンド

## API認証
前述の通りIDaasには、`Firebase Authentication`を採用。API認証に、Lambdaオーソライザを利用している。

IDトークンの検証に成功したら成功ポリシー、失敗したら失敗ポリシーを返す。成功ポリシーが返却されれば後続のLambdaが実行され、失敗ポリシーの場合は401 Unauthorizedを返す。

検証しているコードはこんな感じ。ライブラリ優秀。
```ts
import * as FirebaseAdmin from 'firebase-admin';

// (中略)

export const verifyToken = async (
  idToken: string,
): Promise<FirebaseAdmin.auth.DecodedIdToken> => {
  try {
    const decodeToken = await FirebaseAdmin.auth().verifyIdToken(idToken);

    return decodeToken;
  } catch (e) {
    if (e.code === 'auth/id-token-expired') {
      throw new FireBaseIdTokneExpiredError(idToken, e);
    }

    throw new FireBaseUnknownError(idToken, e);
  }
};
```

## 記事一覧取得
トップページで記事一覧を取得する際に実行されるAPI。Lambdaから記事テーブルに問い合わせて記事の一覧を取得する。

記事テーブル定義は下記。

|属性名|型|必須|説明|
|---|---|---|---|
|userId(PK)|string|◯|Firebase AuthenticationのUID|
|articleId(SK)|string|◯|slug的なやつ。userIdがPKなので、被りは気にしなくていい|
|content|string|◯|記事内容|
|title|string|◯|タイトル|
|type|string|◯|記事のタイプ(技術とかポエムとか)|
|(beta)category|Array|◯|
|createAt|number|◯|作成日時|
|updateAt|number|◯|更新日時|
|publishAt|string||公開日時.公開->非公開にしても消し込まれない|
|thumnail|string||記事のサムネイル|
|typePublishAt|string||type-publishAtをつなげた値.非公開にすると消し込まれる|

userId(SK) typePublishAt(SK)のGSIを作成している。typePublishAtは、{記事のタイプ}-{公開タイムスタンプ}形式。故にuserIdと記事タイプをbegins_withで指定してDyanmoDBに対してクエリすれば、指定した記事タイプの一覧がソート済(公開日時順)で取得できる。ページネーションも可能。問い合わせているコードは下記。

```ts
export const getArticleListByUserIdAndType = async (
  userId: string,
  type: ArticleType,          // 記事タイプを指定
  scanIndexForward?: boolean, // 降順,昇順を指定
): Promise<ArticleUserIdTypePublishAtIndexTableItem[]> => {
  const params: DynamoDB.DocumentClient.QueryInput = {
    TableName: ARTICLE_TABLE_NAME,
    IndexName: 'userIdTypePublishAtIndex',
    KeyConditionExpression:
      'userId = :userId AND begins_with(typePublishAt, :type)',
    ExpressionAttributeValues: {
      ':userId': userId,
      ':type': `${type}-`,
    },
    ScanIndexForward: scanIndexForward ?? true,
  };

  const response = await dynamodbDocumentClient.query(params).promise();

  if (!response.Items) {
    return [];
  }

  return response.Items.map((item) => {
    const artcleTableItem: ArticleUserIdTypePublishAtIndexTableItem = {
      userId: item.userId,
      articleId: item.articleId,
      content: item.content,
      title: item.title,
      type: item.type,
      updateAt: item.updateAt,
      createAt: item.createAt,
      category: item.category,
      description: item.description,
      typePublishAt: item.typePublishAt,
    };

    if (item.thumbnail) {
      artcleTableItem.thumbnail = item.thumbnail;
    }

    return artcleTableItem;
  });
};
```

## 記事単体取得
前述したuserIdとarticleIdで記事単体を取得する。マークダウンのHTMLパースはLamba側で実行してフロントはHTMLをレンダリングするだけにしている。

## GitHub Webhook受け取り
ZennのようなGitHub連携機能を付けたくて実装した。GitHub Appsをインストールしたリポジトリにpushすると、APIGatewayを通して本Lambdaが実行される。ロジックは別途記事を書く予定。

GitHub Appsとはなんぞや？という方は、以前投稿した[GitHubとの連携手段(OAuth Apps, GitHub Apps)を整理する](https://shuntaka.dev/shuntaka/articles/01ezm5k2rt1jm6zbsewm33r0xw)を参考にする良いかも。

## ユーザー情報取得
Firebase AuthenticationのUIDに紐づけた情報を取得するAPI。Lambdaからユーザーテーブルに問い合わせ取得した内容を返却する。
ユーザーテーブル定義は下記。

|属性名|型|必須|説明|
|---|---|---|---|
|userId(PK)|string|◯|Firebase AuthenticationのUID|
|field(SK)|string|◯|userIdに関する属性名|
|value|string|◯|fieldに対する値|

[GSI OVERLOADING](https://www.slideshare.net/slideshow/embed_code/key/q6PyeWlr3O8iPF?startSlide=33)を参考にした。filed(PK) value(SK)のGSIを貼っている。例えばageという属性があった場合、field=age, value=24でクエリすれば24歳のuserId一覧を取得出来る。

下記の値でFirebase AuthenticationのUIDを 引く or 逆引き する用途で利用。
* ブログに一意なユーザーID
* GitHub AppsのinstallationId

GSI OVERLOADINGは1つのGSIであらゆる属性からユーザーIDを**逆引き**出来るのが強みで採用している。

## GitHub Appsインストール
[GitHub Appsのインストールのフロー](https://shuntaka.dev/shuntaka/articles/01ezm5k2rt1jm6zbsewm33r0xw#%E3%83%95%E3%83%AD%E3%83%BC-1)のバックエンドの部分の処理を担う。
主な処理は下記。
* 前述のユーザーテーブルにinstalltionIdとFirebase AuthenticationのUIDを紐づけて保存する

不特定多数がGitHub Appsをインストールする場合は、インストール後のバリデーションに本エンドポイントを利用すると良いと感じた。具体的には、ユーザーが設定したGitHub Appsのインストール設定のバリデーションをして、違反ならアンインストールのエンドポイントを叩き、4XXコードを返却。連携に失敗しましたページを表示して、設定ミスをユーザーに伝える。

## GitHub Appsアンインストール
主な処理は下記。
* ユーザーテーブルのinstalltionIdとFirebase AuthenticationのUIDを紐づけを削除
* GitHub Appsのアンイストールエンドポイントを叩く(DELETE /app/installations/{installation_id})

# プレビュー環境
ローカルでリアルタイムでプレビュー可能なNeovimのプラグインを作った。マークダウンパーサーとCSSをライブラリ化しているので実装は簡単。
![Neovimプレビュー](https://res.cloudinary.com/dkerzyk09/image/upload/v1615089687/blog/01ezsr2jdx19bg00pgwt1rnsk6/ttpskowmt8t41u79dwbo.gif)

リポジトリは[hozi-dev/preview-hozi-dev.nvim](https://github.com/hozi-dev/preview-hozi-dev.nvim)。本プラグインでもNext.jsをラップしている。

追加本プラグインに下記の機能をつけると、捗りそう。
* Cloudinaryに画像をアップロードする機能
* movをgifに変換する可能
* 自動スクロール

仕組みは過去記事を参考に。
* [NeovimでSwaggerをリアルタイムプレビューするプラグインを作った話](https://qiita.com/hozi894/items/d9b296f923b2d394a26f)

# 料金
最後に2021年2月のAWS費用を記載しておく。環境は開発と本番の2つがあります。

|サービス|料金($)|補足|
|---|---|---|
|DynamoDB|3.21|DBは2つで2環境の合計4つ<br>プロビジョニングされたキャパシティモードでデフォルト値<br>(オンデマンドにしたらもっと安い)
|API Gateway|0.07|
|Route53|0.55|ホストゾーン1つ(hozi.dev)に対して$0.5
|Lambda|0.00|
|合計|3.83|

従量課金のサービスを多く活用しているので、ほぼ誰も見ていない現環境では対してお金はかからない(血涙)。VercelはHobby Planで、その他SaasSは無料枠で収まっているため、実質AWSにかかっている金額がブログの運用費用となる。リクエスト数やLambda実行回数を載せないと大した参考にならない気がするので、別で記事化することにする。あくまで参考程度に。

# 最後に
ざっくりでしたが、以上が本ブログの構成。SaaSを組み合わせることで、比較的に簡単に任意のユーザーがマークダウンで投稿可能なブログが作れる時代なんだなーと感じた。独自の機能やデザインをしたり、SaaSとの連携をする際にやはり既存静的サイトジェネレータ(HugoやJekyllのこと)だと柔軟にできなかったりする。

一方で本ブログのようにサーバサイド、フロントエンドを独自で組むと、認証つきページで様々な検証が可能。サーバレスなので、AWSが死なない限り、気にする必要がないので運用も楽でおすすめです。

以上。追加で聞きたいことがあれば、[こちら](https://github.com/hozi-dev/article/issues)でissueとして起票して頂ければと思います。
