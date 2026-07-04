---
title: "DeepL APIを試す"
type: "tech"
description: "DeepL APIを試します"
publish: true
tags:
  - "tech/deepl"
  - "tech/cli"
---

# はじめに
翻訳ツールDeepLのAPIが、いつの間にか整っていたので、試してみました。今後関連した個人ツールを作る予定があり、利用者はトークン取得が必須なため手順を記事化しました。小ネタです。。

# アカウント作成

メールアドレス、パスワード登録
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1637455364/blog/01fmzy00fzvq736j42pa1jgnm5/tc6s01ey91o60bvijb3h.png)

クレジットカードは必須みたいです
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1637458076/blog/01fmzy00fzvq736j42pa1jgnm5/kirrxiqncbgh9v9xl06f.png)

登録完了しました
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1637459174/blog/01fmzy00fzvq736j42pa1jgnm5/atxr4igrpb4bgz4hgryo.png)

# APIキーの取得

認証キーを取得します
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1637460016/blog/01fmzy00fzvq736j42pa1jgnm5/ig9sgatq2qs9z6wpqh4k.png)

ページ下部までスクロールすると、APIキーが記載されております。
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1637460119/blog/01fmzy00fzvq736j42pa1jgnm5/ijb0qe3khqghzjtdatzm.png)

## API実行

[APIドキュメント](https://www.deepl.com/docs-api)が結構充実しています。**ログインしている場合は、APIドキュメントで表示されるサンプルコードが自分の発行済みトークンに置き換えられているため、コピペで動作します**。

### 英語から日本語

```bash 
curl https://api-free.deepl.com/v2/translate \
        -d auth_key=APIキー \
        -d "text=Hello, world"  \
        -d "target_lang=JA"
```

```json:レスポンス
{"translations":[{"detected_source_language":"EN","text":"ハロー、ワールド"}]}
```
こんにちはと訳さないは、学習の結果..なんですかね（）

### 日本語から英語

```bash
curl https://api-free.deepl.com/v2/translate \
        -d auth_key=APIキー \
        -d "text=こんにちは"  \
        -d "target_lang=EN"
```

```json:レスポンス
{"translations":[{"detected_source_language":"JA","text":"hello"}]}
```

### 利用状況の確認
```bash
curl -H "Authorization: DeepL-Auth-Key APIキー" https://api-free.deepl.com/v2/usage                                                                                                                                      21-11-21 11:13:39
```

```json:レスポンス
{"character_count":41,"character_limit":500000}
```

利用頻度は、Webページからも確認可能です

![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1637461062/blog/01fmzy00fzvq736j42pa1jgnm5/jqid605sj0thxq7ipzwc.png)


# さいごに
様々な場面で、活用の機会がありそうですね。訳も某社と比較して、直訳っぽくないので気に入っています。
