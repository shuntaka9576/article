---
title: "[M5Stack Core2] PlatformIOでesp-aws-iotのサンプルsubscribe_publishを動かしてみる"
type: "tech"
description: "esp-aws-iotに入門する上で良さそうなミニマム実装をM5Stack Core2上で動かします"
publish: true
tags:
  - "tech/iot"
  - "tech/aws"
  - "tech/arduino"
  - "tech/mqtt"
---

# はじめに
ESP32で、AWS IoTと通信する[esp-aws-iot](https://github.com/espressif/esp-aws-iot)というライブラリがあります。同リポジトリの[examples](https://github.com/espressif/esp-aws-iot/tree/master/examples/subscribe_publish)に、ESP32ボードからAWS IoTに対してMQTT通信を行うサンプルが用意されています。
サンプルは、ESP-IDFを使うとすぐに試せますが、自分のプロジェクトに組み込む且つ、PlatformIOを使う想定の場合、設定ファイルを修正する必要があり、備忘のため記事にしました。


# リポジトリ
手順を省略出来るように[shuntaka9576/esp-aws-iot-subscribe-publish-sample](https://github.com/shuntaka9576/esp-aws-iot-subscribe-publish-sample.git)を用意しています。リポジトリを使う場合は、[モノと証明書の作成](#モノと証明書の作成)から手順を進められます。

:::message
`shuntaka9576/esp-aws-iot-subscribe-publish-sample`を利用する場合、submoduleを再起的に取得するコマンドを実行してください。
```bash
$ git clone https://github.com/shuntaka9576/esp-aws-iot-subscribe-publish-sample.git
$ cd esp-aws-iot-subscribe-publish-sample 
$ git submodule update --init --recursive
```
:::


# プロジェクトの作成
## プロジェクトの初期化
```bash
pio init --board=m5stack-core2 --project-option="framework=espidf"
```

## esp-aws-iotライブラリの追加
```bash
git submodule add https://github.com/espressif/esp-aws-iot.git components/esp-aws-iot
```

`esp-aws-iot`は、`aws-iot-device-sdk-embedded-C`をsubmoduleとしている。上記のコマンドだけでは、`aws-iot-device-sdk-embedded-C`はクローンされないため、下記のコマンドで再起的にgit submoduleをクローンする

```bash
git submodule update --init --recursive
```

## platformio.iniの修正

```diff:platformio.ini
[env:m5stack-core2]
platform = espressif32
board = m5stack-core2
framework = espidf
+monitor_speed = 115200
+
+board_build.embed_txtfiles =
+  src/certs/aws-root-ca.pem
+  src/certs/certificate.pem.crt
+  src/certs/private.pem.key
```


## [examples/subscribe_publish](https://github.com/espressif/esp-aws-iot/tree/249c573cbdcf653c03a6bf3a30f658f005856880/examples/subscribe_publish)を参考にファイルを移植

```bash:src/CMakeLists.txt
# This file was automatically generated for projects
# without default 'CMakeLists.txt' file.

FILE(GLOB_RECURSE app_sources ${CMAKE_SOURCE_DIR}/src/*.*)

set(COMPONENT_SRCS "main.c")
set(COMPONENT_ADD_INCLUDEDIRS ".")

register_component()

if(CONFIG_EXAMPLE_EMBEDDED_CERTS)
target_add_binary_data(${COMPONENT_TARGET} "certs/aws-root-ca.pem" TEXT)
target_add_binary_data(${COMPONENT_TARGET} "certs/certificate.pem.crt" TEXT)
target_add_binary_data(${COMPONENT_TARGET} "certs/private.pem.key" TEXT)
endif()
```

* `src/Kconfig.projbuild`は、[Kconfig.projbuild](https://github.com/espressif/esp-aws-iot/blob/249c573cbdcf653c03a6bf3a30f658f005856880/examples/subscribe_publish/main/Kconfig.projbuild)の内容をそのまま利用
* `src/component.mk`は、[component.mk](https://github.com/espressif/esp-aws-iot/blob/249c573cbdcf653c03a6bf3a30f658f005856880/examples/subscribe_publish/main/component.mk)の内容をそのまま利用
* `src/main.c`は、[subscribe_publish_sample.c](https://github.com/espressif/esp-aws-iot/blob/249c573cbdcf653c03a6bf3a30f658f005856880/examples/subscribe_publish/main/subscribe_publish_sample.c)の内容をそのまま利用

:::message
`src/Kconfig.projbuild`はコミットされるファイルです。デフォルト値に秘匿したい値は設定しないでください。
`pio run -t menuconfig`経由で設定し、sdkconfigとして最終的に設定が出力されます。sdkconfigはgitginoreします。
:::

## モノと証明書の作成

モノを作成
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1632558785/blog/01fgdc0bawyb6d34gs54vxgpg9/hhokeizbuzynvbqfnqlw.png)

![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1632558857/blog/01fgdc0bawyb6d34gs54vxgpg9/hxql73nmurtp9ous6t4b.png)
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1632558942/blog/01fgdc0bawyb6d34gs54vxgpg9/rzliwmnqxnmlo025siwx.png)

ポリシーは後ほど作成します
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1632558980/blog/01fgdc0bawyb6d34gs54vxgpg9/o0w7wsmdtpcqmndtrevl.png)

![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1632559174/blog/01fgdc0bawyb6d34gs54vxgpg9/ai0wijt2pwlsuccxau5f.png)

ポリシーを作成します。名前は、`esp32-test-policy`とします。

```bash:ポリシーステートメント
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
```

![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1632560868/blog/01fgdc0bawyb6d34gs54vxgpg9/yrto2fivnrmhxkiz3udc.png)

作成できたら、証明書にポリシーをアタッチします。
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1632561069/blog/01fgdc0bawyb6d34gs54vxgpg9/kfqk8gyply6pyi9nqiku.png)
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1632561142/blog/01fgdc0bawyb6d34gs54vxgpg9/xggiqagdpgf9eeukr1rw.png)


ダウンロードした秘密鍵や証明書を、`./src/certs`配下にリネームしつつ、配置します。

|名称|ファイル名|リネームして配置する場所
|---|---|---|
|デバイス証明書|xxxx-certificate.pem.crt|./src/certs/certificate.pem.crt
|デバイス証明書秘密鍵|xxxx-private.pem.key|./src/certs/private.pem.key
|デバイス証明書公開鍵|xxxx-public.pem.key|利用しない
|AmazonルートCA証明書|AmazonRootCA1.pem|./src/certs/aws-root-ca.pem

最終的にsrc配下は、下記となります。

```bash:srcのファイルの構成
src
├── CMakeLists.txt
├── Kconfig.projbuild
├── certs
│   ├── aws-root-ca.pem
│   ├── certificate.pem.crt
│   └── private.pem.key
├── component.mk
└── main.c
```

## sdkconfigの生成

コマンドを実行すると、TUIが起動します
```bash
pio run -t menuconfig
```

`Example Configureation`を選択します。補足ですが、本設定の枠組みは`src/Kconfig.projbuild`に記述されています。
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1632559789/blog/01fgdc0bawyb6d34gs54vxgpg9/q2d8dcnlmoazyirreanc.png)

wifiのSSIDを入力後、Enterを押下
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1632559771/blog/01fgdc0bawyb6d34gs54vxgpg9/pphvh2siwmg4mtcuajeq.png)

wifiのパスワードを入力後、Enterを押下
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1632559774/blog/01fgdc0bawyb6d34gs54vxgpg9/n11kqz9ewazcyqrmbkex.png)

入力完了したら、ESCを1回押下し、1つ前の選択画面に戻り、Component configを選択します
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1632560090/blog/01fgdc0bawyb6d34gs54vxgpg9/lyzm4a4o3nijun4scgrw.png)

下の方にAmaozn Web Services IoT Platformがあるので、選択します
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1632560094/blog/01fgdc0bawyb6d34gs54vxgpg9/vjk8k7zwnidku58ormet.png)

急に画面が変わりますが、AWSマネジメントコンソール > AWS IoT > 設定 からエンドポイントをクリップボードに保存します
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1632560102/blog/01fgdc0bawyb6d34gs54vxgpg9/fdje2n4t14krgr31wgvj.png)

TUIの画面に戻ります。
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1632560181/blog/01fgdc0bawyb6d34gs54vxgpg9/vunbie76gmw0rri3gncx.png)

先ほどコピーしたエンドポイントを、そのまま貼り付けて、Enterを押下
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1632560287/blog/01fgdc0bawyb6d34gs54vxgpg9/pxycaclzrcyuunsg8x1d.png)

完了したら、`s`を押して`sdkconfig`に保存します
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1632567913/blog/01fgdc0bawyb6d34gs54vxgpg9/by3q0vlw0ebwnxjtiw5s.png)

ESCを数回(3回)押して、TUIから抜ければ`sdkconfig`の設定は完了です。本項の設定で、新しく下記のファイルが生成されます。sdkconfigはwifiのパスワードや環境のエンドポイントが入っています。かならずgitignoreするようにしましょう。
* CMakeLists.txt
* sdkconfig

# ファームの書き込みと実行
```bash
pio run --upload-port /dev/cu.SLAB_USBtoUART -t upload
pio device monitor -p /dev/cu.SLAB_USBtoUART
```

AWS IoTのMQTTテストクライアントを使って`test_topic/esp32`をサブスクライブしてみると、M5Stack Core2から定期的に電文がpublishされていることが確認できます。
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1632561699/blog/01fgdc0bawyb6d34gs54vxgpg9/i0gkrohfmofplqu69tjc.gif)

:::message warning
シリアルモニターの出力の`␛[0;32mI`が気になる場合、`--raw`オプションをつける解決します

```bash
pio device monitor -p /dev/cu.SLAB_USBtoUART --raw
```

:::


# 補足
## ccls定義の出力
Neovimとcoc.nvimで開発しているため、補完や定義ジャンプをするには、`.ccls`定義が必要となります。下記の手順で出力することが出来ます。

`target_add_binary_data` の箇所をコメントアウト

```bash:src/CMakeLists.txt
# This file was automatically generated for projects
# without default 'CMakeLists.txt' file.

FILE(GLOB_RECURSE app_sources ${CMAKE_SOURCE_DIR}/src/*.*)

set(COMPONENT_SRCS "main.c")
set(COMPONENT_ADD_INCLUDEDIRS ".")

register_component()

if(CONFIG_EXAMPLE_EMBEDDED_CERTS)
	# target_add_binary_data(${COMPONENT_TARGET} "certs/aws-root-ca.pem" TEXT)
	# target_add_binary_data(${COMPONENT_TARGET} "certs/certificate.pem.crt" TEXT)
	# target_add_binary_data(${COMPONENT_TARGET} "certs/private.pem.key" TEXT)
endif()
```

下記のコマンドを実行すると`.ccls`定義が出力されます。
```bash
pio init --board=m5stack-core2 --project-option="framework=espidf" --ide vim
```




# 最後に
ESP32環境で、比較的ミニマムなコードでMQTT通信を行うサンプルを試しました。`esp-aws-iot`の魅力は、[Using an ATECC608A with the ESP-AWS-IoT](https://github.com/espressif/esp-aws-iot/tree/249c573cbdcf653c03a6bf3a30f658f005856880#using-an-atecc608a-with-the-esp-aws-iot)にあるように、secure elementをサポートしている点です。Edukitだとソースが膨大すぎて、追うのが大変でした。今回のようにミニマムな実装を繰り返していくことで、secure elemetnサポート周りの理解が進められたらなぁと思います。

今回サンプルのCコードはまだ1ミリも理解していないので、まずはそれが先ですね...


# 参考

[ESP32からAWS-IoTにつなぐ（Espressif社のSDKを利用）](https://qiita.com/hisashij/items/f0b5d0d53ed70aa59b12)

