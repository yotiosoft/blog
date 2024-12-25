---
layout: post
title: "サポート終了した古いUbuntuをアップグレードさせる（apt update 404エラーへの対処）"
tags: [Ubuntu, Linux]
excerpt_separator: <!--more-->
---

メリークリナックス！今回は Ubuntu ディストリビューションのアップグレードに関するお話です。

先日まで、恥ずかしながら数ヶ月前にサポート切れした Ubuntu 環境（23.10）を使っていました。LTS ではないのでサポート期間が短いことは分かっていたんですが、その環境は研究で使用しており、実験環境に影響が出うるなどの不安事項によりアップグレードを怠っていました。

普通に ``apt`` でアップグレードすればいいじゃないか、と思われるかもしれないですが、Ubuntu の場合はそのバージョンのサポートが切れると、``apt`` のリポジトリからもパッケージリストが削除されてしまうのです。つまり、**apt update できなくなります**。

今回はその対処をしたときの話です。

<!--more-->	

# 環境

- Ubuntu 23.10.1

# アップグレードできない問題

ずばり、``apt update`` しようとしたときに 404 Not Found が発生してしまいます。ネットワークには接続できているのに、一部のパッケージリストにアクセスできないと言われているのです。

```bash
$ sudo apt update
無視:1 http://ports.ubuntu.com/ubuntu-ports mantic InRelease                   
無視:2 http://ports.ubuntu.com/ubuntu-ports mantic-updates InRelease           
無視:3 http://ports.ubuntu.com/ubuntu-ports mantic-security InRelease     
ヒット:4 https://download.opensuse.org/repositories/home:/npreining:/debian-ubuntu-onedrive/xUbuntu_24.04 ./ InRelease
エラー:5 http://ports.ubuntu.com/ubuntu-ports mantic Release
  404  Not Found [IP: 2620:2d:4000:1::16 80]
エラー:6 http://ports.ubuntu.com/ubuntu-ports mantic-updates Release
  404  Not Found [IP: 2620:2d:4000:1::16 80]
エラー:7 http://ports.ubuntu.com/ubuntu-ports mantic-security Release
  404  Not Found [IP: 2620:2d:4000:1::16 80]
パッケージリストを読み込んでいます... 完了
E: リポジトリ http://ports.ubuntu.com/ubuntu-ports mantic Release には Release ファイルがありません。
N: このようなリポジトリから更新を安全に行うことができないので、デフォルトでは更新が無効になっています。
N: リポジトリの作成とユーザ設定の詳細は、apt-secure(8) man ページを参照してください。
E: リポジトリ http://ports.ubuntu.com/ubuntu-ports mantic-updates Release には Release ファイルがありません。
N: このようなリポジトリから更新を安全に行うことができないので、デフォルトでは更新が無効になっています。
N: リポジトリの作成とユーザ設定の詳細は、apt-secure(8) man ページを参照してください。
E: リポジトリ http://ports.ubuntu.com/ubuntu-ports mantic-security Release には Release ファイルがありません。
N: このようなリポジトリから更新を安全に行うことができないので、デフォルトでは更新が無効になっています。
N: リポジトリの作成とユーザ設定の詳細は、apt-secure(8) man ページを参照してください。
```

（ログが流れてしまったので上記は再現です。細かいところが違うかもしれません）

# 原因

23.10 Mantic Minotaur のサポート期限が切れ、ubuntu-ports のリポジトリから削除されてしまったのが原因です。

サポートが切れると ``apt update`` もできなくなるんですね。

# アップグレードする

サポートが切れてるならさっさとディストリビューションをアップグレードして、その後 ``apt update`` するか…と思ってもできません（このままでは）。なぜなら ``do-release-upgrade`` では、事前に ``apt update`` と ``apt upgrade`` によって全パッケージの更新が済んでないとアップグレードが中断されてしまうからです。これは詰んだ！

```bash
$ sudo do-release-upgrade
Checking for a new Ubuntu release
Your Ubuntu release is not supported anymore.
For upgrade information, please visit:
http://www.ubuntu.com/releaseendoflife
Please install all available updates for your release before upgrading.
```

## 対処

Ubuntu では、サポートが切れたディストリビューションのリポジトリは ``old-releases.ubuntu.com`` に移動します。

よって、``apt`` のソースリストを編集し、リポジトリの URL を ``old-releases.ubuntu.com`` に変更すれば OK です。



まずは ``/etc/apt/sources.list`` のバックアップを取っておき、エディタで開きます。

```bash
$ sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
$ sudo vim /etc/apt/sources.list
```

私の環境では内容は以下の通りになっていました。

```
deb http://ports.ubuntu.com/ubuntu-ports mantic main restricted universe multiverse 
deb http://ports.ubuntu.com/ubuntu-ports/mantic-updates main restricted universe multiverse 
deb http://ports.ubuntu.com/ubuntu-ports/mantic-security main restricted universe multiverse 
deb http://archive.raspberrypi.org/debian/ buster main
deb-src http://raspbian.raspberrypi.org/raspbian/ buster main contrib non-free rpi 
deb http://ftp.de.debian.org/debian buster main
```

このうち、``ports.ubuntu.com`` を ``old-releases.ubuntu.com`` に変更します。サードパーティ製のリポジトリ（この場合は ``archive.raspberrypi.org/`` など）は変更しなくてよいです。

```
deb http://old-releases.ubuntu.com/ubuntu mantic main restricted universe multiverse
deb http://old-releases.ubuntu.com/ubuntu/ mantic-updates main restricted universe multiverse
deb http://old-releases.ubuntu.com/ubuntu/ mantic-security main restricted universe multiverse
deb http://archive.raspberrypi.org/debian/ buster main
deb-src http://raspbian.raspberrypi.org/raspbian/ buster main contrib non-free rpi
deb http://ftp.de.debian.org/debian buster main
```

これで保存し、さらに ``apt`` のパッケージ情報のキャッシュをクリアしておきます。

```bash
$ sudo rm -rf /var/lib/apt/lists/*
```

## 再度 apt update してみる

これで ``apt update`` できるか試してみました。

別の opensuse のサードパーティリポジトリで署名無効のエラーが発生しましたが、これについては下記の記事に従い対処しました。今回の本題とは無関係ですので、ここでは触れません。

- [Ubuntu-debianのonedrive更新で苦労した～公開鍵更新 #Linux - Qiita](https://qiita.com/2SB100/items/0107d2ca7b844e9423a3){:target="_blank"}

![image-20241225225802038](../../../assets/img/post/2024-12-25-old-ubuntu/image-20241225225802038.webp)

なんとかサポート切れしたディストリビューションでも ``apt update`` が叶いました。

## 今度こそアップグレードする

以降、``do-release-upgrade`` でアップグレードを試みました。

![image-20241225225605485](../../../assets/img/post/2024-12-25-old-ubuntu/image-20241225225605485.webp)

特に何の問題もなくアップグレードが進みました。無事 Ubuntu 24.04.1 LTS にアップグレード完了。

```bash
$ lsb_release -a
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 24.04.1 LTS
Release:	24.04
Codename:	noble
```

``/etc/apt/sources.list`` も新しいバージョンにリポジトリ URL に書き換わっていました。

```
deb http://ports.ubuntu.com/ubuntu-ports/ lnoble main restricted universe multiverse
deb http://ports.ubuntu.com/ubuntu-ports/ lnoble-updates main restricted universe multiverse
deb http://ports.ubuntu.com/ubuntu-ports/ lnoble-security main restricted universe multiverse
```

# おわりに

サポートが切れてしまったら ``apt`` のリポジトリを ``old-releases.ubuntu.com`` に置き換えよう。置き換えたらササッとディストリビューションをアップグレードしよう。

というかサポートが切れる前にこまめにアップグレードしよう。自分への戒めです。

# 参考文献

- [Ubuntu で apt-get update が404になる問題 #Ubuntu - Qiita](https://qiita.com/nyanchu/items/a8cfc5cf627d70d798bf){:target="_blank"}
