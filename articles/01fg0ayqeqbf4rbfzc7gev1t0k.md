---
title: "ESP-IDFの導入と使い心地を試す"
description: "M5Stack Core2のファームをESP32で開発したいので、ESP-IDFを導入した"
publish: true
tags:
  - "tech/iot"
  - "tech/arduino"
  - "tech/開発環境"
---

# はじめに

ESP32でファームを作成する場合、ESP32の開発元であるEspressif(エスプレッシフ社)が提供しているESP-IDFという純正統合開発ツールがあり、コマンドラインで開発が出来そうだったので試してみました。

先に結論として、ESP-IDFだと下記のような事象がありました。
* EdukitのGetting-Startedでビルド後の書き込みでファームにバグ
* [私が試したサンプル](https://github.com/usedbytes/m5core2-basic-idf)ではsdkconfig周りで、ビルドエラー

このような事象は、PlatformIO Core (CLI)のESP-IDF向けビルドだと起こらなかったです。素人並の感想なので、なにか決定的な設定忘れの可能性はありますが、とりあえずPlatformIO Core (CLI)ベース開発を続けることになりそうです。

最近環境構築記事ばかりで、組み込み周りは選択肢が意外と多いことを実感しています。。。

# ESP-IDFとは

[ESP-IDF Programming Guide](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/index.html)が非常に充実しています。

> Espressifによるソフトウェア開発フレームワークは、Wi-Fi、Bluetooth、電力管理、およびその他のいくつかのシステム機能を備えたモノのインターネット（IoT）アプリケーションの開発を目的としています。


# ESP-IDFの導入

[Installation Step by Step](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/get-started/index.html#installation-step-by-step)をやってみました。

## Step 1. Windows、Linux、またはmacOSの前提条件をインストールする

```bash
brew install cmake ninja dfu-util
```

ビルドを高速化するために、ccacheもインストールが強く推奨されていたため導入します
```bash
brew install ccache
```

Python導入手順については割愛

### Step 2. ESP-IDFの入手

```bash
mkdir -p ~/esp
cd ~/esp
git clone -b v4.3.1 --recursive https://github.com/espressif/esp-idf.git
```

:::message warning
EdukitをESP-IDFでビルドしたところ、アプリがバグる事象が発生したため、4.2にダウングレードした方法を書く

```bash
$ cd ~/esp
$ rm -rf esp-idf
$ git clone -b v4.2 https://github.com/espressif/esp-idf.git
$ cd ~/esp/esp-idf
$ ./install.sh
# => 初回よくみたら、Pythonパッケージが追加されておらずエラーになっていた。新しいzshを立ち上げ再度実行したところ解消
$ idf.py --version
ESP-IDF v4.2
```
結局バージョンを下げても解消せず。。。PlatformIOは内部でどのバージョンのESP-IDFを利用しているのだろうか..
:::


## Step 3. ツールのセットアップ

```bash
cd ~/esp/esp-idf
./install.sh
```

## Step 4. 環境変数のセットアップ

```bash
. $HOME/esp/esp-idf/export.sh
```

:::message warning
zshやbashにcdコマンド実行後、exaを実行する設定がされており、正しくexport.sh機能しませんでした。
設定をコメントアウトすることで、解決。

```bash: ~/.zshrcのcd後にexaを実行する設定をコメントアウト
# exec ls(exa) after cd
# chpwd() { exa -ahl --git }
```

:::

よく使う場合はエイリアスに登録します

```bash:~/.zshrc
alias get_idf='. $HOME/esp/esp-idf/export.sh'
```

## Step 5. プロジェクトの開始
Step 4が実行されていないと、IDF_PATHが設定されないためうまく実行されないので注意です

```bash
cd ~/esp
cp -r $IDF_PATH/examples/get-started/hello_world .
```

## Step 6. デバイスへの接続
macOSなので、starting with `/dev/cu.xxx`になります

## Step 7. コンフィグ

```bash
cd ~/esp/hello_world
idf.py set-target esp32
idf.py menuconfig
```

## Step 8. プロジェクトのビルド

```bash
idf.py build
```

## Step 9. デバイスへ書き込み

```bash
idf.py -p /dev/cu.SLAB_USBtoUART flash
```

## Step 10. 監視

```bash
idf.py -p /dev/cu.SLAB_USBtoUART monitor
```

`Ctr+]`で停止


次のコマンドを実行することで、ビルド、フラッシュ、モニタリングを1つのステップに組み合わせることができます。

```bash
idf.py -p /dev/cu.SLAB_USBtoUART flash monitor
```

ビルドはキャッシュされるので、ビルドが通らないときはcleanを試してみるのもよいかもしれません
```bash
idf.py clean
```

# プロジェクト作成手順

`ESP-IDF`を導入したので、改めてプロジェクト作成手順を整理します

## idf.pyの初期化
`Step 4`で設定した`get_idf`コマンドを実行します

```bash
get_idf
```

## プロジェクトの初期化

```bash
mkdir m5stack-espidf-practice
cd m5stack-espidf-practice
idf.py create-project .
```

```bash
$ tree
.
├── CMakeLists.txt
└── main
    ├── ..c
    └── CMakeLists.txt
```

## ソース、設定ファイルの変更
とりあえずシリアルモニターからHello Worldが見れるように最低限の準備を整えます

```bash: ..c -> main.cへリネーム
mv main/..c main/main.c
```

```diff: main.c
 void app_main(void)
 {
-
+  printf("Hello World");
 }
```

```diff: CMakeLists.txt
-project(.)
+project(m5stack-espidf-practice)
```
※ 本設定がないとビルドが落ちる(理由は不明)

```diff: main/CMakeLists.txt
-idf_component_register(SRCS "..c"
+idf_component_register(SRCS "main.c"
                     INCLUDE_DIRS ".")
```

```bash
idf.py clean
idf.py -p /dev/cu.SLAB_USBtoUART flash monitor
```

出力されました
```bash:実行結果
I (251) heap_init: At 3FFAE6E0 len 00001920 (6 KiB): DRAM
I (257) heap_init: At 3FFB31B8 len 0002CE48 (179 KiB): DRAM
I (264) heap_init: At 3FFE0440 len 00003AE0 (14 KiB): D/IRAM
I (270) heap_init: At 3FFE4350 len 0001BCB0 (111 KiB): D/IRAM
I (276) heap_init: At 4008AC10 len 000153F0 (84 KiB): IRAM
I (284) spi_flash: detected chip: generic
I (287) spi_flash: flash io: dio
W (291) spi_flash: Detected size(16384k) larger than the size in the binary image header(2048k). Using the size in the binary image header.
I (305) cpu_start: Starting scheduler on PRO CPU.
I (0) cpu_start: Starting scheduler on APP CPU.
Hello World
```

## その他必要ファイルを設定

```bash:Makefile
clean:
	idf.py clean
build:
	idf.py build
watch: clean
	idf.py -p /dev/cu.SLAB_USBtoUART flash monitor
```

```bash:.gitignore
.cache
build
```

# 少し複雑なサンプルを動かしてみる
## ライブラリの追加

基本的にgit submoduleを使うことになる

[github.com/usedbytes/m5core2-basic-idf](https://github.com/usedbytes/m5core2-basic-idf)を参考にライブラリを追加します。
明らかに個人っぽいリポジトリは、削除される可能性を考慮して、私の環境にForkしたものを利用します。

lvgl_esp32_driversは、[リビジョン6191d4](https://github.com/shuntaka9576/esp_i2c_helper/tree/6191d4a56a4297e0fd9e0d2d1a7341aff85f9378)を使いたかったが、どのブランチにも存在しないコミットでした。Forkして、HEADを6191d4と同等に書き換えました。

submodule addして、それぞれのリビジョンを調整します
```bash
git submodule add https://github.com/lvgl/lvgl.git components/lvgl
git submodule add https://github.com/shuntaka9576/axp192 components/axp192
git submodule add https://github.com/shuntaka9576/lvgl_esp32_drivers components/lvgl_esp32_drivers
git submodule add https://github.com/shuntaka9576/esp_i2c_helper components/esp_i2c_hal

# lvgl_esp32_drivers
git reset --hard 98b177a

# axp192
git reset --hard 99acd4a

# lvgl
git reset --hard f0c52b3
```

::: message warning
gitmodulesで失敗した場合
* .gitmodulesから対象のモジュール情報を削除する
* .git/configのモジュール情報を削除する
* componentsディレクトリを削除
* .git/modules/以下にあるsubmoduleでとってきたリポジトリを削除する
:::


## ソースの追加

cコードは、[こちら](https://github.com/shuntaka9576/m5core2-basic-idf/blob/f83bb95239e424c184572390c4eceb06d3705afd/main/main.c)を参考
```bash:main/main.c
$ make watch
(中略)
E (1604) i2c: i2c_master_cmd_begin(1166): i2c driver not installed
(中略)
```

[github.com/usedbytes/m5core2-basic-idf](https://github.com/usedbytes/m5core2-basic-idf)のsdkconfigをコピー貼り付けて実行したら、うまくいった


## もやもや

`sdkconfig`を削除して、下記の`platformio.ini`を追加して`pio run`と`pio --target upload`で安定して動くことを確認出来た。これが冒頭に述べた結論につながります。

```bash:platformio.ini
; PlatformIO Project Configuration File
;
;   Build options: build flags, source filter, extra scripting
;   Upload options: custom port, speed and extra flags
;   Library options: dependencies, extra library storages
;
; Please visit documentation for the other options and examples
; http://docs.platformio.org/page/projectconf.html

[platformio]
src_dir = main

[env:esp32dev]
platform = espressif32
framework = espidf
board = esp32dev

monitor_speed = 115200
monitor_filters = colorize
monitor_flags= --raw
upload_port = /dev/cu.SLAB_USBtoUART
```

# 最後に
そもそもですが、ArudiunoとESP-IDFのプロジェクト構成を比較すると、前者は`pio lib install`で依存を落とせましたが、ESP-IDFはgit submoduleを駆使しないといけないんですかね..。

それはさておき、当面の目標としてArdiunoに慣れて、ESP-IDF向けのサンプルを量産したいですね
