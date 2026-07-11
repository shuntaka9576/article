---
title: "OpenSSLでJWTの作成、検証ロジックを確認する"
description: "OpenSSLを使ってJWTの作成、検証を自前でやってみます"
publish: true
tags:
  - "tech/セキュリティ"
  - "tech/jwt"
  - "tech/oauth"
---

# はじめに

本ブログは、自分用途に下書きをプレビューしたり、公開するまでもないメモを認証を通して閲覧する機能があります。認証には、Firebase AuthenticationとJWTで、AWS API GatewayとLambdaオーソライザで認証しています。詳しくは[こちら](https://shuntaka.dev/shuntaka/articles/01f07hctzhjcwtdq4h6ew9stk8#%E6%A7%8B%E6%88%90%E6%A6%82%E8%A6%81)

Firebase Authenticationのライブラリは、非常に優秀でJWTの検証はライブラリがいい感じにやってくれます。Auth0などの他のIDaaSも一緒だと思います。
ただ、JWTの概念、作成・検証ロジックを知っておくと様々な認証設計時に有効です。故に記事にすることにしました。

記事の構成は大きく分けて2つです。

* [Firebase AutheticationのJWTを検証する](#firebase-autheticationのjwtを検証する)
Firebase Authenticationで発行されたJWTをOpenSSLを使って検証します。基本的に[JWTの署名をOpenSSLコマンドで検証する](https://qiita.com/yyu/items/e1023619249f30911496)をなぞっています。私がうまく行かなかった点を補足しており、元記事を読んだ上で不明点があったら読むと良いと思います。

* [自分でJWTを作って検証する](#自分でjwtを作って検証する)
OpenSSLを使ってJWTを作り検証します。

# 環境設定
## base64
base64コマンドはbrewではなく、デフォルトのものを利用しています。(`-D`でデコード出来る方です)

## OpenSSLの確認
brewでインストールしたバイナリを利用します
```bash
$ brew install openssl
$ brew --prefix openssl
/usr/local/opt/openssl@1.1
$ export PATH="/usr/local/opt/openssl@1.1/bin:$PATH"
$ openssl version
OpenSSL 1.1.1k  25 Mar 2021
```

# Firebase AutheticationのJWTを検証する
本項で書くことは下記
* IDaaSを使う側のライブラリに隠蔽されているJWT検証方法

逆に書かないことは下記
* IDaaS側の処理。JWTを作る側の話。JWT作る話は、[自分でJWTを作って検証する](#自分でjwtを作って検証する)を参考に

## Firebaseの証明書(in 公開鍵)を取得
FirebaseのX.509証明書を取得します。中に公開鍵が入っています。

```bash
$ wget -O firebase_public_key_list.json \
https://www.googleapis.com/robot/v1/metadata/x509/securetoken@system.gserviceaccount.com

$ cat firebase_public_key_list.json
{
  "4e9df5a4f28ad025064d66553bcb9b339685ef94": "-----BEGIN CERTIFICATE-----\nMIIDHDCCAgSgAwIBAgIIc4en75abvtowDQYJKoZIhvcNAQEFBQAwMTEvMC0GA1UE\nAxMmc2VjdXJldG9rZW4uc3lzdGVtLmdzZXJ2aWNlYWNjb3VudC5jb20wHhcNMjEw\nNDIyMDkyMDIwWhcNMjEwNTA4MjEzNTIwWjAxMS8wLQYDVQQDEyZzZWN1cmV0b2tl\nbi5zeXN0ZW0uZ3NlcnZpY2VhY2NvdW50LmNvbTCCASIwDQYJKoZIhvcNAQEBBQAD\nggEPADCCAQoCggEBALYEX6eorAQ5bDhcU0Ts/v9SkqC9mFr/B5dMs9d4cp9cbf5o\nmOk1XXpnKBER9Fqq0he5bazFI9/JLAg++hxXo1th/qHKgWxPhSe2NGIFEjAgKN+b\naDKjm3I/w3d3AEW3JthbomujXnycsj+UxdDkGFX0BZdK7iddalYocqnnRETlyg34\n36iGRllRY+d99G1XM45Ow9pJW9z2pttzOBeDS0wuQSx2M6lABRj7BDbs7iyJfRqO\nJFw9qbKXitGx6wYy3aoKx4U5JWAKOrGwE55HPAg5ufm0bLW0qcn6pvF/BJqk7YoZ\nJNxNOeVAOSe0jKMlihTfyyYVLQ1E1Pz/lVrzg78CAwEAAaM4MDYwDAYDVR0TAQH/\nBAIwADAOBgNVHQ8BAf8EBAMCB4AwFgYDVR0lAQH/BAwwCgYIKwYBBQUHAwIwDQYJ\nKoZIhvcNAQEFBQADggEBAE1LGRA9vJrd3E/SLPYPZixaEpOiMzLUhFZbP4H9/oZu\n94iHdI3sd8RKU5i9fkufAvQuYz8zDTWJE+/XIYGv5QZ/v7Y+dWMKiH9P4iQ1hY/I\num8rp+Vtsp0LSvjACDzpiVfn4ljH71xbw5kd51BtPJyD/QEqKJE0XU6YQEriHZH1\no/t9bfk82r3cxKakpM+8aB2fHPxbKTBRA6eBbfeqkyj4Depx3bMIB/EBA2YrHs2n\nwxrUUw/9OCdDm0J4tk0Oq+mo2cBBJK/qvjTLy6fW0NWhLc9w4zG7QiQyCtXaaWSh\nfq1RLV+QmAvCy3qSsrkKQMN+vvIhufYE+uVlMc9D3T0=\n-----END CERTIFICATE-----\n",
  "cc3f4e8b2f1d02f0ea4b1bdde55add8b08bc5386": "-----BEGIN CERTIFICATE-----\nMIIDHDCCAgSgAwIBAgIIBYBvKEWCey0wDQYJKoZIhvcNAQEFBQAwMTEvMC0GA1UE\nAxMmc2VjdXJldG9rZW4uc3lzdGVtLmdzZXJ2aWNlYWNjb3VudC5jb20wHhcNMjEw\nNDMwMDkyMDIwWhcNMjEwNTE2MjEzNTIwWjAxMS8wLQYDVQQDEyZzZWN1cmV0b2tl\nbi5zeXN0ZW0uZ3NlcnZpY2VhY2NvdW50LmNvbTCCASIwDQYJKoZIhvcNAQEBBQAD\nggEPADCCAQoCggEBAKVS+ThsK2Sm2nX00q++5CghCrJTLETbnJUbUTRzZ7vTsUsJ\nD1kL3eaYE0FdDKylBkbV8CWcR1PH3PIU7VTZ6YhGsJhVbO+mtVjPpTHeRWTa1cSb\nigtPw+NsjosaeoAXPkjvu+YZQW97Hes+YHjAWL++ZiQV0QJUy2/f51Da4V6v2dEW\nSRNCUhkhaNqYTwgw5qKwU2iox4u+UkMK42iIpumlVtJBXc01QXFHdnjhPrQpseGb\npZrAzcqUcZWzv4ZioZW3dT4KT/AYgZQD5iU1TfoJuhug/m6D86XOo3cqoLFgu8Gs\nutAM3svytrU10TxagoEuE0SLrMZ2pefwQ6+EF+0CAwEAAaM4MDYwDAYDVR0TAQH/\nBAIwADAOBgNVHQ8BAf8EBAMCB4AwFgYDVR0lAQH/BAwwCgYIKwYBBQUHAwIwDQYJ\nKoZIhvcNAQEFBQADggEBAJma2XXlSFwj1++Eue1m0KCCI3VIFidvuZ0kbxqB6JUI\nI9W2rpqvmssGbMPJVHBadDOO2Z5FrY4S863240k3dWxhU2zt0XVfaiGl7WH1o5Am\nrAvc4IQ2B1n/6FpUGuIXQfAokGWusj6zcAlJ/jiHw9ehxcVt0QKxWnxSsWuNxrUs\nZFTPgqL52yo5f0NFMq8yRrMDpMJ3FTzTdscK0vBfbG/P79+CSG1pNcq3AZUfuYyz\nHZXKqzokuyAA7T5ts/9XdwSH1KRdKBO22k2X/69R+YS5l307Mjgh+vjWDN1WWnI1\nJ4o3Tct9WsgQzFNp3/p4A1CFd2fCIjq/INPnpMSLg94=\n-----END CERTIFICATE-----\n"
}
```

後述するが今回サンプル使うJWTのHeaderの`kid`が`cc3f4e8b2f1d02f0ea4b1bdde55add8b08bc5386`なので、`firebase_public_key_list.json`の値に入っている公開鍵を`firebase_public_key.pem`に保存します。

```bash
cat firebase_public_key_list.json | \
  jq -r ".\"cc3f4e8b2f1d02f0ea4b1bdde55add8b08bc5386\"" > firebase_public_key.pem
```

前述のとおり公開鍵が含まれており、`Public-Key: (2048 bit)`より鍵長が2048bitであることがわかる
```bash
$ openssl x509 -text -noout -in firebase_public_key.pem
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 396438985964157741 (0x5806f2845827b2d)
    Signature Algorithm: sha1WithRSAEncryption
        Issuer: CN=securetoken.system.gserviceaccount.com
        Validity
            Not Before: Apr 30 09:20:20 2021 GMT
            Not After : May 16 21:35:20 2021 GMT
        Subject: CN=securetoken.system.gserviceaccount.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:a5:52:f9:38:6c:2b:64:a6:da:75:f4:d2:af:be:
                    e4:28:21:0a:b2:53:2c:44:db:9c:95:1b:51:34:73:
                    67:bb:d3:b1:4b:09:0f:59:0b:dd:e6:98:13:41:5d:
                    0c:ac:a5:06:46:d5:f0:25:9c:47:53:c7:dc:f2:14:
                    ed:54:d9:e9:88:46:b0:98:55:6c:ef:a6:b5:58:cf:
                    a5:31:de:45:64:da:d5:c4:9b:8a:0b:4f:c3:e3:6c:
                    8e:8b:1a:7a:80:17:3e:48:ef:bb:e6:19:41:6f:7b:
                    1d:eb:3e:60:78:c0:58:bf:be:66:24:15:d1:02:54:
                    cb:6f:df:e7:50:da:e1:5e:af:d9:d1:16:49:13:42:
                    52:19:21:68:da:98:4f:08:30:e6:a2:b0:53:68:a8:
                    c7:8b:be:52:43:0a:e3:68:88:a6:e9:a5:56:d2:41:
                    5d:cd:35:41:71:47:76:78:e1:3e:b4:29:b1:e1:9b:
                    a5:9a:c0:cd:ca:94:71:95:b3:bf:86:62:a1:95:b7:
                    75:3e:0a:4f:f0:18:81:94:03:e6:25:35:4d:fa:09:
                    ba:1b:a0:fe:6e:83:f3:a5:ce:a3:77:2a:a0:b1:60:
                    bb:c1:ac:ba:d0:0c:de:cb:f2:b6:b5:35:d1:3c:5a:
                    82:81:2e:13:44:8b:ac:c6:76:a5:e7:f0:43:af:84:
                    17:ed
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Key Usage: critical
                Digital Signature
            X509v3 Extended Key Usage: critical
                TLS Web Client Authentication
    Signature Algorithm: sha1WithRSAEncryption
         99:9a:d9:75:e5:48:5c:23:d7:ef:84:b9:ed:66:d0:a0:82:23:
         75:48:16:27:6f:b9:9d:24:6f:1a:81:e8:95:08:23:d5:b6:ae:
         9a:af:9a:cb:06:6c:c3:c9:54:70:5a:74:33:8e:d9:9e:45:ad:
         8e:12:f3:ad:f6:e3:49:37:75:6c:61:53:6c:ed:d1:75:5f:6a:
         21:a5:ed:61:f5:a3:90:26:ac:0b:dc:e0:84:36:07:59:ff:e8:
         5a:54:1a:e2:17:41:f0:28:90:65:ae:b2:3e:b3:70:09:49:fe:
         38:87:c3:d7:a1:c5:c5:6d:d1:02:b1:5a:7c:52:b1:6b:8d:c6:
         b5:2c:64:54:cf:82:a2:f9:db:2a:39:7f:43:45:32:af:32:46:
         b3:03:a4:c2:77:15:3c:d3:76:c7:0a:d2:f0:5f:6c:6f:cf:ef:
         df:82:48:6d:69:35:ca:b7:01:95:1f:b9:8c:b3:1d:95:ca:ab:
         3a:24:bb:20:00:ed:3e:6d:b3:ff:57:77:04:87:d4:a4:5d:28:
         13:b6:da:4d:97:ff:af:51:f9:84:b9:97:7d:3b:32:38:21:fa:
         f8:d6:0c:dd:56:5a:72:35:27:8a:37:4d:cb:7d:5a:c8:10:cc:
         53:69:df:fa:78:03:50:85:77:67:c2:22:3a:bf:20:d3:e7:a4:
         c4:8b:83:de
```

## JWTの構造
```bash
eyJhbGciOiJSUzI1NiIsImtpZCI6ImNjM2Y0ZThiMmYxZDAyZjBlYTRiMWJkZGU1NWFkZDhiMDhiYzUzODYiLCJ0eXAiOiJKV1QifQ.eyJpc3MiOiJodHRwczovL3NlY3VyZXRva2VuLmdvb2dsZS5jb20vaG96aWRldiIsImF1ZCI6ImhvemlkZXYiLCJhdXRoX3RpbWUiOjE2MjAyMDYyMjQsInVzZXJfaWQiOiJxd213Z01tRE1ZWkZLODNrNVZYWUVUTDU3VkMyIiwic3ViIjoicXdtd2dNbURNWVpGSzgzazVWWFlFVEw1N1ZDMiIsImlhdCI6MTYyMDIwNjIyNCwiZXhwIjoxNjIwMjA5ODI0LCJlbWFpbCI6InRlc3RAZXhhbXBsZS5jb20iLCJlbWFpbF92ZXJpZmllZCI6ZmFsc2UsImZpcmViYXNlIjp7ImlkZW50aXRpZXMiOnsiZW1haWwiOlsidGVzdEBleGFtcGxlLmNvbSJdfSwic2lnbl9pbl9wcm92aWRlciI6InBhc3N3b3JkIn19.fegApHLra7RU8IGTYXBvdHhKmdNe7ZUHyAmuRBifAgcL5IyPxBp3NqC7x39bPEGLfOJL2W-GyHBT8mVHD7HF7SzhYH7unLlbRsdGBAKeQZPjH9FFYo_lXwmPLjLG5gjpf46yxVzZ1w28Ct2JS1HGvTrCUAEM_o23HZryK60uFXx0bTYzDtKeXN4ASwenTrX874KMCHqmZBnWGKB6htW49uCMg-upxnqSHq407nP7dZnDegTGzd5I7HkpsgMS8Kwf8K9UX5ebadaYfUeY2tWYEV04GNJ6X_xq8ougI4n9meINP2y8fyz9jfKciB6J0QNQglvkqvp5npQQoySWpZ1O2Q
```

JWTはHeader,Payload,Signatureが`.`で連結されたトークン形式です。

## Header
```bash
eyJhbGciOiJSUzI1NiIsImtpZCI6ImNjM2Y0ZThiMmYxZDAyZjBlYTRiMWJkZGU1NWFkZDhiMDhiYzUzODYiLCJ0eXAiOiJKV1QifQ
```

`base64`でデコードします。

**注意点: base64でエンコードされた文字長は、4の倍数です。今回は102(4 * 25 + 2)文字で、2文字分パディング文字`=`を付与しないとbase64がデコードしてくれません。**
```bash
echo -n 'eyJhbGciOiJSUzI1NiIsImtpZCI6ImNjM2Y0ZThiMmYxZDAyZjBlYTRiMWJkZGU1NWFkZDhiMDhiYzUzODYiLCJ0eXAiOiJKV1QifQ==' | \
base64 -D
```

```bash
{
  "alg": "RS256",
  "kid": "cc3f4e8b2f1d02f0ea4b1bdde55add8b08bc5386",
  "typ": "JWT"
}
```

## Payload
```bash
eyJpc3MiOiJodHRwczovL3NlY3VyZXRva2VuLmdvb2dsZS5jb20vaG96aWRldiIsImF1ZCI6ImhvemlkZXYiLCJhdXRoX3RpbWUiOjE2MjAyMDYyMjQsInVzZXJfaWQiOiJxd213Z01tRE1ZWkZLODNrNVZYWUVUTDU3VkMyIiwic3ViIjoicXdtd2dNbURNWVpGSzgzazVWWFlFVEw1N1ZDMiIsImlhdCI6MTYyMDIwNjIyNCwiZXhwIjoxNjIwMjA5ODI0LCJlbWFpbCI6InRlc3RAZXhhbXBsZS5jb20iLCJlbWFpbF92ZXJpZmllZCI6ZmFsc2UsImZpcmViYXNlIjp7ImlkZW50aXRpZXMiOnsiZW1haWwiOlsidGVzdEBleGFtcGxlLmNvbSJdfSwic2lnbl9pbl9wcm92aWRlciI6InBhc3N3b3JkIn19
```

`base64`でデコードします。448で4の剰余が0なので、`=`を追加する必要はありません。
```bash
echo -n 'eyJpc3MiOiJodHRwczovL3NlY3VyZXRva2VuLmdvb2dsZS5jb20vaG96aWRldiIsImF1ZCI6ImhvemlkZXYiLCJhdXRoX3RpbWUiOjE2MjAyMDYyMjQsInVzZXJfaWQiOiJxd213Z01tRE1ZWkZLODNrNVZYWUVUTDU3VkMyIiwic3ViIjoicXdtd2dNbURNWVpGSzgzazVWWFlFVEw1N1ZDMiIsImlhdCI6MTYyMDIwNjIyNCwiZXhwIjoxNjIwMjA5ODI0LCJlbWFpbCI6InRlc3RAZXhhbXBsZS5jb20iLCJlbWFpbF92ZXJpZmllZCI6ZmFsc2UsImZpcmViYXNlIjp7ImlkZW50aXRpZXMiOnsiZW1haWwiOlsidGVzdEBleGFtcGxlLmNvbSJdfSwic2lnbl9pbl9wcm92aWRlciI6InBhc3N3b3JkIn19' | \
base64 -D | \
jq
```

```bash
{
  "iss": "https://securetoken.google.com/hozidev",
  "aud": "hozidev",
  "auth_time": 1620206224,
  "user_id": "qwmwgMmDMYZFK83k5VXYETL57VC2",
  "sub": "qwmwgMmDMYZFK83k5VXYETL57VC2",
  "iat": 1620206224,
  "exp": 1620209824,
  "email": "test@example.com",
  "email_verified": false,
  "firebase": {
    "identities": {
      "email": [
        "test@example.com"
      ]
    },
    "sign_in_provider": "password"
  }
}
```

## Signature

```bash
fegApHLra7RU8IGTYXBvdHhKmdNe7ZUHyAmuRBifAgcL5IyPxBp3NqC7x39bPEGLfOJL2W-GyHBT8mVHD7HF7SzhYH7unLlbRsdGBAKeQZPjH9FFYo_lXwmPLjLG5gjpf46yxVzZ1w28Ct2JS1HGvTrCUAEM_o23HZryK60uFXx0bTYzDtKeXN4ASwenTrX874KMCHqmZBnWGKB6htW49uCMg-upxnqSHq407nP7dZnDegTGzd5I7HkpsgMS8Kwf8K9UX5ebadaYfUeY2tWYEV04GNJ6X_xq8ougI4n9meINP2y8fyz9jfKciB6J0QNQglvkqvp5npQQoySWpZ1O2Q
```

バイナリ形式のため、HeaderやPayloadのように文字列は表示されません。hexdumpを使います。(4の剰余が2のなので末尾に`==`を付与)
```bash
$ echo -n "fegApHLra7RU8IGTYXBvdHhKmdNe7ZUHyAmuRBifAgcL5IyPxBp3NqC7x39bPEGLfOJL2W-GyHBT8mVHD7HF7SzhYH7unLlbRsdGBAKeQZPjH9FFYo_lXwmPLjLG5gjpf46yxVzZ1w28Ct2JS1HGvTrCUAEM_o23HZryK60uFXx0bTYzDtKeXN4ASwenTrX874KMCHqmZBnWGKB6htW49uCMg-upxnqSHq407nP7dZnDegTGzd5I7HkpsgMS8Kwf8K9UX5ebadaYfUeY2tWYEV04GNJ6X_xq8ougI4n9meINP2y8fyz9jfKciB6J0QNQglvkqvp5npQQoySWpZ1O2Q==" | \
base64 -D | \
hexdump -C

00000000  7d e8 00 a4 72 eb 6b b4  54 f0 81 93 61 70 6f 74  |}...r.k.T...apot|
00000010  78 4a 99 d3 5e ed 95 07  c8 09 ae 44 18 9f 02 07  |xJ..^......D....|
00000020  0b e4 8c 8f c4 1a 77 36  a0 bb c7 7f 5b 3c 41 8b  |......w6....[<A.|
00000030  7c e2 4b d9 6f 86 c8 70  53 f2 65 47 0f b1 c5 ed  ||.K.o..pS.eG....|
00000040  2c e1 60 7e ee 9c b9 5b  46 c7 46 04 02 9e 41 93  |,.`~...[F.F...A.|
00000050  e3 1f d1 45 62 8f e5 5f  09 8f 2e 32 c6 e6 08 e9  |...Eb.._...2....|
00000060  7f 8e b2 c5 5c d9 d7 0d  bc 0a dd 89 4b 51 c6 bd  |....\.......KQ..|
00000070  3a c2 50 01 0c fe 8d b7  1d 9a f2 2b ad 2e 15 7c  |:.P........+...||
00000080  74 6d 36 33 0e d2 9e 5c  de 00 4b 07 a7 4e b5 fc  |tm63...\..K..N..|
00000090  ef 82 8c 08 7a a6 64 19  d6 18 a0 7a 86 d5 b8 f6  |....z.d....z....|
000000a0  e0 8c 83 eb a9 c6 7a 92  1e ae 34 ee 73 fb 75 99  |......z...4.s.u.|
000000b0  c3 7a 04 c6 cd de 48 ec  79 29 b2 03 12 f0 ac 1f  |.z....H.y)......|
000000c0  f0 af 54 5f 97 9b 69 d6  98 7d 47 98 da d5 98 11  |..T_..i..}G.....|
000000d0  5d 38 18 d2 7a 5f fc 6a  f2 8b a0 23 89 fd 99 e2  |]8..z_.j...#....|
000000e0  0d 3f 6c bc 7f 2c fd 8d  f2 9c 88 1e 89 d1 03 50  |.?l..,.........P|
000000f0  82 5b e4 aa fa 79 9e 94  10 a3 24 96 a5 9d 4e d9  |.[...y....$...N.|
00000100
```
16 * 16(byte) * 8(bit) = 2048(bit)

Signatureのデコード結果を`jwt.sign`に保存(4の剰余が2なので、末尾に`==`を付与)
```bash
$ echo -n "fegApHLra7RU8IGTYXBvdHhKmdNe7ZUHyAmuRBifAgcL5IyPxBp3NqC7x39bPEGLfOJL2W-GyHBT8mVHD7HF7SzhYH7unLlbRsdGBAKeQZPjH9FFYo_lXwmPLjLG5gjpf46yxVzZ1w28Ct2JS1HGvTrCUAEM_o23HZryK60uFXx0bTYzDtKeXN4ASwenTrX874KMCHqmZBnWGKB6htW49uCMg-upxnqSHq407nP7dZnDegTGzd5I7HkpsgMS8Kwf8K9UX5ebadaYfUeY2tWYEV04GNJ6X_xq8ougI4n9meINP2y8fyz9jfKciB6J0QNQglvkqvp5npQQoySWpZ1O2Q==" | \
base64 -D > jwt.sign
```


## JWTの検証
公開鍵を利用して、JWTの署名を検証する
```bash
$ openssl rsautl -verify -asn1parse -in jwt.sign -certin -inkey firebase_public_key.pem
    0:d=0  hl=2 l=  49 cons: SEQUENCE
    2:d=1  hl=2 l=  13 cons:  SEQUENCE
    4:d=2  hl=2 l=   9 prim:   OBJECT            :sha256
   15:d=2  hl=2 l=   0 prim:   NULL
   17:d=1  hl=2 l=  32 prim:  OCTET STRING
      0000 - 5e 75 be 36 89 30 45 9b-ed 4f 51 93 40 9c 27 94   ^u.6.0E..OQ.@.'.
      0010 - a6 15 1d 98 a6 6a 71 7b-7d a5 4f 45 50 37 cb 21   .....jq{}.OEP7.!
```

上記の下から2,3行目の値と、JWTのHeaderとPayloadをsha256でハッシュ化した結果と一致する
```bash
$ echo -n 'eyJhbGciOiJSUzI1NiIsImtpZCI6ImNjM2Y0ZThiMmYxZDAyZjBlYTRiMWJkZGU1NWFkZDhiMDhiYzUzODYiLCJ0eXAiOiJKV1QifQ.eyJpc3MiOiJodHRwczovL3NlY3VyZXRva2VuLmdvb2dsZS5jb20vaG96aWRldiIsImF1ZCI6ImhvemlkZXYiLCJhdXRoX3RpbWUiOjE2MjAyMDYyMjQsInVzZXJfaWQiOiJxd213Z01tRE1ZWkZLODNrNVZYWUVUTDU3VkMyIiwic3ViIjoicXdtd2dNbURNWVpGSzgzazVWWFlFVEw1N1ZDMiIsImlhdCI6MTYyMDIwNjIyNCwiZXhwIjoxNjIwMjA5ODI0LCJlbWFpbCI6InRlc3RAZXhhbXBsZS5jb20iLCJlbWFpbF92ZXJpZmllZCI6ZmFsc2UsImZpcmViYXNlIjp7ImlkZW50aXRpZXMiOnsiZW1haWwiOlsidGVzdEBleGFtcGxlLmNvbSJdfSwic2lnbl9pbl9wcm92aWRlciI6InBhc3N3b3JkIn19' | \
shasum -a 256

5e75be368930459bed4f5193409c2794a6151d98a66a717b7da54f455037cb21  -
```

逆説的だが、JWTの作成、検証の流れは下記のようになる

1. JWTの作成
1-1. Header,Payloadを作る
1-2. `Headerのbase64暗号化文字列.Payloadのbase64暗号化文字列`を、sha256でハッシュ化し、秘密鍵で署名(バイナリ)
1-3. `Headerのbase64暗号化文字列.Payloadのbase64暗号化文字列.項1-2のbase64暗号化文字列`でJWTトークンが完成

2. JWTの検証
2-1. JWTトークンを受け取り、項1-3の3節目を取り出す
2-2. 項2-1.をbase64でデコード(項1-2と同じ)
2-3. 項2-2.を公開鍵で署名を取得
2-4. 項1-3.の`Headerのbase64暗号化文字列.Payloadのbase64暗号化文字列`をsha256ハッシュをとる
2-5. 項目2-3.と項目2-4.が一致することを確認

# 自分でJWTを作って検証する
## OpenSSL確認
```bash
$ export PATH="/usr/local/opt/openssl@1.1/bin:$PATH";openssl version

OpenSSL 1.1.1k  25 Mar 2021
```

## RSA秘密鍵を作成

```bash
$ openssl genrsa -out private.key 2048
```

## 秘密鍵からX.509証明書(in 公開鍵)を作成

```bash
$ openssl req -x509 -new -nodes \
    -key private.key \
    -sha256 -days 1024 \
    -subj "/C=JP/ST=Tokyo/O=shuntaka corp./CN=shuntaka root 2021" \
    -out rootCA.pem
```

## HeaderとPayloadをbase64にする

### Headerを作成
kidは適当です。
```json
{
  "alg": "RS256",
  "kid": "tekitou",
  "typ": "JWT"
}
```

```bash
$ echo -n '{"alg":"RS256","kid":"tekitou","typ":"JWT"}' | \
base64

eyJhbGciOiJSUzI1NiIsImtpZCI6InRla2l0b3UiLCJ0eXAiOiJKV1QifQ==
```

firebaseでは、`=`を削除していたので合わせます。
```bash: JWTのHeader
eyJhbGciOiJSUzI1NiIsImtpZCI6InRla2l0b3UiLCJ0eXAiOiJKV1QifQ
```

### Payloadを作成
```json
{
  "iss": "https://hoge",
  "aud": "hozidev",
  "auth_time": 1620206224,
  "user_id": "foo",
  "sub": "foohoge",
  "iat": 1620206224,
  "exp": 1620209824,
  "email": "hoge@example.com",
  "email_verified": false,
  "firebase": {
    "identities": {
      "email": [
        "hoge@example.com"
      ]
    },
    "sign_in_provider": "password"
  }
}
```

```bash
$ echo -n '{"iss":"https://hoge","aud":"hozidev","auth_time":1620206224,"user_id":"foo","sub":"foohoge","iat":1620206224,"exp":1620209824,"email":"hoge@example.com","email_verified":false,"firebase":{"identities":{"email":["hoge@example.com"]},"sign_in_provider":"password"}}' |\
base64

eyJpc3MiOiJodHRwczovL2hvZ2UiLCJhdWQiOiJob3ppZGV2IiwiYXV0aF90aW1lIjoxNjIwMjA2MjI0LCJ1c2VyX2lkIjoiZm9vIiwic3ViIjoiZm9vaG9nZSIsImlhdCI6MTYyMDIwNjIyNCwiZXhwIjoxNjIwMjA5ODI0LCJlbWFpbCI6ImhvZ2VAZXhhbXBsZS5jb20iLCJlbWFpbF92ZXJpZmllZCI6ZmFsc2UsImZpcmViYXNlIjp7ImlkZW50aXRpZXMiOnsiZW1haWwiOlsiaG9nZUBleGFtcGxlLmNvbSJdfSwic2lnbl9pbl9wcm92aWRlciI6InBhc3N3b3JkIn19
```

## Signatureを作成

### HeaderとPayloadのsha256ハッシュを取得

```bash
$ echo -n 'eyJhbGciOiJSUzI1NiIsImtpZCI6InRla2l0b3UiLCJ0eXAiOiJKV1QifQ.eyJpc3MiOiJodHRwczovL2hvZ2UiLCJhdWQiOiJob3ppZGV2IiwiYXV0aF90aW1lIjoxNjIwMjA2MjI0LCJ1c2VyX2lkIjoiZm9vIiwic3ViIjoiZm9vaG9nZSIsImlhdCI6MTYyMDIwNjIyNCwiZXhwIjoxNjIwMjA5ODI0LCJlbWFpbCI6ImhvZ2VAZXhhbXBsZS5jb20iLCJlbWFpbF92ZXJpZmllZCI6ZmFsc2UsImZpcmViYXNlIjp7ImlkZW50aXRpZXMiOnsiZW1haWwiOlsiaG9nZUBleGFtcGxlLmNvbSJdfSwic2lnbl9pbl9wcm92aWRlciI6InBhc3N3b3JkIn19' | \
shasum -a 256

1952f4eb9a37c4edc3ddd7574fa6e38cd170beedc0edfa5b6c6ae3027c9ba1e7
```

`1952f4eb9a37c4edc3ddd7574fa6e38cd170beedc0edfa5b6c6ae3027c9ba1e7`を秘密鍵で署名します。

### 秘密鍵で署名
```bash
$ export JWT_HEADER=eyJhbGciOiJSUzI1NiIsImtpZCI6InRla2l0b3UiLCJ0eXAiOiJKV1QifQ
$ export JWT_PAYLOAD=eyJpc3MiOiJodHRwczovL2hvZ2UiLCJhdWQiOiJob3ppZGV2IiwiYXV0aF90aW1lIjoxNjIwMjA2MjI0LCJ1c2VyX2lkIjoiZm9vIiwic3ViIjoiZm9vaG9nZSIsImlhdCI6MTYyMDIwNjIyNCwiZXhwIjoxNjIwMjA5ODI0LCJlbWFpbCI6ImhvZ2VAZXhhbXBsZS5jb20iLCJlbWFpbF92ZXJpZmllZCI6ZmFsc2UsImZpcmViYXNlIjp7ImlkZW50aXRpZXMiOnsiZW1haWwiOlsiaG9nZUBleGFtcGxlLmNvbSJdfSwic2lnbl9pbl9wcm92aWRlciI6InBhc3N3b3JkIn19
$ echo -n "${JWT_HEADER}.${JWT_PAYLOAD}" | \
openssl dgst -sha256 -binary -sign private.key | \
base64

QAEoPI3B+R87Y3qRvIT7pWDf1KsPuLYiHHJMPOB8ERqljNVHXY3HQuEJ3/0ZGbT8dzRt9b8WZqn4fmVHe5OqNQO9bIJKk506Ic+Aghzk69Fpo6QPUtC5WLP1eD2RBaDc9dpoMhVT/6ebDOrNA9ZIna2RHRfc9xpjyYOakSNgG+/ux32gfTqJwj7JsW/PL5EcHbUHnF+0MdPS1EZ8CShKYsEXG8hI6O78zaKphSUMj6Z8eoUsBzAqXeyTjf11TfD+Ef1sOWGt0RHxjLXNNTYFst4pkxcRHguYvbBG3nJvVsTILWm10gRRLYXGQX8AFKsDW7ztEWLW2lLcGpTkg3i+wA==
```

末尾の`==`を削ります
```bash
QAEoPI3B+R87Y3qRvIT7pWDf1KsPuLYiHHJMPOB8ERqljNVHXY3HQuEJ3/0ZGbT8dzRt9b8WZqn4fmVHe5OqNQO9bIJKk506Ic+Aghzk69Fpo6QPUtC5WLP1eD2RBaDc9dpoMhVT/6ebDOrNA9ZIna2RHRfc9xpjyYOakSNgG+/ux32gfTqJwj7JsW/PL5EcHbUHnF+0MdPS1EZ8CShKYsEXG8hI6O78zaKphSUMj6Z8eoUsBzAqXeyTjf11TfD+Ef1sOWGt0RHxjLXNNTYFst4pkxcRHguYvbBG3nJvVsTILWm10gRRLYXGQX8AFKsDW7ztEWLW2lLcGpTkg3i+wA
```

```bash:完成したJWT
eyJhbGciOiJSUzI1NiIsImtpZCI6InRla2l0b3UiLCJ0eXAiOiJKV1QifQ.eyJpc3MiOiJodHRwczovL2hvZ2UiLCJhdWQiOiJob3ppZGV2IiwiYXV0aF90aW1lIjoxNjIwMjA2MjI0LCJ1c2VyX2lkIjoiZm9vIiwic3ViIjoiZm9vaG9nZSIsImlhdCI6MTYyMDIwNjIyNCwiZXhwIjoxNjIwMjA5ODI0LCJlbWFpbCI6ImhvZ2VAZXhhbXBsZS5jb20iLCJlbWFpbF92ZXJpZmllZCI6ZmFsc2UsImZpcmViYXNlIjp7ImlkZW50aXRpZXMiOnsiZW1haWwiOlsiaG9nZUBleGFtcGxlLmNvbSJdfSwic2lnbl9pbl9wcm92aWRlciI6InBhc3N3b3JkIn19.QAEoPI3B+R87Y3qRvIT7pWDf1KsPuLYiHHJMPOB8ERqljNVHXY3HQuEJ3/0ZGbT8dzRt9b8WZqn4fmVHe5OqNQO9bIJKk506Ic+Aghzk69Fpo6QPUtC5WLP1eD2RBaDc9dpoMhVT/6ebDOrNA9ZIna2RHRfc9xpjyYOakSNgG+/ux32gfTqJwj7JsW/PL5EcHbUHnF+0MdPS1EZ8CShKYsEXG8hI6O78zaKphSUMj6Z8eoUsBzAqXeyTjf11TfD+Ef1sOWGt0RHxjLXNNTYFst4pkxcRHguYvbBG3nJvVsTILWm10gRRLYXGQX8AFKsDW7ztEWLW2lLcGpTkg3i+wA==
```

### 公開鍵で検証
鍵長を確認
```bash
$ echo -n "QAEoPI3B+R87Y3qRvIT7pWDf1KsPuLYiHHJMPOB8ERqljNVHXY3HQuEJ3/0ZGbT8dzRt9b8WZqn4fmVHe5OqNQO9bIJKk506Ic+Aghzk69Fpo6QPUtC5WLP1eD2RBaDc9dpoMhVT/6ebDOrNA9ZIna2RHRfc9xpjyYOakSNgG+/ux32gfTqJwj7JsW/PL5EcHbUHnF+0MdPS1EZ8CShKYsEXG8hI6O78zaKphSUMj6Z8eoUsBzAqXeyTjf11TfD+Ef1sOWGt0RHxjLXNNTYFst4pkxcRHguYvbBG3nJvVsTILWm10gRRLYXGQX8AFKsDW7ztEWLW2lLcGpTkg3i+wA==" | \
base64 -D | \
hexdump -C

00000000  40 01 28 3c 8d c1 f9 1f  3b 63 7a 91 bc 84 fb a5  |@.(<....;cz.....|
00000010  60 df d4 ab 0f b8 b6 22  1c 72 4c 3c e0 7c 11 1a  |`......".rL<.|..|
00000020  a5 8c d5 47 5d 8d c7 42  e1 09 df fd 19 19 b4 fc  |...G]..B........|
00000030  77 34 6d f5 bf 16 66 a9  f8 7e 65 47 7b 93 aa 35  |w4m...f..~eG{..5|
00000040  03 bd 6c 82 4a 93 9d 3a  21 cf 80 82 1c e4 eb d1  |..l.J..:!.......|
00000050  69 a3 a4 0f 52 d0 b9 58  b3 f5 78 3d 91 05 a0 dc  |i...R..X..x=....|
00000060  f5 da 68 32 15 53 ff a7  9b 0c ea cd 03 d6 48 9d  |..h2.S........H.|
00000070  ad 91 1d 17 dc f7 1a 63  c9 83 9a 91 23 60 1b ef  |.......c....#`..|
00000080  ee c7 7d a0 7d 3a 89 c2  3e c9 b1 6f cf 2f 91 1c  |..}.}:..>..o./..|
00000090  1d b5 07 9c 5f b4 31 d3  d2 d4 46 7c 09 28 4a 62  |...._.1...F|.(Jb|
000000a0  c1 17 1b c8 48 e8 ee fc  cd a2 a9 85 25 0c 8f a6  |....H.......%...|
000000b0  7c 7a 85 2c 07 30 2a 5d  ec 93 8d fd 75 4d f0 fe  ||z.,.0*]....uM..|
000000c0  11 fd 6c 39 61 ad d1 11  f1 8c b5 cd 35 36 05 b2  |..l9a.......56..|
000000d0  de 29 93 17 11 1e 0b 98  bd b0 46 de 72 6f 56 c4  |.)........F.roV.|
000000e0  c8 2d 69 b5 d2 04 51 2d  85 c6 41 7f 00 14 ab 03  |.-i...Q-..A.....|
000000f0  5b bc ed 11 62 d6 da 52  dc 1a 94 e4 83 78 be c0  |[...b..R.....x..|
00000100
```

署名をbase64でデコード
```bash
$ echo -n "QAEoPI3B+R87Y3qRvIT7pWDf1KsPuLYiHHJMPOB8ERqljNVHXY3HQuEJ3/0ZGbT8dzRt9b8WZqn4fmVHe5OqNQO9bIJKk506Ic+Aghzk69Fpo6QPUtC5WLP1eD2RBaDc9dpoMhVT/6ebDOrNA9ZIna2RHRfc9xpjyYOakSNgG+/ux32gfTqJwj7JsW/PL5EcHbUHnF+0MdPS1EZ8CShKYsEXG8hI6O78zaKphSUMj6Z8eoUsBzAqXeyTjf11TfD+Ef1sOWGt0RHxjLXNNTYFst4pkxcRHguYvbBG3nJvVsTILWm10gRRLYXGQX8AFKsDW7ztEWLW2lLcGpTkg3i+wA==" | \
base64 -D > my_jwt.sign
```

証明書(公開鍵)を使って、署名を検証し、↑で秘密鍵で署名した`1952f4eb9a37c4edc3ddd7574fa6e38cd170beedc0edfa5b6c6ae3027c9ba1e7`と一致することを確認
```bash
$ openssl rsautl -verify -asn1parse -in my_jwt.sign -certin -inkey rootCA.pem
    0:d=0  hl=2 l=  49 cons: SEQUENCE
    2:d=1  hl=2 l=  13 cons:  SEQUENCE
    4:d=2  hl=2 l=   9 prim:   OBJECT            :sha256
   15:d=2  hl=2 l=   0 prim:   NULL
   17:d=1  hl=2 l=  32 prim:  OCTET STRING
      0000 - 19 52 f4 eb 9a 37 c4 ed-c3 dd d7 57 4f a6 e3 8c   .R...7.....WO...
      0010 - d1 70 be ed c0 ed fa 5b-6c 6a e3 02 7c 9b a1 e7   .p.....[lj..|...
```

# さいごに
誤った記述や不明点な点がありましたら、[こちら](https://github.com/hozi-dev/article/issues)まで。

## 時間があったら下記追記
* 鍵長の話
* 検証ロジックのシーケンス

# 参考
* [JWT ハンドブック(Auth0)](https://assets.ctfassets.net/2ntc334xpx65/5HColfm15cUhMmDQnupNzd/30d5913d94e79462043f6d8e3f557351/jwt-handbook-jp.pdf)
* [Generate a JWT with RSA keys](https://learn.akamai.com/en-us/webhelp/iot/jwt-access-control/GUID-CB17F8FF-3367-4D4B-B3FE-FDBA53A5EA02.html)
  * OpenSSLでSignatureを作成する方法が参考になった(なぜか日本の記事はプログラム言語を実行するパターンが多かった)
