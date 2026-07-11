---
title: "AWS IoTで独自CA証明書を使ってMQTT通信を試してみる"
description: "AWS IoTで独自CA証明書を使ってMQTT通信を試してみる"
publish: true
tags:
  - "tech/iot"
  - "tech/aws"
  - "tech/mqtt"
  - "tech/セキュリティ"
---

# はじめに
[AWS IoTがサポートするX.509証明書(AWS)](https://docs.aws.amazon.com/ja_jp/iot/latest/developerguide/x509-client-certs.html)は下記。本記事では項2を試します。

1. AWSIoTによって生成されたX.509証明書
1. AWSIoTに登録されたCAによって署名されたX.509証明書
1. AWSIoTに登録されていないCAによって署名されたX.509証明書(JITRのこと?)

余談だが、項1のようにAmazonのルートCA証明書で署名された証明書、公開鍵、秘密鍵を下記のワンコマンドで発行し、AWS IoT側に登録可能。
```bash
aws iot create-keys-and-certificate \
    --set-as-active \
    --certificate-pem-outfile certificate_filename \
    --public-key-outfile public_key_filename \
    --private-key-outfile private_key_filename
```
デメリットとして、クライアント証明書秘密鍵がまず発行する際にネットワーク上に晒され、次にデバイス側に反映する際にもやり方次第で晒される可能性がある。

## 注意点
本記事は、全体的にもやもやが多く、推測を多く含む記事になっているため、間違えがあれば[こちら](https://github.com/hozi-dev/article/issues)から起票お願いします。

# MacでOpenSSLをインストール
MacのデフォルトのLibreSSLは使わず、brewでインストールしたOpenSSLバイナリを利用
```bash
$ openssl version
LibreSSL 2.8.3
```

```bash
$ brew install openssl
$ brew --prefix openssl
/usr/local/opt/openssl@1.1
$ export PATH="/usr/local/opt/openssl@1.1/bin:$PATH"
$ openssl version
OpenSSL 1.1.1k  25 Mar 2021
```

# CA証明書をAWSに登録する
本手順は、[Register your CA certificate(AWS)](https://docs.aws.amazon.com/iot/latest/developerguide/register-CA-cert.html)通りに実施します

## CA証明書の作成
RSA秘密鍵の作成

```bash
openssl genrsa -out rootCA.key 2048
```
`rootCA.key`というファイルが出力

ルートCA証明書の作成

```bash
openssl req -x509 -new -nodes \
    -key rootCA.key \
    -sha256 -days 1024 \
    -subj "/C=JP/ST=Tokyo/O=shuntaka corp./CN=shuntaka root 2021" \
    -out rootCA.pem
# AWSの手順は-subjオプションがない。これは対話型がクソ面倒なので追記した。
```
`rootCA.pem`というファイルが出力される。下記のコマンド内容を確認できる。
IssuerとSubjectが同じなので、自己署名証明書という事になる

```bash
$ openssl x509 -text -noout -in rootCA.pem

Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            3d:e9:23:37:af:80:57:88:9e:3c:6a:41:1b:27:32:bf:bd:88:3d:60
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = JP, ST = Tokyo, O = shuntaka corp., CN = shuntaka root 2021
        Validity
            Not Before: Apr 30 08:00:19 2021 GMT
            Not After : Feb 18 08:00:19 2024 GMT
        Subject: C = JP, ST = Tokyo, O = shuntaka corp., CN = shuntaka root 2021
(中略)
```

:::details (余談) SSL/TLSの場合
本ブログの証明書を確認してみる

```bash
openssl s_client -connect shuntaka.dev:443 -showcerts

CONNECTED(00000005)
depth=2 O = Digital Signature Trust Co., CN = DST Root CA X3
verify return:1
depth=1 C = US, O = Let's Encrypt, CN = R3
verify return:1
depth=0 CN = shuntaka.dev
verify return:1
---
Certificate chain
 0 s:CN = shuntaka.dev
   i:C = US, O = Let's Encrypt, CN = R3
# 1
-----BEGIN CERTIFICATE-----
(中略)
-----END CERTIFICATE-----
 1 s:C = US, O = Let's Encrypt, CN = R3
   i:O = Digital Signature Trust Co., CN = DST Root CA X3
# 2
-----BEGIN CERTIFICATE-----
(中略)
-----END CERTIFICATE-----
(中略)
read R BLOCK
```

1の---BEGINからEND CERTIFICATE---には、証明書
2の---BEGINからEND CERTIFICATE---には、中間証明書
それぞれ1と2というファイルにし、OpenSSLでデコードする

Issuer(発行者=証明書を発行した主体)は、Let's Encrypt
Subject(主体者=発行を受けた者)は、blogl.hozi.dev

```bash
$ openssl x509 -text -noout -in 1
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            04:88:f3:80:92:2a:ca:36:6b:5d:1c:0a:56:91:7d:fd:be:7a
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = US, O = Let's Encrypt, CN = R3
        Validity
            Not Before: Mar 10 22:28:02 2021 GMT
            Not After : Jun  8 22:28:02 2021 GMT
        Subject: CN = shuntaka.dev
(中略)
```

Issuer(発行者=証明書を発行した主体)は、Digital Signature Trust Co.
Subject(主体者=発行を受けた者)は、Let's Encrypt
```bash
$ openssl x509 -text -noout -in 2
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            40:01:75:04:83:14:a4:c8:21:8c:84:a9:0c:16:cd:df
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: O = Digital Signature Trust Co., CN = DST Root CA X3
        Validity
            Not Before: Oct  7 19:21:40 2020 GMT
            Not After : Sep 29 19:21:40 2021 GMT
        Subject: C = US, O = Let's Encrypt, CN = R3
(中略)
```

中間証明書に書いていある`DST Root CA X3`は、Macの場合キーチェーン(トラストストア)に保存されており、確認可能。
サブジェクト名(Subject)と発行者名(Issuer)が一致しいる。これはルートCA証明書であるため。
![image](https://user-images.githubusercontent.com/12817245/116668547-7f609c00-a9d8-11eb-9e80-2983583ac795.png)
:::

# CA証明書をIoT Coreへ登録

## 登録コードの取得
`get-registration-code`を使用してAWSIoTから登録コードを取得する。
registrationCode返されたものを保存して Common Name秘密鍵検証証明書として使用します。(?)

```bash
$ aws iot get-registration-code
{
    "registrationCode": "axxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
}
```

## 秘密鍵検証証明書の鍵ペアを生成

```bash
$ openssl genrsa -out verificationCert.key 2048
```

## 秘密鍵検証証明書のCSRを作成
秘密鍵検証証明書の証明書署名要求(CSR)を作成。
Common Name証明書のフィールドを`get-registration-coderegistrationCode`によって返されるものに設定。

```bash
openssl req -new \
    -subj "/CN=axxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" \
    -key verificationCert.key \
    -out verificationCert.csr
```
verificationCert.csrが作成される

## 秘密鍵検証証明書
```bash
openssl x509 -req \
    -in verificationCert.csr \
    -CA rootCA.pem \
    -CAkey rootCA.key \
    -CAcreateserial \
    -out verificationCert.pem \
    -days 500 -sha256
```
verificationCert.pemが作成される

内容を確認指定みると、IssuerとSubjectが適切な値であると分かる
```bash
$ openssl x509 -text -noout -in verificationCert.pem
Certificate:
    Data:
        Version: 1 (0x0)
        Serial Number:
            59:09:3d:2b:4d:e0:66:6d:a1:ae:03:68:81:71:a1:84:ed:e6:90:69
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = JP, ST = Tokyo, O = shuntaka corp., CN = shuntaka root 2021
        Validity
            Not Before: Apr 30 09:49:19 2021 GMT
            Not After : Sep 12 09:49:19 2022 GMT
        Subject: CN = a5e84cdbbc102f02b16e62bd611101d20106da39badc0c872a4298b1e6087f51
```

## CA証明書をAWSIoT側に登録

```bash
$ aws iot register-ca-certificate \
    --ca-certificate file://rootCA.pem \
    --verification-cert file://verificationCert.pem
{
    "certificateArn": "arn:aws:iot:ap-northeast-1:アカウントID:cacert/7xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "certificateId": "7xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
}
```

## CA証明書をアクティブ化

**更新前**
![更新前](https://user-images.githubusercontent.com/12817245/116681588-213bb500-a9e8-11eb-81ad-75fe3afc6402.png)

```bash:CA証明書をアクティブ化
aws iot update-ca-certificate \
    --certificate-id 7xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx \
    --new-status ACTIVE
```

**更新後**
![更新後](https://user-images.githubusercontent.com/12817245/116681728-4f20f980-a9e8-11eb-8cc9-c921bf3b3975.png)

# クライアント証明書の作成

## クライアント証明書秘密鍵を作成
```bash
openssl genrsa -out deviceCert.key 2048
```

## クライアント証明書CSRを作成
subjには**デバイス固有の値**を入れるのが適切だと思う。

```bash
openssl req -new \
  -subj "/C=JP/ST=Tokyo/O=shuntaka corp./CN=deviceHoge" \
  -key deviceCert.key \
  -out deviceCert.csr
```

## クライアント証明書を作成
```bash
openssl x509 -req \
  -in deviceCert.csr \
  -CA rootCA.pem \
  -CAkey rootCA.key \
  -CAcreateserial \
  -out deviceCert.pem \
  -days 500 -sha256
```

おなじみの確認コマンド。発行者にshuntak root 2021があり、SubjectはCSR作成時に指定したものが含まれているので問題なし。
```bash
$ openssl x509 -text -noout -in deviceCert.pem
Certificate:
    Data:
        Version: 1 (0x0)
        Serial Number:
            59:09:3d:2b:4d:e0:66:6d:a1:ae:03:68:81:71:a1:84:ed:e6:90:6a
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = JP, ST = Tokyo, O = shuntaka corp., CN = shuntaka root 2021
        Validity
            Not Before: Apr 30 10:38:36 2021 GMT
            Not After : Sep 12 10:38:36 2022 GMT
        Subject: C = JP, ST = Tokyo, O = shuntaka corp., CN = deviceHoge
```

# クライアント証明書を手動登録する
`register-certificate`を省略すると、アクティブ化しない。

```bash
$ aws iot register-certificate \
    --set-as-active \
    --certificate-pem file://deviceCert.pem \
    --ca-certificate-pem file://rootCA.pem
{
    "certificateArn": "arn:aws:iot:ap-northeast-1:<AWSアカウントID>:cert/1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "certificateId": "1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
}
```

:::details CA証明書に登録してない適当なファイルを指定すると、エラーになる。
```bash
$ aws iot register-certificate \
    --set-as-active \
    --certificate-pem file://deviceCert.pem \
    --ca-certificate-pem file://verificationCert.pem

An error occurred (CertificateStateException) when calling the RegisterCertificate operation: No CA certificate exists for the given certificate
```
:::

# MQTT接続を試す
## mosquittoをインストール
```bash
brew install mosquitto
```

## CA証明書とクライアント証明書を連結
```bash
cat deviceCert.pem rootCA.pem > deviceCertAndCACert.crt
```

## AWS Root CA証明書をDL
AmazonRootCA1.pemという名前で保存されます。

```bash
wget https://www.amazontrust.com/repository/AmazonRootCA1.pem
```

## IoT policyを作成
```bash
cat <<EOF > iot-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "iot:*",
      "Resource": "*"
    }
  ]
}
EOF
```

```bash
aws iot create-policy --policy-name "PubSubToAnyTopic" --policy-document file://iot-policy.json
```

## ポリシーをクライアント証明書にアタッチ

```bash
aws iot attach-principal-policy --principal "arn:aws:iot:ap-northeast-1:<AWSアカウントID>:cert/1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" --policy-name "PubSubToAnyTopic"
```


## MQTTでイベント送信
`--cafile`は、暗号化を有効にするための信頼できるCA証明書を含むファイルへのパスを指定する、このオプションを指定することで、`MQTT over TLS`になる。`--capath`でディレクトリを指すことも可能。(ブラウザでいうところのトラストストアにみたいな運用が可能?)
`--cert`と`--key`オプションは、サーバー側で必要な場合、クライアント証明書とその秘密鍵が必要になる。特にサーバー側で必要ない場合は、[Mosquittoを用いてMQTT＋SSL/TLS通信を試してみる](https://qiita.com/udai1532/items/c0f58e73f76900a8469f)のようになくても`MQTT over TLS`になるはず(?)。また、クライアント証明書秘密鍵は、オプションで指定しているが、ネットワークには晒されないものと推測している。

```bash
mosquitto_pub \
  --cafile AmazonRootCA1.pem \
  --cert deviceCertAndCACert.crt \
  --key deviceCert.key \
  -h <IOTエンドポイント>-ats.iot.ap-northeast-1.amazonaws.com \
  -p 8883 \
  -q 1 \
  -t topic \
  -i anyclientID \
  --tls-version tlsv1.2 \
  -m '{"event": "hello"}' -d
```

```bash:結果
Client anyclientID sending CONNECT
Client anyclientID received CONNACK (0)
Client anyclientID sending PUBLISH (d0, q1, r0, m1, 'topic', ... (18 bytes))
Client anyclientID received PUBACK (Mid: 1, RC:0)
Client anyclientID sending DISCONNECT
```

テストツールで取得でイベントの送信を確認
![imag](https://user-images.githubusercontent.com/12817245/116689518-336f2080-a9f3-11eb-921f-9c21506ef0c8.png)

---

# もやもや
## 秘密鍵検証証明書とは..?
Common NameにAWSアカウント一意なIDを入れて作成した証明書。AWS側でどのような目的で使われているかよくわからない。
推測だが、AWS IoT側には、CA証明書と秘密鍵検証証明書しか登録していないので、登録者がCA証明書の秘密鍵を持っていることを秘密鍵をネットワークに晒すことなく証明するためのもの。
秘密鍵検証証明書は、CA証明書の秘密鍵で署名されている。なので公開鍵で署名を検証すれば、CA証明書の秘密鍵を持っている証明になる。(しかも証明書にはAWSアカウントに一意なCommon Nameも入っているため、どこかからパクってきたCA証明書と中間証明書を突っ込んでも使うことはできない)

## MQTT接続時、AmazonのルートCA証明書を指定する理由
[サーバー認証(AWS)](https://docs.aws.amazon.com/ja_jp/iot/latest/developerguide/server-authentication.html)より引用。

>デバイスがAWS IoT Coreに接続しようとすると、AWS IoT Coreサーバーは、デバイスがサーバーの認証に使用するX.509証明書を送信する。
>認証は、X.509証明書チェーンの検証を通じてTLSレイヤーで行われる。**これは、HTTPSURLにアクセスするときにブラウザーで使用されるのと同じ方法です。**

MQTT接続する際に、AmazonのルートCA証明書を指定してる。引用の太字部から、クライアントはトラストストアを持っていないため、ルート証明書を持っておく必要があると推測している。

>サーバー認証用のCA証明書
>使用しているデータエンドポイントのタイプとネゴシエートした暗号スイートに応じて、AWS IoTCoreサーバー認証証明書は次のルートCA証明書のいずれかによって署名されます。

ここでAmazonルートCA 1の記述あり。

>サーバー認証ガイドライン
>AWS IoTCoreサーバー認証証明書を検証するデバイスの機能に影響を与える可能性のある多くの変数があります。たとえば、デバイスのメモリが制限されすぎて、考えられるすべてのルートCA証明書を保持できない場合や、デバイスが非標準の証明書検証方法を実装している場合があります。

[X.509クライアント証明書の使用](https://docs.aws.amazon.com/ja_jp/iot/latest/developerguide/x509-client-certs.html)より引用。

> TLSクライアント認証では、AWS IoTはX.509クライアント証明書を要求し、証明書のステータスとAWSアカウントを証明書のレジストリに対して検証します。次に、証明書に含まれている公開鍵に対応する秘密鍵の所有権の証明をクライアントに要求します。

前述サーバー認証の話と変わり、クライアント認証の話。秘密鍵の所有権の証明をクライアントに要求はどの行っているのか謎。

これらの記述から、下記のような認証シーケンスがあるのではと予測(推測を非常に多く含みます..)
![image](https://user-images.githubusercontent.com/12817245/116724546-c2902e80-aa1b-11eb-8247-e246601fdfcf.png)

こうみると、デバイスがサーバーのサーバーがデバイスのルート証明書をそれぞれ持っており、認証しあっていることが分かる(当たり前か..)

# その他
## VeriSignクラス3パブリックプライマリG5ルートCA証明書の扱い
AWS IoTには、2つの異なるデータエンドポイントタイプを[用意している](https://docs.aws.amazon.com/ja_jp/iot/latest/developerguide/server-authentication.html#endpoint-types)。各エンドポイントのルート証明書が異なる。

* iot:Data
  * account-specific-prefix.iot.your-region.amazonaws.com
  * VeriSignクラス3パブリックプライマリG5ルートCA証明書
* iot:Data-ATS
  * account-specific-prefix **-ats** .iot.your-region.amazonaws.com
  * Amazon TrustServicesによって署名されたルートCA証明書

:::details opensslでルート証明書を確認した結果
```bash
$ openssl s_client -connect <ACCOUNT_PREFIX>.iot.ap-northeast-1.amazonaws.com:443 -showcerts
CONNECTED(00000006)
depth=2 C = US, O = "VeriSign, Inc.", OU = VeriSign Trust Network, OU = "(c) 2006 VeriSign, Inc. - For authorized use only", CN = VeriSign Class 3 Public Primary Certification Authority - G5
```

```bash
$ openssl s_client -connect <ACCOUNT_PREFIX>-ats.iot.ap-northeast-1.amazonaws.com:443 -showcerts
CONNECTED(00000006)
depth=4 C = US, O = "Starfield Technologies, Inc.", OU = Starfield Class 2 Certification Authority
verify return:1
depth=3 C = US, ST = Arizona, L = Scottsdale, O = "Starfield Technologies, Inc.", CN = Starfield Services Root Certificate Authority - G2
```
:::

[boto3でIoT CoreのATSエンドポイントに接続する（SSL validation failedエラー対応）](https://dev.classmethod.jp/articles/boto3-iot-core-use-ats-endpointo-ssl-validation-failed/)にある通り、boto3のcertifiで`VeriSign Class 3 Public Primary Certification Authority - G5 O`のルート証明書が削除された。
boto3は、デフォルトでiot:Dataを参照しており、certifiでVeriSignルート証明書が削除されたことで、ルート証明書の検証が出来ず、SSL検証でエラーになった。VeriSignルート証明書が削除されたcertifiと依存があるboto3を使った場合明示的にiot:Data-ATSエンドポイントを指定する必要があある。

実際に経験した例だと、AWS Lambdaからboto3を使ってiot:Dataエンドポイントを参照してモノ情報を取得していた。Lambdaのランタイムにはboto3が元々入っており、新しいバージョンの(VeriSignルートCAが削除された)certifiに依存したboto3が使われたため、iot:Dataエンドポイントの証明書を受け取ったとき、証明書チェーンを確認し、boto3側でVeriSignのルートCAがないので、検証できずサーバー認証でエラーになる。

:::details MQTT接続をiot:Dataエンドポイントに行うと、TLSのエラーが確認できる
ACCOUNT_PREFIXに、 **-ats** なし。
```bash
$ mosquitto_pub \
  --cafile AmazonRootCA1.pem \
  --cert deviceCertAndCACert.crt \
  --key deviceCert.key \
  -h <ACCOUNT_PREFIX>.iot.ap-northeast-1.amazonaws.com \
  -p 8883 \
  -q 1 \
  -t topic \
  -i anyclientID \
  --tls-version tlsv1.2 \
  -m '{"event": "hello"}' -d
```
もちろんこれは、`--cafile`にAmazon TrustServicesによって署名されたルートCA証明書を指定しているため。

```bash:結果
Client anyclientID sending CONNECT
OpenSSL Error[0]: error:1416F086:SSL routines:tls_process_server_certificate:certificate verify failed
Error: A TLS error occurred.
```
:::


# 参考
* [X.509クライアント証明書(AWS)](https://docs.aws.amazon.com/ja_jp/iot/latest/developerguide/x509-client-certs.html)
* [AWS IoTでデバイス証明書のジャストインタイム登録をやってみた](https://dev.classmethod.jp/articles/aws-iot-just-in-time-registration/)
* [AWS IoTにおけるデバイスへの認証情報のプロビジョニング](https://pages.awscloud.com/rs/112-TZM-766/images/EV_iot-deepdive-aws2_Sep-2020.pdf)


