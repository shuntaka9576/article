---
title: "SPAにTwitter OAuth 2.0を組み込む際の雑メモ"
type: "tech"
description: "SPAにTwitter OAuth2.0を組み込む際の雑メモ"
thumbnail: ""
publish: true
tags:
  - "tech/oauth"
  - "tech/cloudflare"
  - "tech/セキュリティ"
  - "tech/typescript"
---

# はじめに
Twitter OAuth 2.0の認可の仕組みを利用して、Webアプリを作る場合、どういうアーキテクチャがいいか検討してみました。表題の通り雑です。またセキュリティ的にも裏が取れているわけではないので、参考程度にお願いします。
# 構成


## トークンの扱い検討
方法はざっくり以下の3つ(雑すぎ)。3がいいかなぁと思いシーケンスを書いてみる

1. Authraizationヘッダー + localStorage
  => XSS怖い
2. Authraizationヘッダー + Cookie(not HTTP Only属性)
  => CSRF怖いのでAuthraizationヘッダーを使うと、CookieにHTTP only属性は使えない。。(XSS怖い)
3. Cookie認証 + Cookie(HTTP Only属性+Same site(Lax,Strict))
  => XSS,CSRF対策出来てそう

## シーケンス

補足事項
* Twitter OAuth 2.0は、`OAuth 2.0 Authorization Code Flow with PKCE`というフローを採用している
* Cloudflare Functionsを使って、クラサバを同一ドメインにしている
* SetCookieは、SameSite=Lax;Secure;HttpOnly;を付与

### (1) User-Agentとサーバーサイド間をセッション管理しつつ、サーバサイドでaccessToken,refreshTokenを保持する

:::message warn
編集中
:::

@startuml
actor "User" as user
box "Cloudflare Pages" #dee6ee
  participant "トップページ" as clPages
end box
box "CDN Edge Server" #e6e6fa
  participant "Cloudflare Functions" as clFunctions
  database "User" as dbUser
  database "Session" as dbSession
end box
box "Twitter" #2bc4ff
  participant "Twitter認可画面" as twAuth
  participant "Twitter認可サーバー" as twServer
  participant "TwitterAPI" as twApi
end box
activate clPages
activate user
user->clPages: login with Twitterを押下
clPages->clPages: code_verifier生成
clPages->clPages: code_verifierをcookieに保存(※1)
clPages->clPages: code_verifierからcode_challenge生成
clPages->twAuth: code_challenge
activate twAuth
twAuth->twAuth: Present authorization dialog
twAuth-->user
deactivate twAuth
user->user: Authraize or declines app(※2)
user->twServer
activate twServer
twServer-->twServer: Redirects to your app's callback URL
twServer-->clPages: authraization code
deactivate twServer
clPages->clPages: cookieからcode_verifierを取り出し
clPages->clFunctions: code, code_verifier
activate clFunctions
clFunctions->twServer: code, code_verifier, clientSecret(※3)
activate twServer
twServer-->clFunctions: accessToken
deactivate twServer
clFunctions->twApi: 認可情報の取得
activate twApi
twApi-->clFunctions: 取得OK
deactivate twApi
clFunctions->clFunctions: 認可情報からidを取得
clFunctions->clFunctions: userIdのジェネレート
clFunctions->dbUser: idで検索
activate dbUser
alt 存在しない場合
  dbUser-->clFunctions: 取得NG
  clFunctions->dbUser: id,userId,accessToken,refreshTokenを登録
  dbUser-->clFunctions: 登録OK
end
dbUser-->clFunctions: 取得OK
deactivate dbUser
clFunctions->clFunctions: SessionIdを生成
clFunctions->dbSession: SessionIdとidを保存
activate dbSession
dbSession-->clFunctions: 保存OK
deactivate dbSession
clFunctions-->clPages: SessionId(SetCookie: SameSite=Lax;Secure;HttpOnly;)
deactivate clFunctions
clPages-->user: ログインSuccess
deactivate clPages
deactivate user
@enduml


### (2) CookieにaccessToken,refreshTokenを保持する場合(非推奨)

@startuml
actor "User" as user
box "Cloudflare Pages" #dee6ee
  participant "トップページ" as clPages
end box
box "CDN Edge Server" #e6e6fa
  participant "Cloudflare Functions" as clFunctions
  database "Cloudflare KV" as clKv
end box
box "Twitter" #2bc4ff
  participant "Twitter認可画面" as twAuth
  participant "Twitter認可サーバー" as twServer
  participant "TwitterAPI" as twApi
end box
activate clPages
activate user
user->clPages: login with Twitterを押下
clPages->clPages: code_verifier生成
clPages->clPages: code_verifierをcookieに保存(※1)
clPages->clPages: code_verifierからcode_challenge生成
clPages->twAuth: code_challenge
activate twAuth
twAuth->twAuth: Present authorization dialog
twAuth-->user
deactivate twAuth
user->user: Authraize or declines app(※2)
user->twServer
activate twServer
twServer-->twServer: Redirects to your app's callback URL
twServer-->clPages: authraization code
deactivate twServer
clPages->clPages: cookieからcode_verifierを取り出し
clPages->clFunctions: code, code_verifier
activate clFunctions
clFunctions->twServer: code, code_verifier, clientSecret(※3)
activate twServer
twServer-->clFunctions: accessToken
deactivate twServer
clFunctions-->clPages: set cookie(SameSite=Lax;Secure;HttpOnly;)
deactivate clFunctions
clPages->clFunctions: 認可情報取得(/api/me)
activate clFunctions
clFunctions->twApi: 認可情報取得(/users/me) (※4)
activate twApi
twApi-->clFunctions: 取得OK
deactivate twApi
alt 認可失敗した場合
  alt refreshトークンがヘッダーにある場合
    clFunctions->twAuth: tokenのリフレッシュ
    activate twAuth
    twAuth-->clFunctions: リフレッシュOK
    deactivate twAuth
    clFunctions->twApi: 認可情報取得
    activate twApi
    twApi-->clFunctions: 取得OK
    deactivate twApi
    ref over clFunctions, clKv
      ユーザーデータ取得処理参照
    end
    clFunctions-->clPages: 認可OK, set cookie(token,refresh_token)
  else それ以外
    clFunctions-->clPages: 401 Unauthorized
  end
end
group ユーザーデータ取得処理
  clFunctions->clFunctions: idを取り出す
  clFunctions->clKv: idに紐づくユーザーデータを取得
  activate clKv
  clKv-->clFunctions: 取得OK
  deactivate clKv
end
clFunctions-->clPages: 認可OK
deactivate clFunctions
clPages-->user: ログインSuccess
deactivate clPages
deactivate user
@enduml


※1
もやもや箇所。Twitterに遷移してしまうため、永続領域に書き込まないといけない。code_verifierの格納は、secure属性とSameSite=Laxを付与。HttpOnly属性はJSからなので付与できない。プラスで適切なexpire時間を設定する。
```bash
document.cookie="code_verifier=hoge;secure;SameSite=Lax"
```

※2
scopeは以下の通り
* /2/users/meエンドポイントが実行できる ... `users.read` `tweet.read`
* refresh tokeの取得 ... `offline.access`

※3
Twitterには、Public ClientとConfidetial Clientが指定できます。サーバーサイドで安全にClientSecretを運用できる今回のケースでは、Confidetial Clientでいいかなと思いました。Public Clientの場合、`Authorization: Basic`ヘッダーを利用せず、ClientSecretも利用しません。Confidetial Clientを指定すると、Public Clientのトークン取得方法が利用できませんが、逆は可能みたいです。

※4
[Rate limits](https://developer.twitter.com/en/docs/twitter-api/rate-limits)と[Users lookup](https://developer.twitter.com/en/docs/twitter-api/users/lookup/api-reference/get-users-me)を参照。アプリレート制限とユーザーレート制限があり、別で計算される

* ユーザーレート制限
  * 認証されたユーザー毎に15分間に75リクエスト
* アプリレート制限
  * アプリのOAuth 2.0ベアラートークンは、15分間に900リクエスト


# 留意点

## アクセストークンのライフサイクル

|項目|値|
|---|---|
|アクセストークンの期限|2(h)
|リフレッシュトークンの期限|不明

## トークンの無効化
[Authentication](https://developer.twitter.com/en/docs/authentication/oauth-2-0/user-access-token)より、`/2/oauth2/revoke`エンドポイントが用意されている。`token_type_hint`でアクセストークンもしくはリフレッシュトークンを利用して、トークンをrevokeできる。revokeされるのは両方のトークン。

```bash
$ curl --location --request POST 'https://api.twitter.com/2/oauth2/revoke' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--basic -u '<ClientId>:<ClientSecret>' \
--data-urlencode 'token_type_hint=refresh_token' \
# --data-urlencode 'token_type_hint=access_token' \
--data-urlencode 'token=xxx'
```


# さいごに
実際にこの構成でアプリを作ってみるので、更新情報があればまた書こうと思います。仕事で公開されているPOSTのAPIのHTTPレスポンスヘッダー(`Access-Control-Allow-Origin`)に呼び出し元を指定して、CSRF対策したのを思い出した。

# 参考

* [OAuth 2.0 Making requests on behalf of users](https://developer.twitter.com/en/docs/authentication/oauth-2-0/authorization-code)
* [stackoverflow| Best practice on Securing code_verifier in PKCE-enhanced Authorization Code Flow](https://stackoverflow.com/questions/67517436/best-practice-on-securing-code-verifier-in-pkce-enhanced-authorization-code-flow)


