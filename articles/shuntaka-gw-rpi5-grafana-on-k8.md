---
title: "自作基盤メモ(内部用途のためブログにする場合別記事へ)"
type: "note"
description: ""
thumbnail: ""
publish: false
---


![img](https://devio2023-media.developers.io/wp-content/uploads/2024/05/1714533363-640x336.png)


## はじめに
世の中ではGrafana Weekということで、Raspberry Pi 5複数台をクラスタリングしてKubernetesを作成し、Grafanaを載せてみたいと思います。

というのは冗談ですが、最近趣味で安価に常駐プロセスをデプロイできるホスティング環境に悩んでいました。常駐しないなら最近はゼロコールドスタートなV8 Isolateを使ったCloudflare WorkersやDeno Deployが無料枠が大きくいい感じです。
一方常駐プロセスはHerokuの無料プランがなくなりました。AWS AppRunnerは起動時間を人間が稼働している時間のみに絞っても10$はかかります。fly.ioは、Legacy hobby planでCPU-1x 256mb VM 3つと3 GB 永続ボリュームストレージは無料で扱えます。fly.ioはCLIもよくできているので、軽い検証の場合こちらで良さそうです。

より永続化領域とメモリが潤沢な環境を求めて、自前でコンテナオーケストレーション基盤を作ることにしました。

Kubernetesクラスタ自体は4年前くらいにRPi3で作ったことがあります。エコシステム的な難しさもあり、早々に飽きてしまったので今回はそこそこ真面目に構築しました。実務的で使ったことはないので、経験者からすると拙いのとN番煎じですが、ご参考にして頂けたら幸いです。

以下の点ご理解お願いします。

* クラウドでサーバレス開発をしているので、電気やハード的なところはあまり参考にならないと思います
* 本構成は動作を保証しません。特に電気的な部分は、他の記事と合わせて確認するのが良いと思います

以後Raspberry Pi 5は、RPi5と表記します。


## 構成

### ゴール

Mac(Kubernetesクラスタ外)から`kubectl port-forward`なしで、Grafanaが閲覧できるようになることを目指します。こんな形です。


![img](https://devio2023-media.developers.io/wp-content/uploads/2024/05/1714537552-2.gif)


### 注意点

本項以降の各手順は、最小2台で進めることを推奨します。理由は、一気に4台でやるよりは、2台でgrafana起動まで確認し、その後2ノード追加する方が何か手戻りがあった際にやり直しがしやすいためです。

### 前提

* PoEは利用しない(RPi5ではまだPoE HATがないため)
* RPi5のストレージには NVMe M.2 SSDを利用
  * microSDで代用可能です、RPi5側で対応しているのと家に余っていたので活用しました
* RPi5のイメージには、Ubuntu Server 24.04 LTS(64bit)を利用
* 電源はスイッチング電源2台で、各スイッチング電源に2つのRPi5を接続


|項|名称|個数|補足|リンク|
|---|---|---|---|---|
|1|Raspberry Pi 5 8GB|4|RPi5本体|[リンク](https://www.switch-science.com/products/9250)|
|2|Raspberry Pi 5用アクティブクーラー|4|RPi5のクーラー|[リンク](https://www.switch-science.com/products/9253)|
|3|Geekworm X1001 V1.1 PCIe NVMe M.2 拡張ボード|4|RPi5とM.2.を接続するための拡張ボード|[リンク](https://amzn.asia/d/0CfLay6)|
|4|KIOXIA EXCERIA G2 SSD-CK1.0N3G2/N 1TB|4|M.2. M.2 NVMe SSD。正確には家に余っている別メーカーのものもありますが大きな差はないので割愛。|[リンク](https://amzn.asia/d/4OSW8eP)|
|5|ORICO M.2 SSD 外付けケース|1|NVMe SSDへイメージを書き込むための外付けケース|[リンク](https://amzn.asia/d/eHkpa4V)|
|6|スイッチング電源 DC5V10A 出力50Wタイプ RWS50B-5|2|RPi5は5V/5Aなので、1つあたりRPi52台接続想定。GPIO 5Vピン2つから電源供給|[リンク](https://akizukidenshi.com/catalog/g/g109086/)
|7|Qiコネクタ|1|スイッチング電源からGPIO経由で電源供給するため利用|[リンク](https://akizukidenshi.com/catalog/g/g109733/)
|8|コネクタ用ハウジング|4|上記のQiコネクトを半分に切って、ケーブル4本を1つのハウジングに納めて計4本作ります|[リンク](https://akizukidenshi.com/catalog/g/g112437/)
|9|TP-Link スイッチングハブ 5ポート PoE+|1|PoE+は利用しません|[リンク](https://amzn.asia/d/2owPNQR)|
|10|Type C ケーブル 0.5M 2本組|2|2本組を2つ購入。計4本|[リンク](https://amzn.asia/d/491DZzx)|
|11|LANケーブル 0.3m 5本|4|5本組を1つ購入。計5本|[リンク](https://amzn.asia/d/8mHTViH)|
|12|microSD 32GB|1|ブート設定を変更するためだけに利用します。恒常的に利用しません。|なし|
|13|microSD読めるカードリーダー|1||なし|
|14|キッチンラック|1|ラズパイを積む用のラック。専用のものは高かったので、キッチンラックで代用しました...RPi5の裏面の導通ある部分がメタルラックに接触するのがリスクなので紙の上にRPi5を載せています|[リンク](https://amzn.asia/d/eGq23dw)|
|15|片プラグ付耐熱コード|2|丸端子圧着済みなので、スイッチング電源に接続しやすいです|[リンク](https://amzn.asia/d/av3Cy42)|



## ブート設定

### はじめに

### SDカードにRasberry Pi OSを書き込む

書き込みには、Rasberry Pi Imagerを利用します。ダウンロードは[こちら](https://www.raspberrypi.com/software/)から可能です。私はWindows版を利用しました。

一時的にしか使わないので、OSはRASBERY PI OS LITE(64-BIT)を利用します

![img](https://devio2023-media.developers.io/wp-content/uploads/2024/05/1713571189-640x403.jpg)

設定の編集をします

![img](https://devio2023-media.developers.io/wp-content/uploads/2024/05/1713571296-640x403.jpg)

ユーザーとWi-Fi設定を入れたら、サービスタブに移動します
![img](https://devio2023-media.developers.io/wp-content/uploads/2024/05/1713571468-640x653.jpg)


SSHを有効化します。書き込みの段階で指定できるの便利ですね！

![img](https://devio2023-media.developers.io/wp-content/uploads/2024/05/1713571545-640x653.jpg)

書き込み開始します

![img](https://devio2023-media.developers.io/wp-content/uploads/2024/05/1713572144-1-640x403.jpg)

書き込み完了まで待ちます。

![img](https://devio2023-media.developers.io/wp-content/uploads/2024/05/1713572144-640x403.jpg)


### M.2 SSDにUbuntu Serverを書き込む

先に恒常的に利用するM.2にもOSを書き込みます。6の外付けケースを利用して、PCに画像のような形で接続します。
![img](https://devio2023-media.developers.io/wp-content/uploads/2024/05/1713572773-640x686.jpg)

OSは`Ubuntu Server 24.04 LTS`を利用します
![img](https://devio2023-media.developers.io/wp-content/uploads/2024/05/1714522805-640x444.jpg)

![img](https://devio2023-media.developers.io/wp-content/uploads/2024/05/1713572906-640x654.jpg)

![img](https://devio2023-media.developers.io/wp-content/uploads/2024/05/1713572937-640x654.jpg)


初回はイメージダウンロードの時間がかかるため、通信環境によっては時間がかかります。一度内部でイメージをダウンロードすれば、2回目以降は数分もかからず終わります。

![img](https://devio2023-media.developers.io/wp-content/uploads/2024/05/1714522805-1-640x444.jpg)

書き込みが終わったら、再度M.2をPCにマウントします。すると画像のようにブートパーティションを認識します。

![img](https://devio2023-media.developers.io/wp-content/uploads/2024/05/1713584023-640x482.jpg)

ブートパーティションを開いたら、config.txtを見つけます。

![img](https://devio2023-media.developers.io/wp-content/uploads/2024/05/1713584235-640x426.jpg)


config.txtに2行追記します
```diff
[all]
+dtparam=nvme
+dtparam=pciex1
```

### RPi5にクーラーを取り付け

![img](https://devio2023-media.developers.io/wp-content/uploads/2024/05/1713573425-640x480.jpg)
![img](https://devio2023-media.developers.io/wp-content/uploads/2024/05/1713758751-640x395.jpg)


### RPi5 M.2拡張
PCIe NVME M.2 拡張ボードを取り付けます

![img](https://devio2023-media.developers.io/wp-content/uploads/2024/05/1713758843-640x471.jpg)

![img](https://devio2023-media.developers.io/wp-content/uploads/2024/05/1714534398-640x457.jpg)

裏は家にあったネジで高さを出すようにしました。

![img](https://devio2023-media.developers.io/wp-content/uploads/2024/05/1713758580-640x410.jpg)


### 電源の設定

電源はRPi5の2,4ピンに+5V、6,9ピンにGNDに繋ぎます。スイッチング電源1つに2つのRPi5を接続します。以下の図が分かりやすいです。下の直方体はコネクタのハウジングを示してます。


![img](https://devio2023-media.developers.io/wp-content/uploads/2024/05/1714538385.png)

※ 弊社のこの領域に強い方に教えて頂きました。ありがとうございます;;

実際はこんな形です。

![img](https://devio2023-media.developers.io/wp-content/uploads/2024/05/1714531320-640x853.jpg)

### ブート設定(EEPROM書き込み)

microSDを入れて、RPi5の電源を入れます。ホスト名は`raspberrypi.local`です。

```bash
$ ssh shuntaka@raspberrypi.local
(中略)
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
(中略)
Linux raspberrypi 6.6.20+rpt-rpi-2712 #1 SMP PREEMPT Debian 1:6.6.20-1+rpt1 (2024-03-07) aarch64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
shuntaka@raspberrypi:~ $
```

ブートローダーを更新します。

```bash
shuntaka@raspberrypi:~ $ sudo rpi-eeprom-update
*** UPDATE AVAILABLE ***
BOOTLOADER: update available
   CURRENT: Wed Dec  6 18:29:25 UTC 2023 (1701887365)
    LATEST: Fri Feb 16 15:28:41 UTC 2024 (1708097321)
   RELEASE: default (/lib/firmware/raspberrypi/bootloader-2712/default)
            Use raspi-config to change the release.
```

利用可能な更新がありそうなので、更新を実行します。

```bash
shuntaka@raspberrypi:~ $ sudo rpi-eeprom-update -d -a
*** PREPARING EEPROM UPDATES ***

BOOTLOADER: update available
   CURRENT: Wed Dec  6 18:29:25 UTC 2023 (1701887365)
    LATEST: Fri Feb 16 15:28:41 UTC 2024 (1708097321)
   RELEASE: default (/lib/firmware/raspberrypi/bootloader-2712/default)
            Use raspi-config to change the release.
   CURRENT: Wed Dec  6 18:29:25 UTC 2023 (1701887365)
    UPDATE: Fri Feb 16 15:28:41 UTC 2024 (1708097321)
    BOOTFS: /boot/firmware
'/tmp/tmp.ov5cNFr05e' -> '/boot/firmware/pieeprom.upd'

UPDATING bootloader.

*** WARNING: Do not disconnect the power until the update is complete ***
If a problem occurs then the Raspberry Pi Imager may be used to create
a bootloader rescue SD card image which restores the default bootloader image.

flashrom -p linux_spi:dev=/dev/spidev10.0,spispeed=16000 -w /boot/firmware/pieeprom.upd
UPDATE SUCCESSFUL
```


変わっていないので、再起動する必要がありそう
```bash
shuntaka@raspberrypi:~ $ sudo rpi-eeprom-update
*** UPDATE AVAILABLE ***
BOOTLOADER: update available
   CURRENT: Wed Dec  6 18:29:25 UTC 2023 (1701887365)
    LATEST: Fri Feb 16 15:28:41 UTC 2024 (1708097321)
   RELEASE: default (/lib/firmware/raspberrypi/bootloader-2712/default)
            Use raspi-config to change the release.
```

`sudo reboot` 後再度実行したところ更新完了していることを確認

```bash
shuntaka@raspberrypi:~ $ sudo rpi-eeprom-update
BOOTLOADER: up to date
   CURRENT: Fri Feb 16 15:28:41 UTC 2024 (1708097321)
    LATEST: Fri Feb 16 15:28:41 UTC 2024 (1708097321)
   RELEASE: default (/lib/firmware/raspberrypi/bootloader-2712/default)
            Use raspi-config to change the release.
```


ブート設定を変更します。極度のnanoアレルギーなのでviを設定します。

```bash
env EDITOR=vi sudo -E rpi-eeprom-config --edit
```

```bash:変更前
[all]
BOOT_UART=1
POWER_OFF_ON_HALT=0
BOOT_ORDER=0xf461
```


以下のように設定変更します
```diff:変更後(diff)
[all]
BOOT_UART=1
POWER_OFF_ON_HALT=0
-BOOT_ORDER=0xf461
+BOOT_ORDER=0xf416
+PCIE_PROBE=1
```

```bash:実行結果
shuntaka@raspberrypi:~ $ env EDITOR=vi sudo -E rpi-eeprom-config --edit
Updating bootloader EEPROM
 image: /lib/firmware/raspberrypi/bootloader-2712/default/pieeprom-2024-02-16.bin
config_src: blconfig device
config: /tmp/tmpp6gugkfa/boot.conf
################################################################################
[all]
BOOT_UART=1
POWER_OFF_ON_HALT=0
BOOT_ORDER=0xf416
PCIE_PROBE=1

################################################################################

*** To cancel this update run 'sudo rpi-eeprom-update -r' ***

*** CREATED UPDATE /tmp/tmpp6gugkfa/pieeprom.upd  ***

   CURRENT: Fri 16 Feb 15:28:41 UTC 2024 (1708097321)
    UPDATE: Fri 16 Feb 15:28:41 UTC 2024 (1708097321)
    BOOTFS: /boot/firmware
'/tmp/tmp.53DnGVNp9Z' -> '/boot/firmware/pieeprom.upd'

UPDATING bootloader.

*** WARNING: Do not disconnect the power until the update is complete ***
If a problem occurs then the Raspberry Pi Imager may be used to create
a bootloader rescue SD card image which restores the default bootloader image.

flashrom -p linux_spi:dev=/dev/spidev10.0,spispeed=16000 -w /boot/firmware/pieeprom.upd
UPDATE SUCCESSFUL
```

### M.2からブート

一度RPi5の電源をきり、**microSDを抜いて**、M.2を挿します。Ubuntu 24.04 LTSの文字があるので、M.2から起動していることが確認できました。

```bash
$ ssh shuntaka@pi1.local
shuntaka@pi1.local's password:
Welcome to Ubuntu 24.04 LTS (GNU/Linux 6.8.0-1004-raspi aarch64)
[/bash]

[bash]
shuntaka@pi4:~$ sudo df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           795M  3.2M  792M   1% /run
/dev/nvme0n1p2  917G  1.9G  878G   1% /
tmpfs           3.9G     0  3.9G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
/dev/nvme0n1p1  505M   85M  420M  17% /boot/firmware
tmpfs           795M   12K  795M   1% /run/user/1000
```

### カーネルの更新

`Ubuntu Server 24.04 LTS` では更新はありません。

```bash
$ uname -r
6.8.0-1004-raspi
```

```bash
sudo apt update
sudo apt -y upgrade
```

```bash
$ uname -r
6.8.0-1004-raspi
```

`Ubuntu Server 23.10` の場合、クーラーが回りっぱなしで音が大きいので確実にやるのが良いです。
[参考](https://askubuntu.com/questions/1490462/fan-speed-control-on-raspberry-pi-5-not-working-in-ubuntu-23-10)


## ネットワーク設定

### はじめに

以下のような形でアドレスを固定します。

|ホスト名|eth0 IP|
|---|---|
|pi1|192.168.1.1<br>192.168.86.10
|pi2|192.168.1.2|
|pi3|192.168.1.3|
|pi4|192.168.1.4|


### p1のnetplanの設定

```bash
cat <<EOF | sudo tee /etc/netplan/01-netcfg.yaml > /dev/null
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 192.168.1.1/24
        - 192.168.86.10/24
      nameservers:
        addresses: [8.8.8.8]
EOF
```

```bash
sudo netplan apply
```

### pi2以降のnetplanの設定

pi2~pi4のnetplanの設定をします

```bash
cat <<EOF | sudo tee /etc/netplan/01-netcfg.yaml > /dev/null
network:
    version: 2
    ethernets:
      eth0:
        dhcp4: false
        addresses:
          - 192.168.1.2/24
          # pi3の場合
          # - 192.168.1.3/24
          # pi4の場合
          # - 192.168.1.4/24
        nameservers:
          addresses:
            - 8.8.8.8
EOF
```

```bash
sudo netplan apply
```

`192.168.1.2`が割り当てられていることを確認
```bash
shuntaka@pi2:~$ ip a
(中略)
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
(中略)
    inet 192.168.1.2/24 brd 192.168.1.255 scope global eth0
       valid_lft forever preferred_lft forever
(中略)
```

## Kubernetesクラスタの構築

### kubelet kubeadm kubectlのインストール

モジュールのロード設定を実施します

```bash:全てのノード
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

読み込まれているか確認
```bash:全てのノード
lsmod | grep br_netfilter
lsmod | grep overlay
```

kubelet kubeadm kubectlをインストールします

```bash:全てのノード
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl # <--最初は apt-getでやっていたの確認

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### cri-oのインストール

[download.opensuse.org](http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/)から要件にあったものを選択

```bash:全てのノード
export OS=xUbuntu_22.04

sudo echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /" | sudo tee -a /etc/apt/sources.list.d/cri-o-stable.list >/dev/null
sudo echo "deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/1.28/$OS/ /" | sudo tee -a /etc/apt/sources.list.d/cri-o.list >/dev/null
 
sudo curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/libcontainers-crio-archive-keyring.gpg
sudo curl -L http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/1.28/$OS/Release.key | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/cri-o-apt-keyring.gpg
 
sudo apt update
sudo apt install cri-o cri-o-runc -y

sudo rm -rf /etc/cni/net.d/*

sudo systemctl daemon-reload
sudo systemctl enable crio
sudo systemctl start crio
```


cri-oの動作確認
```bash:全てのノード
shuntaka@pi1:~$ sudo crictl info
{
  "status": {
    "conditions": [
      {
        "type": "RuntimeReady",
        "status": true,
        "reason": "",
        "message": ""
      },
      {
        "type": "NetworkReady",
        "status": false,
        "reason": "NetworkPluginNotReady",
        "message": "Network plugin returns error: No CNI configuration file in /etc/cni/net.d/. Has your network provider started?"
      }
    ]
  },
  "config": {
    "sandboxImage": "registry.k8s.io/pause:3.9"
  }
```

Kubernetesクラスタの初期化を実行します。

```bash:マスターノード
$ sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --control-plane-endpoint=192.168.1.1 \
  --apiserver-advertise-address=192.168.1.1 \
  --apiserver-cert-extra-sans=192.168.1.1
```

ワーカーノード側で実行する必要があるコマンドをメモします。
```bash:メモする出力結果
sudo kubeadm join 192.168.1.1:6443 --token gdfhzr.gv99gncmeax7lqyi \
        --discovery-token-ca-cert-hash sha256:8714a21a302f13ed2b400e34386f66f8bc5094e348941f2a80805674a21fd8a0
```

`kubectl` が利用できるように、設定を行います

```bash:マスターノード
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Kubernetesクラスタに参加するコマンドを実行します

```bash:ワーカーノード
sudo kubeadm join 192.168.1.1:6443 --token rwmoll.7vfynzwt0wbparok \
        --discovery-token-ca-cert-hash sha256:bf28ffd7b92f505b93409da63dacc610d05e4d3566e50b629738a6a4cf4b259f
```

ノードを確認します。この段階ではNotReadyですが、CNIをインストールするとReady状態になります
```bash:マスターノード
$ kubectl get node
NAME   STATUS     ROLES           AGE     VERSION
pi1    NotReady   control-plane   9m21s   v1.28.9
pi2    NotReady   <none>          9s      v1.28.9
```

### CNI(calico)のインストール

手順と異なり、Ubuntu Server 23.10の場合、以下の設定が必要です
```bash:全てのノード
# Ubuntu Server 23.10利用の場合のみ
sudo apt install linux-modules-extra-raspi -y
```

CNIをインストールします

```bash:マスターノード
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.3/manifests/tigera-operator.yaml
curl https://raw.githubusercontent.com/projectcalico/calico/v3.27.3/manifests/custom-resources.yaml -O

# custom-resources.yamlのcidr: 10.244.0.0/16 に修正
kubectl create -f custom-resources.yaml
kubectl get pods -n calico-system
```

CNIがインストールされ、全てReadyになりました
```bash
$ kubectl get nodes
NAME   STATUS   ROLES           AGE     VERSION
pi1    Ready    control-plane   17m     v1.28.9
pi2    Ready    <none>          8m16s   v1.28.9
```


### エイリアス設定

使いやすいように、エイリアスと補完設定を入れます。この設定で省略された`k`コマンドと`tab`での補完が効くようになります。

```bash
cat <<EOF >> ~/.bashrc
# k8s
source <(kubectl completion bash)
alias k=kubectl
complete -o default -F __start_kubectl k
EOF
```


## Helmの設定

### はじめに

今回KubernetesリソースはHelm経由でデプロイすることにします。

### Helmのインストール

```bash:マスターノード
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt update
sudo apt install helm
```

### Helmfileのインストール

```bash:マスターノード
export VERSION="0.163.1"
wget -O "/tmp/helmfile_${VERSION}_linux_arm64.tar.gz" "https://github.com/helmfile/helmfile/releases/download/v${VERSION}/helmfile_${VERSION}_linux_arm64.tar.gz"
tar -xzvf "/tmp/helmfile_${VERSION}_linux_arm64.tar.gz" -C /tmp

sudo mv /tmp/helmfile /usr/bin/helmfile
sudo chmod 755 /usr/bin/helmfile
```


## MetalLBの設定

### はじめに

MetalLBは、ベアメタルKubernetes環境向けのロードバランサーでKubernetesクラスタ外からアクセス出来るようにIPを割り当てます。

### IPアドレスプールの定義

割り当てるIPプールを定義します。IPレンジはKubernetesの内部IPではなく、ルータで払い出されているものを利用してください。

```yaml:ip-address-pool.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default
  namespace: metallb
spec:
  addresses:
    - 192.168.86.200-192.168.86.250
  autoAssign: true
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb
spec:
  ipAddressPools:
  - default
```

適用します
```bash
kubectl apply -f ip-address-pool.yaml
```

### MetalLBのホスティング

[こちら](https://metallb.universe.tf/installation/) を参考に、以下のコマンドで、`strictARP`をtrueにします

```bash
kubectl edit configmap -n kube-system kube-proxy
```

```yml:helmfile.yaml
repositories:
  - name: metallb
    url: https://metallb.github.io/metallb

releases:
  - name: metallb
    namespace: metallb
    chart: metallb/metallb
    version: 0.14.5
    values:
      - values.yaml
```

実行自体はこけますが、repositoriesは追加されます。

```bash
helmfile sync
```

以下のコマンドで変更可能箇所を出力します。出力しますが変更箇所はありません。

```bash
helm show values metallb/metallb>values.yaml
```

適用します。
```bash
helmfile sync
```

## PromethuesとGrafanaの設定

### はじめに

* PromethuesとGrafana間で通信させるために同じnamespaceに所属
* MetalLBを使ってGrafanaをクラスタ外へ公開

### 永続ストレージ設定

PersistentVolumeとPersistentVolumeClaimの設定をします。`prometheus-alertmanager`は、ストレージは作っていますが、後述のprometheusの設定でOFFにするのでなくても大丈夫です。

| ストレージタイプ      | name                       | path                                          |
|---------------------  |----------------------------|-----------------------------------------------|
| PersistentVolume      | prometheus-server          | /mnt/k8s/pv/prometheus-server                 |
| PersistentVolume      | prometheus-alertmanager    | /mnt/k8s/pv/prometheus-alertmanager           |
| PersistentVolume      | grafana                    | /mnt/k8s/pv/grafana                           |

| ストレージタイプ      | namespace     | name                       |
|---------------------  |---------------|----------------------------|
| PersistentVolumeClaim | monitoring    | prometheus-server          |
| PersistentVolumeClaim | monitoring    | prometheus-alertmanager    |
| PersistentVolumeClaim | monitoring    | grafana                    |

[bash title="monitoringネームスペース作成"]
kubectl create namespace monitoring
[/bash]

pvとpvcはcontrol-planeにデフォルトでは設定できないため、pi2側で設定します

```bash:pi2で実行!!!
sudo mkdir -p /mnt/k8s/pv/grafana
sudo mkdir -p /mnt/k8s/pv/prometheus-server
sudo mkdir -p /mnt/k8s/pv/prometheus-alertmanager
```

pvとpvcを定義します
```yaml:volume.yaml
# grafana
kind: PersistentVolume
apiVersion: v1
metadata:
  name: grafana
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: grafana
  local:
    path: /mnt/k8s/pv/grafana
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - pi2
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: grafana
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: grafana
---

# prometheus-server
kind: PersistentVolume
apiVersion: v1
metadata:
  name: prometheus-server
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: prometheus-server
  local:
    path: /mnt/k8s/pv/prometheus-server
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - pi2
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: prometheus-server
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: prometheus-server
---

# prometheus-alertmanager
kind: PersistentVolume
apiVersion: v1
metadata:
  name: prometheus-alertmanager
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: prometheus-alertmanager
  local:
    path: /mnt/k8s/pv/prometheus-alertmanager
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - pi2
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: prometheus-alertmanager
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: prometheus-alertmanager
```

volumeを適用します
```bash
kubectl apply -f volume.yaml
```

表通り作成されていることを確認できたらOKです

```bash
kubectl get pv
kubectl get pvc -A
```

### Promethuesの設定

helmfileを書きます

```yml:helmfile.yaml
repositories:
  - name: prometheus-community
    url: https://prometheus-community.github.io/helm-charts

releases:
  - name: prometheus
    namespace: monitoring
    chart: prometheus-community/prometheus
    version: 25.9.0
    values:
      - values.yaml
```

実行自体はこけますが、repositoriesは追加されます

```bash
helmfile sync
```

以下のコマンドで変更可能箇所を出力します。出力しますが変更箇所はありません。
```bash
helm show values prometheus-community/prometheus > values.yaml
```

以下のように修正します
```yml:values.yaml
server:
  ## Prometheus server container name
  ##
  (中略)
  persistentVolume:
  (中略)
    existingClaim: "prometheus-server" # <- 先ほど作成したPVCを指定
  (中略)
    size: 10Gi # <- 8Giを10Giに変更
  (中略)
  service:
    (中略)
    type: NodePort # <- ClusterIPをNodePortへ変更
(中略)
alertmanager:
  ## If false, alertmanager will not be installed
  ##
  enabled: false # <- 今回は利用しないためtrueからfalseへ変更
```

適用します
```bash
helmfile sync
```

### Grafanaの設定

同様にhelmfileを書きます

```yml:helmfile.yaml
repositories:
  - name: grafana
    url: https://grafana.github.io/helm-charts

releases:
  - name: grafana
    namespace: monitoring
    chart: grafana/grafana
    version: 7.2.3
    values:
      - values.yaml
```

こちらも同様です。
```bash
helmfile sync
```

以下のコマンドで変更可能箇所を出力します。

```bash
helm show values grafana/grafana > values.yaml
```

以下のように修正します
```yaml:values.yaml
service:
  enabled: true
  type: NodePort # <- ClusterIPをNodePortへ変更
(中略)
persistence:
  type: pvc
  enabled: true # <- ストレージ永続化ため、falseからtrueへ変更
  (中略)
  existingClaim: "grafana" # <- 先ほど作成したPVCを指定
(中略)
# データソースを追加
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-server.monitoring.svc.cluster.local/
      isDefault: true
```

適用します
```bash
helmfile sync
```

以下のコマンド実行して、マスターノードのクラスタ外から見れるプライベートIP:3000で、PCからアクセスしてGrafanaのログイン画面が見れれば成功です。
```bash:マスターノード
kubectl port-forward --address 0.0.0.0 svc/grafana 3000:80 -n monitoring
```

ログインIDとパスワードは以下のコマンド出力できます

```bash
kubectl get secret grafana --namespace monitoring  -o jsonpath="{.data.admin-user}" | base64 --decode ; echo
kubectl get secret grafana --namespace monitoring -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

### Grafanaをクラスタ外から閲覧する

MetalLBを利用して外部アクセス可能なLoadBalancerタイプのサービスを定義します。宛先にgrafanaを指定します。

```yaml:grafana-loadbalancer.yaml
apiVersion: v1
kind: Service
metadata:
  name: grafana-loadbalancer
  namespace: monitoring
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 3000
      protocol: TCP
  selector:
    app.kubernetes.io/instance: grafana
    app.kubernetes.io/name: grafana
```

適用します

```bash
kubectl apply -f grafana-loadbalancer.yaml
```

`grafana-loadbalancer`に`192.168.86.200`が振られていることが確認できます。

```bash
$ k get svc -A
NAMESPACE          NAME                                  TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)                  AGE
calico-apiserver   calico-api                            ClusterIP      10.109.235.205   <none>           443/TCP                  5h2m
calico-system      calico-kube-controllers-metrics       ClusterIP      None             <none>           9094/TCP                 5h2m
calico-system      calico-typha                          ClusterIP      10.96.11.42      <none>           5473/TCP                 5h3m
default            kubernetes                            ClusterIP      10.96.0.1        <none>           443/TCP                  5h8m
kube-system        kube-dns                              ClusterIP      10.96.0.10       <none>           53/UDP,53/TCP,9153/TCP   5h8m
metallb            metallb-webhook-service               ClusterIP      10.98.53.232     <none>           443/TCP                  4h31m
monitoring         grafana                               NodePort       10.100.215.182   <none>           80:31851/TCP             4h35m
monitoring         grafana-loadbalancer                  LoadBalancer   10.98.112.253    192.168.86.200   80:30699/TCP             4h17m
monitoring         prometheus-kube-state-metrics         ClusterIP      10.110.198.118   <none>           8080/TCP                 4h34m
monitoring         prometheus-prometheus-node-exporter   ClusterIP      10.108.144.213   <none>           9100/TCP                 4h34m
monitoring         prometheus-prometheus-pushgateway     ClusterIP      10.107.27.149    <none>           9091/TCP                 4h34m
monitoring         prometheus-server                     NodePort       10.111.109.182   <none>           80:31648/TCP             4h34m
```

http://192.168.86.200にアクセスすると、http://192.168.86.200/loginにリダイレクトされて、grafanaのダッシュボードが閲覧できました


![img](https://devio2023-media.developers.io/wp-content/uploads/2024/05/1714536800-640x352.png)


Data sourcesを開きます

![img](https://devio2023-media.developers.io/wp-content/uploads/2024/05/1714536797-640x322.png)


DatasourceはHelmのvalue.yamlで定義しているため、Prometheusがすでにある状態です。

![img](https://devio2023-media.developers.io/wp-content/uploads/2024/05/1714536786-640x327.png)

![img](https://devio2023-media.developers.io/wp-content/uploads/2024/05/1714536792-640x324.png)


今回は、[こちら](https://grafana.com/grafana/dashboards/11074-node-exporter-for-prometheus-dashboard-en-v20201010/?tab=revisions)のダッシュボードを利用しますので、11074を入力してLoadします。

![img](https://devio2023-media.developers.io/wp-content/uploads/2024/05/1714536789-640x493.png)


Loadされたら、importします

![img](https://devio2023-media.developers.io/wp-content/uploads/2024/05/1714536785-640x400.png)


無事importできました。このときはまだ3つしかノードがないので、正常に動作しています!

![img](https://devio2023-media.developers.io/wp-content/uploads/2024/05/1714536794-640x303.png)


## ArgoCDの導入

本手順は特別記載があるところを除き、全てpi1上で実行します。

```yml
repositories:
  - name: argocd
    url: https://argoproj.github.io/argo-helm

releases:
  - name: argocd
    namespace: argocd
    chart: argocd/argo-cd
    version: 7.3.11
    values:
      - values.yaml
```

実行自体はこけますが、repositoriesは追加されます
```bash
helmfile sync
```

```bash
helm show values argocd/argo-cd>values.yaml
```

適用します
```bash
helmfile sync
```

```bash:出力結果
Adding repo argocd https://argoproj.github.io/argo-helm
"argocd" has been added to your repositories

Upgrading release=argocd, chart=argocd/argo-cd
Release "argocd" does not exist. Installing it now.
NAME: argocd
LAST DEPLOYED: Sat Jul 27 23:46:43 2024
NAMESPACE: argocd
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
In order to access the server UI you have the following options:

1. kubectl port-forward service/argocd-server -n argocd 8080:443

    and then open the browser on http://localhost:8080 and accept the certificate

2. enable ingress in the values file `server.ingress.enabled` and either
      - Add the annotation for ssl passthrough: https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-1-ssl-passthrough
      - Set the `configs.params."server.insecure"` in the values file and terminate SSL at your ingress: https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-2-multiple-ingress-objects-and-hosts


After reaching the UI the first time you can login with username: admin and the random password generated during the installation. You can find the password by running:

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

(You should delete the initial secret afterwards as suggested by the Getting Started Guide: https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli)

Listing releases matching ^argocd$
argocd  argocd          1               2024-07-27 23:46:43.682250172 +0000 UTC deployed        argo-cd-7.3.11  v2.11.7


UPDATED RELEASES:
NAME     CHART            VERSION   DURATION
argocd   argocd/argo-cd   7.3.11         41s
```

argocd関連のPodが動作していることを確認
```bash
$ k get pods -n argocd
NAME                                               READY   STATUS      RESTARTS   AGE
argocd-application-controller-0                    1/1     Running     0          4m11s
argocd-applicationset-controller-dc8cfcbdc-59q4m   1/1     Running     0          4m11s
argocd-dex-server-76cc78cfcf-7rkrd                 1/1     Running     0          4m11s
argocd-notifications-controller-59c46695f5-jsj2v   1/1     Running     0          4m11s
argocd-redis-5cf79bddc4-nl4jz                      1/1     Running     0          4m11s
argocd-redis-secret-init-fl2vk                     0/1     Completed   0          4m46s
argocd-repo-server-ccc55876c-pn92r                 1/1     Running     0          4m11s
argocd-server-77cbc679c-4v672                      1/1     Running     0          4m11s
```

```yaml:algocd-loadbalancer.yaml
apiVersion: v1
kind: Service
metadata:
  name: argocd-loadbalancer
  namespace: argocd
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
  selector:
    app.kubernetes.io/instance: argocd
    app.kubernetes.io/name: argocd-server
```

適用します

```bash
kubectl apply -f algocd-loadbalancer.yaml
```

argocd-serverに `192.168.86.202`がアサインされていることを確認します
```bash
$ k get svc -n argocd
NAME                               TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)             AGE
argocd-applicationset-controller   ClusterIP      10.110.94.104    <none>           7000/TCP            40m
argocd-dex-server                  ClusterIP      10.110.70.9      <none>           5556/TCP,5557/TCP   40m
argocd-loadbalancer                LoadBalancer   10.99.40.242     192.168.86.202   443:32605/TCP       10m
argocd-redis                       ClusterIP      10.100.69.83     <none>           6379/TCP            40m
argocd-repo-server                 ClusterIP      10.109.230.83    <none>           8081/TCP            40m
argocd-server                      ClusterIP      10.109.206.234   <none>           80/TCP,443/TCP      40m
```

![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1722126928/blog/shuntaka-gw-rpi5-grafana-on-k8/qeffcrwnyhzdvrstep5o.png)

ユーザー名 `admin` とパスワードを入力します。

```bash
$ kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

ArgoCDのパスワード再設定。環境を汚さないようにMacにCLIを入れようと思いましたが、k8sクラスタとの接続周りの設定が面倒なため、pi1にインストールします。

linux-brewがARMをサポートしていないことから、バイナリを取得して配備します

```bash
VERSION=2.11.7
curl -SL -o argocd-linux-arm64 https://github.com/argoproj/argo-cd/releases/download/v$VERSION/argocd-linux-arm64
sudo install -m 555 argocd-linux-arm64 /usr/local/bin/argocd
rm argocd-linux-arm64
```

ArgoCDのサーバーへログイン。引数にはMetalLBが採番したIPを利用します。

```bash
$ argocd login --insecure 192.168.86.202
WARNING: server certificate had error: tls: failed to verify certificate: x509: certificate signed by unknown authority. Proceed insecurely (y/n)? y
Username: admin
Password:
'admin:login' logged in successfully
Context '192.168.86.202' updated
```

```bash
argocd account update-password
```

パスワードが更新されたら、シークレットは削除します
```bash
kubectl --namespace argocd delete secret/argocd-initial-admin-secret
```

```bash:p1上で実行
$ kubectl config get-contexts -o name
kubernetes-admin@kubernetes
```


```bash:p1で実行
$ argocd cluster add $(kubectl config current-context)
WARNING: This will create a service account `argocd-manager` on the cluster referenced by context `kubernetes-admin@kubernetes` with full cluster level privileges. Do you want to continue [y/N]? y
INFO[0002] ServiceAccount "argocd-manager" created in namespace "kube-system"
INFO[0002] ClusterRole "argocd-manager-role" created
INFO[0002] ClusterRoleBinding "argocd-manager-role-binding" created
INFO[0007] Created bearer token secret for ServiceAccount "argocd-manager"
Cluster 'https://192.168.1.1:6443' added
```

QuickStart用にサンプルアプリを登録します

```bash
export APPLICATION_NAME="guestbook"
export GITHUB_URL="https://github.com/argoproj/argocd-example-apps.git"
export DIR_PATH="guestbook"
export NAME_SPACE="default"

argocd app create $APPLICATION_NAME \
  --repo $GITHUB_URL \
  --path $DIR_PATH \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace $NAME_SPACE
```

![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1722145126/blog/shuntaka-gw-rpi5-grafana-on-k8/rnn1a7iqc7bhwnhvm4dy.png)
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1722145126/blog/shuntaka-gw-rpi5-grafana-on-k8/ghauynwqxwm9tujgzt6o.png)

状態を確認します
```bash
$ argocd app get guestbook
Name:               argocd/guestbook
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          default
URL:                https://argocd.example.com/applications/guestbook
Source:
- Repo:             https://github.com/argoproj/argocd-example-apps.git
  Target:
  Path:             guestbook
SyncWindow:         Sync Allowed
Sync Policy:        Manual
Sync Status:        OutOfSync from  (d7927a2)
Health Status:      Missing

GROUP  KIND        NAMESPACE  NAME          STATUS     HEALTH   HOOK  MESSAGE
       Service     default    guestbook-ui  OutOfSync  Missing
apps   Deployment  default    guestbook-ui  OutOfSync  Missing
```

このコマンドを実行し、リポジトリからマニフェストを取得し、kubectl applyを実行。立ち上げたUIからも実行可能。
```bash
$ argocd app sync guestbook
TIMESTAMP                  GROUP        KIND   NAMESPACE                  NAME    STATUS    HEALTH        HOOK  MESSAGE
2024-07-28T05:45:00+00:00            Service     default          guestbook-ui  OutOfSync  Missing
2024-07-28T05:45:00+00:00   apps  Deployment     default          guestbook-ui  OutOfSync  Missing
2024-07-28T05:45:00+00:00            Service     default          guestbook-ui  OutOfSync  Missing              service/guestbook-ui created
2024-07-28T05:45:00+00:00   apps  Deployment     default          guestbook-ui  OutOfSync  Missing              deployment.apps/guestbook-ui created
2024-07-28T05:45:00+00:00   apps  Deployment     default          guestbook-ui    Synced  Progressing              deployment.apps/guestbook-ui created
2024-07-28T05:45:00+00:00            Service     default          guestbook-ui    Synced  Healthy                  service/guestbook-ui created

Name:               argocd/guestbook
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          default
URL:                https://argocd.example.com/applications/guestbook
Source:
- Repo:             https://github.com/argoproj/argocd-example-apps.git
  Target:
  Path:             guestbook
SyncWindow:         Sync Allowed
Sync Policy:        Manual
Sync Status:        Synced to  (d7927a2)
Health Status:      Progressing

Operation:          Sync
Sync Revision:      d7927a27b4533926b7d86b5f249cd9ebe7625e90
Phase:              Succeeded
Start:              2024-07-28 05:45:00 +0000 UTC
Finished:           2024-07-28 05:45:00 +0000 UTC
Duration:           0s
Message:            successfully synced (all tasks run)

GROUP  KIND        NAMESPACE  NAME          STATUS  HEALTH       HOOK  MESSAGE
       Service     default    guestbook-ui  Synced  Healthy            service/guestbook-ui created
apps   Deployment  default    guestbook-ui  Synced  Progressing        deployment.apps/guestbook-ui created
```


![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1722145622/blog/shuntaka-gw-rpi5-grafana-on-k8/s1mqdfhrl52ssrucf2f6.png)

RPi上では、ARMのためguestbookは動作しなさそう..(泣
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1722146013/blog/shuntaka-gw-rpi5-grafana-on-k8/xjfyniwwqj8tjhe2lj3y.png)

仕方がないので削除を実施

```bash
argocd app delete guestbook --yes
```

## ArgoCD と GitHub Apps連携

以下を参考に進める

* [ArgoCDにGitHub Appの資格情報を使ってリポジトリを登録する](https://zenn.dev/vampire_yuta/articles/58e85dad7a8e58)

GitHub Appsを作成

* Homgepage URLは適当
* webhookは非アクティブ化でひとまず作成
* 作成したら、デフォルト+
 * Contentsのリード権限を付与
 * Packagesのリード権限を付与

その他やること
* 連携先のリポジトリを作成(https://github.com/shuntaka9576/apps.git)
* Fine-grained personal access tokensを利用している場合、リポジトリ権限を追加

```bash
export GITHUB_URL="https://github.com/shuntaka9576/apps.git"
export GITHUB_APP_ID="955521"
export GITHUB_INSTALL_ID="53237765"
export PEM_NAME="shuntakaargocd.2024-07-27.private-key.pem"

argocd repo add $GITHUB_URL \
  --github-app-id $GITHUB_APP_ID \
  --github-app-installation-id $GITHUB_INSTALL_ID \
  --github-app-private-key-path $PEM_NAME
```

リポジトリが登録されていることを確認

```bash
$ argocd repo list
TYPE  NAME  REPO                                      INSECURE  OCI    LFS    CREDS  STATUS      MESSAGE  PROJECT
git         https://github.com/shuntaka9576/apps.git  false     false  false  false  Successful
```

コンソール上からも確認
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1722148994/blog/shuntaka-gw-rpi5-grafana-on-k8/oj33pyavomadfg0ofggb.png)



## コンテナレジストリの導入

```yml:helmfile.yaml
repositories:
  - name: harbor
    url: https://helm.goharbor.io

releases:
  - name: harbor
    namespace: harbor
    chart: harbor/harbor
    version: 1.15.0
    values:
      - values.yaml
```

実行自体はこけますが、repositoriesは追加されます。

```bash
helmfile sync
```

以下のコマンドで変更可能箇所を出力します。出力しますが変更箇所はありません。

```bash
helm show values harbor/harbor>values.yaml
```

適用します。
```bash
helmfile sync
```

サービスを確認します

```bash
$ k get svc -n harbor
NAME                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
harbor-core         ClusterIP   10.100.67.177    <none>        80/TCP              52s
harbor-database     ClusterIP   10.100.235.34    <none>        5432/TCP            52s
harbor-jobservice   ClusterIP   10.103.106.133   <none>        80/TCP              52s
harbor-portal       ClusterIP   10.104.48.38     <none>        80/TCP              52s
harbor-redis        ClusterIP   10.106.139.93    <none>        6379/TCP            52s
harbor-registry     ClusterIP   10.96.207.6      <none>        5000/TCP,8080/TCP   52s
harbor-trivy        ClusterIP   10.100.36.199    <none>        8080/TCP            52s
```

```yaml:harbor-loadbalancer.yaml
apiVersion: v1
kind: Service
metadata:
  name: harbor-loadbalancer
  namespace: harbor
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  selector:
    app: harbor
    component: portal
```

適用します

```bash
kubectl apply -f harbor-loadbalancer.yaml
```

## アプリケーションをArgoCD経由でホストティングする

ローカルでコンテナをビルドし、ghcrへpushします。ghcrを利用する理由は以下のためです。

* harborがarmに対応していない
* ECRはAWS外の場合、下り料金がかかる

ghの権限でghcrにpushできるscopeを追加する

```bash:Mac上で実行
gh auth login --scopes "write:packages"
```

```bash:Mac上で実行
export APP_NAME="example-app"
export COMMIT_HASH=$(git rev-parse --short HEAD)

docker buildx build --platform linux/arm64 -t $APP_NAME .
docker tag $APP_NAME ghcr.io/$(gh api user --jq .login)/${APP_NAME}:latest
docker tag $APP_NAME ghcr.io/$(gh api user --jq .login)/${APP_NAME}:${COMMIT_HASH}

# 動かす場合
docker run -p 8080:8080 $APP_NAME

# ghcrへpush
echo $(gh auth token) | docker login ghcr.io -u $(gh api user --jq .login) --password-stdin

docker push ghcr.io/$(gh api user --jq .login)/${APP_NAME}:latest
docker push ghcr.io/$(gh api user --jq .login)/${APP_NAME}:${COMMIT_HASH}
```


シークレットを作成します。トークンはGitHub Token(classic)となります。これはghcrがリポジトリに依存しない要素なので、これしかないというのが現状です。scopeは`read:packages`のみ付与すれば良いです。

```bash:pi1上で実行
kubectl create namespace example-app

export PAT=""
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=hoge \
  --docker-password=$PAT \
  --docker-email=hoge@gmail.com \
  -n example-app
```

argo-cd上でアプリケーション作成します。
```bash:pi1上で実行
export APPLICATION_NAME="example-app"
export GITHUB_URL="https://github.com/shuntaka9576/apps.git"
export DIR_PATH="example-app/manifests"
export NAME_SPACE="example-app"

argocd app create $APPLICATION_NAME \
  --repo $GITHUB_URL \
  --path $DIR_PATH \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace $NAME_SPACE

argocd app sync $APPLICATION_NAME
# 削除する場合
# argocd app delete $APPLICATION_NAME
```


## TailScale設定

現在metalLB側でARPリクエストに対して応答するようにし、IP動的に割り当てています。

* `--advertise-routes=192.168.86.0/24` でローカルネットワークのルートをTailScaleネットワークにアドバタイズすることを指示している
* `--accept-routes` は、他のtailscaleノードがアドバタイズしているルートを自動的に受け入れることを許可する

```bash
# Install Tailscale on a node
curl -fsSL https://tailscale.com/install.sh | sh

# Authenticate and enable IP forwarding
sudo tailscale up --advertise-routes=192.168.86.0/24 --accept-routes
```

tailscaleのWebUI側でも設定が必要です(CLI側でオプションをつければ必要ないかもしれません...)

![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1722206214/blog/shuntaka-gw-rpi5-grafana-on-k8/iwlgartb4c2b3lqnt4qx.png)
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1722206217/blog/shuntaka-gw-rpi5-grafana-on-k8/ntw9y8xyjise3swxzgj6.png)
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1722206221/blog/shuntaka-gw-rpi5-grafana-on-k8/ajbn6nezib8yuvvfmpdf.png)


## ネットワーク公開設定

自分は本サイトで利用しようとしたドメインをムームードメインで契約していたため、ムームー側にcloudflareのNSレコードを設定した。このときPending Nameserver UpdateからActivateになるまで、少し時間がかかるので注意。

※ 出来れば移管したかったが、取得後60日経過していなかったので、断念


```bash
brew install cloudflared
```

```bash
cloudflared tunnel login
```

![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1722285370/blog/shuntaka-gw-rpi5-grafana-on-k8/lv4svx0rt5rekkljohon.png)
![img](https://res.cloudinary.com/dkerzyk09/image/upload/v1722285371/blog/shuntaka-gw-rpi5-grafana-on-k8/u2oub29iknqqi4ejfks5.png)

```bash:出力結果
(中略)

If you wish to copy your credentials to a server, they have been saved to:
/Users/shuntaka/.cloudflared/cert.pem
```

```bash
export DOMAIN_NAME="shuntaka.cloud"

# トンネル名
export CF_TUNNEL_NAME=tunnel-dev

# ドメイン指定
export CF_TUNNEL_DOMAIN="tunnel-test.${DOMAIN_NAME}"

# Tunnel作成
$ cloudflared tunnel create $CF_TUNNEL_NAME
Tunnel credentials written to /Users/shuntaka/.cloudflared/<UUID>.json. cloudflared chose this file based on where your origin certificate was found. Keep this file secret. To revoke these credentials, delete the tunnel.

Created tunnel tunnel-dev with id <UUID>

# DNSのCNAMEにTunnelを追加します
$ cloudflared tunnel route dns $CF_TUNNEL_NAME $CF_TUNNEL_DOMAIN

2024-07-29T20:52:27Z INF Added CNAME tunnel-test.shuntaka.cloud which will route to this tunnel tunnelID=<UUID>
```

CloudflareのWebコンソールからZero Trust -> Networks -> Tunnelsで確認できます。

```bash:~/.cloudflared/config.yml
tunnel: <UUID>
credentials-file: /Users/shuntaka/.cloudflared/<UUID>.json

ingress:
  - hostname: tunnel-test.shuntaka.cloud
    service: http://localhost:8080
  - service: http_status:404
```

以下のコマンドで実行できます。`https://tunnel-test.shuntaka.cloud`とlocalhost:8080がマッピングされていることがわかります。
```bash
cloudflared tunnel run tunnel-dev
```


## 最後に

先人が沢山いたのですが、かなり大変でした。環境を完全に消すのも大変だったので、都度M.2にイメージを焼き直してました。気づいたらUbuntu Server 24.04 LTSがリリースされていて手順を差し替えるか迷いました。23.10だと、クーラー周りの問題で音がかなり厳しいのでカーネルアップデートが必須でしたが、24.04は問題なかったので、結果待ち時間が少ない24.04の手順にしました。

大変だった点は主にネットーワーク周りになります。最初はp1をNAT化し、pi2以降はpi1をデフォルトゲートウェイとしてネットーワークに出ていたのですが、MetalLBを入れてから通信がループして設定を削除したりなど。。ここら辺もう少し再現確認してブログにできたらと思います。

各ワーカーノードのストレージとしてM.2を使いましたが、容量の大きいNFSサーバーを作ってそこに永続化リソースをまとめるの方が良いのかなとも思いました！

また弊社のハードに強い方にいくつかアドバイス頂き感謝です！

今後経過についてもブログ記事化できたらと思います！


## 付録

### トラブルシューティング
#### MetalLBで割り当てられたIPに接続できない

curlやブラウザ経由でMetalLBで割り当てられたIPでgrafanaの画面が閲覧できな場合です。以下のコマンドでARPリクエスト送信し、tcpdumpでARP応答を返していることを確認できれば、MetalLBとしては動作しています。FW設定などに問題あるかどうかを切り分けできます。

```bash
# ARPリクエストを送信
arping -I eth0 192.168.86.100
# 応答していることを確認
sudo tcpdump -n -i eth0 arp src host 192.168.86.100
```

#### Grafanaダッシュボードからテストリクエストが通らない

corednsを再起動したら動くケースがありました。

```bash
kubectl rollout restart deployment coredns -n kube-system
```

#### PrometheusやGrafanaのホスティングをやり直したい

永続化ストレージの消し忘れに注意

```bash
helm uninstall prometheus -n monitoring
helm uninstall grafana -n monitoring

sudo rm -rf /mnt/k8s/pv/grafana
sudo rm -rf /mnt/k8s/pv/prometheus-server
sudo rm -rf /mnt/k8s/pv/prometheus-alertmanager
```

## 参考資料

* [Raspberry Pi 5でNVMeストレージ起動を試す](https://zenn.dev/usagi1975/articles/2023-12-29-000_raspi5_boot_nvme)
* [Raspberry Pi 5をM.2 NVMe SSDからブートしてベンチマークしてみた](https://zenn.dev/yuyakato/articles/82383b8c8869d3)
* [オンプレKubernetes環境にMetallbをインストールしてLoadbalanserServiceを利用できるようにする](https://zenn.dev/vampire_yuta/articles/ccbc57be8e092a)
* [minikubeでHelmを勉強してみた](https://zenn.dev/ryoyoshii/articles/f6409ede9ef84b)
* [ローカルKubernetesクラスター上でPrometheusのメトリクスをGrafanaダッシュボードで可視化する](https://zenn.dev/ring_belle/articles/prometheus-grafana-metrics)
* [我流お家クラウドを構築する【1】~kubernetesをセットアップする~](https://zenn.dev/murasame29/articles/ee5868123c1e33)
* [4台のラズパイでk8sクラスタを組んだ](https://skanehira.github.io/this-week-in-gorilla/articles/raspberry-pi-cluster.html)
* [TROUBLESHOOTING METALLB](https://metallb.universe.tf/troubleshooting/)

