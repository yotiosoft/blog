---
layout: post
title: "Ubuntuでxrdpのリモートデスクトップから音声を出力させたい"
tags: [Ubuntu, Linux, Respberry Pi]
excerpt_separator: <!--more-->
---

Raspberry Pi 5 環境の Ubuntu 23.10 にて、リモートデスクトップ環境 xrdp を入れたとき、音声出力ができないという問題にぶち当たりました。

サウンドの設定を見てみると、サウンド出力先に ``Dummy Output`` しかありません。

![Screenshot from 2024-04-07 09-54-16](../../../assets/img/post/2024-03-12-xrdp-sound/Screenshot from 2024-04-07 09-54-16.webp)

今回はこれを解決し、リモートデスクトップからでも音声を出力できるようにするまでの過程を記録しておきます。

<!--more-->

# 環境

```bash
$ neofetch
            .-/+oossssoo+/-.               ytani@raspberry-pi-5
        `:+ssssssssssssssssss+:`           --------------------
      -+ssssssssssssssssssyyssss+-         OS: Ubuntu 23.10 aarch64
    .ossssssssssssssssssdMMMNysssso.       Host: Raspberry Pi 5 Model B Rev 1.0
   /ssssssssssshdmmNNmmyNMMMMhssssss/      Kernel: 6.5.0-1013-raspi
  +ssssssssshmydMMMMMMMNddddyssssssss+     Uptime: 1 hour, 7 mins
 /sssssssshNMMMyhhyyyyhmNMMMNhssssssss/    Packages: 2408 (dpkg), 9 (snap)
.ssssssssdMMMNhsssssssssshNMMMdssssssss.   Shell: bash 5.2.15
+sssshhhyNMMNyssssssssssssyNMMMysssssss+   Terminal: /dev/pts/1
ossyNMMMNyMMhsssssssssssssshmmmhssssssso   CPU: BCM2835 (4) @ 2.400GHz
ossyNMMMNyMMhsssssssssssssshmmmhssssssso   Memory: 1246MiB / 7943MiB
+sssshhhyNMMNyssssssssssssyNMMMysssssss+
.ssssssssdMMMNhsssssssssshNMMMdssssssss.
 /sssssssshNMMMyhhyyyyhdNMMMNhssssssss/
  +sssssssssdmydMMMMMMMMddddyssssssss+
   /ssssssssssshdmNNNNmyNMMMMhssssss/
    .ossssssssssssssssssdMMMNysssso.
      -+sssssssssssssssssyyyssss+-
        `:+ssssssssssssssssss+:`
            .-/+oossssoo+/-.

```

# 注意

近年の Ubuntu（22.04～）では標準でリモートデスクトップ VNC をサポートしており、設定画面からリモートデスクトップを有効化できます。VNC は今回扱う xrdp とは異なる環境ですのでご注意ください。

今回は xrdp を別途インストールし、xrdp のリモートデスクトップ環境を導入しています。その理由として、

- 外部モニターがオフのときでもリモート接続できるようにしたかった
- しかし、Raspberry Pi 5 は HDMI ケーブルを抜くと GUI が自動でオフになるようで、VNC ではモニターに接続しておかないとリモート接続できなかった
  - 正確に言うと、接続はできるけど一瞬でセッションが中断してしまう
- 設定を変えてみたが（[このあたり](https://qiita.com/devemin/items/25477dc8323af2eb5cba){:target="_blank"}を参照）、変化はなかった
- 一方で、xrdp であれば外部モニターに接続しなくともリモート接続できた

といった点が挙げられます。

# はじめに

最初に xrdp をインストールしておきます。インストール、サービス有効化からポートの開放まで。

```bash
$ sudo apt install xrdp
$ sudo systemctl enable xrdp
$ sudo ufw allow 3389
```



この状態でプライベート IP アドレスとアカウント名、パスワードを入力すると、とりあえずはリモートデスクトップ接続できました。

![image-20240408203159465](../../../assets/img/post/2024-03-12-xrdp-sound/image-20240408203159465.webp)

![image-20240408203128700](../../../assets/img/post/2024-03-12-xrdp-sound/image-20240408203128700.webp)

しかし、音が出ません。サウンド設定を見ても ``Dummy Output`` という、いかにも音声出力できませんよという出力デバイスしか見当たりませんでした。

# （失敗）pulseaudio の導入

[ラズパイに音付きでリモート接続して操作する（xrdp+pulseaudio）   ofuton.org](https://hp.ofuton.org/426/){:target="_blank"}

こちらを参考にさせていただき、まずは ``pulseaudio`` および ``pulseaudio-module-xrdp`` をインストールします。

## sources.list の編集

pulseaudio のインスールには ``apt build-dep`` を利用するので、``/etc/apt/sources.list`` にリポジトリを追加しておきます。

```bash
$ sudo vi /etc/apt/sources.list
...
deb-src http://raspbian.raspberrypi.org/raspbian/ buster main contrib non-free rpi # <- 追記
```

その後、``sudo apt update`` しておきます。

```bash
$ sudo apt update
```

しかし、ここで最初のハマりポイントが発生。

### ハマりポイント①：「公開鍵を利用できないため、以下の署名は検証できませんでした」

これに関しては、今 ``sources.list`` に登録した ``http://raspbian.raspberrypi.org/raspbian/`` の公開鍵を登録しておけばよいです。

```bash
公開鍵を利用できないため、以下の署名は検証できませんでした: NO_PUBKEY [公開鍵]
```

``NO_PUBKEY`` の後ろに公開鍵が表示されますので、それを ``apt-key`` で登録しておきます。

```bash
$ sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys [公開鍵をここに入力]
```

## pulseaudio のインストール

次に ``build-dep`` および ``pulseaudio`` をインストールします。

まずはダウンロード用ディレクトリの用意。

```bash
$ mkdir ~/pulseaudio
$ cd ~/pulseaudio
```

続いてインストール。

```bash
$ sudo apt build-dep pulseaudio
$ sudo apt source pulseaudio
$ cd pulseaudio-12.2
$ sudo ./configure
```

このとき、1行目の ``sudo apt build-dep pulseaudio`` で2つ目のハマりポイントがありました。

### ハマりポイント②：「依存: libgconf2-dev しかし、インストールすることができません」

pulseaudio に必要な ``libgconf2-dev`` が存在しないと言われます。

検索してみると、たしかにリポジトリ上に見つかりません。

```bash
$ apt search libgconf
ソート中... 完了
全文検索... 完了
```

ググってみると、arm64 向けの ``libgconf2-dev`` を配布しているリポジトリを見つけました。

[Debian -- buster の libgconf2-dev パッケージに関する詳細](https://packages.debian.org/buster/libgconf2-dev){:target="_blank"}

このリポジトリを先ほどと同じように ``sources.list`` に追記しておきます。

```bash
$ sudo vi /etc/apt/sources.list
...
deb http://ftp.de.debian.org/debian buster main 	# <- 追記
```

そして ``sudo apt update`` します。

```bash
$ sudo apt update
```

すると、ここでも前述のハマりポイント①と同じ現象が見られました。同じように対処します。

```bash
$ sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys [公開鍵をここに入力]
```

## pulseaudio-module-xrdp のビルド＆インストール

次に、``pulseaudio-module-xrdp`` を git clone してきてビルドしインストールします。

```bash
$ sudo apt install libpulse-dev
$ cd ~/pulseaudio
$ git clone https://github.com/neutrinolabs/pulseaudio-module-xrdp.git
$ cd pulseaudio-module-xrdp
$ sudo ./bootstrap
$ sudo ./configure PULSE_DIR=/home/[ユーザ名]/pulseaudio/pulseaudio-12.2
$ sudo make
$ sudo make install
```

### ハマりポイント③：configure の実行中にエラー発生

``libpulse-dev`` がないと言われてしまいました。インストールしたはずなのですが。

一旦 ``autoremove`` してもう一度インストールし直すと直りました。

```bash
$ sudo apt autoremove libpulse-dev
$ sudo apt install libpulse-dev
```

## インストール完了の確認

```bash
$ ls $(pkg-config --variable=modlibexecdir libpulse)
```

を実行し、``module-xrdp-sink.la``、``module-xrdp-sink.so``、``module-xrdp-source.la``、``module-xrdp-source.so`` があれば完了です。

# いざ再起動！…あれ？

その後再起動し、サウンド設定を見てみました。すると…あれ？

![Screenshot from 2024-04-07 09-54-16](../../../assets/img/post/2024-03-12-xrdp-sound/Screenshot from 2024-04-07 09-54-16-1712577501906-2.webp)

相変わらず ``Dummy Output`` しか選択できませんでした。

# （成功）PipeWire を導入する

どうやら、近年の Ubuntu では PulseAudio ではなく PipeWire がデフォルトになっているようで。

[Xrdp - no audio. Dummy Output is only device.](https://www.linuxquestions.org/questions/slackware-14/xrdp-no-audio-dummy-output-is-only-device-4175730631/){:target="_blank"}

Ubuntu 23.10 では、PipeWire の xrdp モジュールを導入しなければなりませんでした。

## **[pipewire-module-xrdp](https://github.com/neutrinolabs/pipewire-module-xrdp)** をインストールする

```bash
$ mkdir ~/pipewire
$ cd ~/pipewire

$ sudo apt install pipewire pipewire-audio pipewire-bin libpipewire-0.3-dev libspa-0.2-dev
$ git clone https://github.com/neutrinolabs/pipewire-module-xrdp.git
$ cd ./pipewire-module-xrdp
$ ./bootstrap
$ ./configure
$ make
$ sudo make install
```

再起動してみて、サウンド設定を確認。

すると、``xrdp-sink`` の表示が！ようやくリモートデスクトップ環境でも音声出力ができるようになりました。

![image-20240408210718729](../../../assets/img/post/2024-03-12-xrdp-sound/image-20240408210718729.webp)
