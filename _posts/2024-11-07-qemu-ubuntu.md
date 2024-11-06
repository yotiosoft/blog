---
layout: post
title: "nographicなCUI環境のQEMUにUbuntuをインストールする"
thumbnail: "/assets/img/thumbnails/feature-img/241107.png"
tags: [QEMU, Ubuntu]
excerpt_separator: <!--more-->
---

QEMU で Ubuntu Server を動かしたいとき、GUI が使える環境であればインストールメディアを指定すれば OK ですが、GUI が使えない（gtk が使えない、gtk initialization failed になるような環境）環境では少々面倒です。

今回は、QEMU/KVM 上で Ubuntu Server インストールメディアを CUI 環境で起動し、仮想ハードディスクにインストールするまでの過程をメモっておこうと思います。

<!--more-->

# この記事の想定

- 最終目標：``-nographic`` オプション付きの QEMU で Ubuntu Server のインストールメディアを立ち上げ、インストールする
- 読者対象：CUI 環境の QEMU で Ubuntu を使いたい人

# 実行環境の概要

- ホスト環境
  - OS : Ubuntu 22.04.3 LTS x86_64 (Linux kernel ver.6.8.0, 64bit)
  - CPU : 13th Gen Intel i7-13700 (4) @ 3.341GHz
  - RAM : 16 GB
- 仮想環境
  - qemu-system-x86_64 6.2.0
    - KVM 有効（``--enable-kvm``）
    - RAM：2048 MB
    - 仮想ハードディスク：qcow2 80GB
  
  - インストールするもの
    - Ubuntu 24.04.1 LTS x86_64
  

# 前準備

## インストールメディアを入手する

[Get Ubuntu Server \| Download \| Ubuntu](https://ubuntu.com/download/server){:target="_blank"}

ここから Ubuntu Server インストールメディアの iso ファイルを入手します。

今回は Ubuntu 24.04.1 の AMD64 版を入手しました。

## 仮想ハードディスクの用意

まずはインストール先に使う仮想ハードディスクを用意しておきます。

```bash
cd ~
mkdir ubuntu-vm
cd ubuntu-vm
qemu-img create -f qcow2 vm.qcow2 80G 
```

## KVM をロード

KVM を使う場合、``kvm_intel`` をカーネルにロードしておきます。

```bash
sudo modprobe kvm_intel 
```

# インストールメディアを起動する

## 普通に起動してみる

まずは

```bash
sudo qemu-system-x86_64 -hda vm.qcow2 -m 2048 -cdrom ubuntu-24.04.1-live-server-amd64.iso -boot d --enable-kvm -usb -nographic
```

で起動してみます。（CUI モードで起動するため、``--nographic`` が必須です）

すると GRUB のメニューが現れます。

![スクリーンショット 2024-11-06 22.54.58](../../../assets/img/post/2024-11-07-qemu-ubuntu/スクリーンショット 2024-11-06 22.54.58.webp)

ここで「Try or Install Ubuntu Server」を選ぶと…

![スクリーンショット 2024-11-06 22.56.38](../../../assets/img/post/2024-11-07-qemu-ubuntu/スクリーンショット 2024-11-06 22.56.38.webp)

このような黒い画面で止まってしまいます。

## 原因

VM の出力先が QEMU の仮想 VGA になっているのが原因。現在、``--nographic`` によってコンソールは VM のシリアルポートに接続されているため、出力先をシリアルポートに設定しなければなりません。

## 解決策：GRUB で出力先をシリアルポートに設定

コンソールの出力先をシリアルポートに設定します。

まずは、先程と同じように

```bash
sudo qemu-system-x86_64 -hda vm.qcow2 -m 2048 -cdrom ubuntu-24.04.1-live-server-amd64.iso -boot d --enable-kvm -usb -nographic
```

で起動します。そして、

![スクリーンショット 2024-11-06 22.54.58](../../../assets/img/post/2024-11-07-qemu-ubuntu/スクリーンショット 2024-11-06 22.54.58.webp)

この画面が現れたら、``E`` キーを押して GRUB のコマンド編集画面を開きます。

![スクリーンショット 2024-11-06 23.06.13](../../../assets/img/post/2024-11-07-qemu-ubuntu/スクリーンショット 2024-11-06 23.06.13.webp)

ここで ``linux        /casper/vmlinuz  ---`` の行に十字キーで移動し、``linux        /casper/vmlinuz`` と ``---`` の間に ``console=ttyS0,115200n8`` を追記します。

![スクリーンショット 2024-11-06 23.08.01](../../../assets/img/post/2024-11-07-qemu-ubuntu/スクリーンショット 2024-11-06 23.08.01.webp)

この ``console=ttyS0`` が出力先の指定です。

``ttyS0`` は Linux の1番目のシリアルポートを指します。``115200n8`` についてはこちら ↓ のページが詳しいです。

[KVMでvirt-installする時にextra-argsに指定するconsole=tty0 console=ttyS0,115200n8rとは一体何者なのか](https://blog.osstech.co.jp/posts/2020/08/tty-console/){:target="_blank"}



この状態で Ctrl+x または F10 キーを押すとインストールメディアが起動します。

まずは Booting a command list と表示され、

![スクリーンショット 2024-11-06 23.12.29](../../../assets/img/post/2024-11-07-qemu-ubuntu/スクリーンショット 2024-11-06 23.12.29.webp)

Linux カーネルの起動が開始し…

![スクリーンショット 2024-11-06 23.12.36](../../../assets/img/post/2024-11-07-qemu-ubuntu/スクリーンショット 2024-11-06 23.12.36.webp)

程なくして、Ubuntu Server のインストールメディアが起動します。

![スクリーンショット 2024-11-06 23.13.33](../../../assets/img/post/2024-11-07-qemu-ubuntu/スクリーンショット 2024-11-06 23.13.33.webp)

ここではインストーラを basic mode（ASCII 文字のみ）で起動するか、rich mode（unicode と rich colour への対応が必須）で起動するかが尋ねられます。説明を見る限り SSH 接続でリモートからもインストールできるっぽいです。

今回は rich mode を選びました。

![スクリーンショット 2024-11-06 23.15.48](../../../assets/img/post/2024-11-07-qemu-ubuntu/スクリーンショット 2024-11-06 23.15.48.webp)

ここからは言語設定、ユーザ設定などなど、いつもどおりの設定が続きます。指示に従って設定を進めていきます。

![スクリーンショット 2024-11-06 23.18.52](../../../assets/img/post/2024-11-07-qemu-ubuntu/スクリーンショット 2024-11-06 23.18.52.webp)

設定が終わるとインストール作業が始まります。

![スクリーンショット 2024-11-06 23.21.44](../../../assets/img/post/2024-11-07-qemu-ubuntu/スクリーンショット 2024-11-06 23.21.44.webp)

2〜3分程で完了しました。

![スクリーンショット 2024-11-06 23.22.08](../../../assets/img/post/2024-11-07-qemu-ubuntu/スクリーンショット 2024-11-06 23.22.08.webp)

CD ROM を取り出さなければならないので、一旦 QEMU を終了し、インストールメディアのオプション指定を削除してからもう一度起動します。

```bash
sudo qemu-system-x86_64 -hda vm.qcow2 -m 2048 -boot d --enable-kvm -usb -nographic
```



![スクリーンショット 2024-11-06 23.24.49](../../../assets/img/post/2024-11-07-qemu-ubuntu/スクリーンショット 2024-11-06 23.24.49.webp)

起動できました！これにてインストール完了です。

# まとめ

QEMU で CUI 環境でインストールする場合、シリアルポートに設定変更する必要があります。あとはインストールメディアがよしなにやってくれます。
