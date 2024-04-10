---
layout: post
title: "Raspberry Pi 5にUbuntuをインストールしたらファンが全力で回りっぱなしになった"
tags: [Raspberry Pi, Ubuntu, Linux]
excerpt_separator: <!--more-->
---

前回に引き続き、Raspberry Pi 5 の話です。

前回：[Raspberry Pi 5を購入しました](https://blog.yotiosoft.com/2024/03/24/raspberry-pi-5.html)



Raspberry Pi 5 を購入して早2週間、サーバとして利用するうえで色々と設定したり調べたりしなければならない点が多く、書きたいトピックが複数溜まっています。今回はその第1回目、OS のインストール編です。

ざっくりいうと、Raspberry Pi 5 には PWM 制御の CPU ファン機能があり、公式ケースを購入すると付属品として CPU ファンが付いてきます。普通は CPU の温度に合わせて回転数が変化するのですが、Raspberry Pi 5 に Ubuntu 23.10 をインストールしたらファンが全力で回り続けてしまいました。

<!--more-->

# 背景

Raspberry Pi 向けの OS としては Raspberry Pi OS が主流ですが、今回は自分が使い慣れている環境を優先し、ひとまず Ubuntu 23.10 をインストールすることにしました（もうじき Ubuntu 24.04 が出ますが待ちきれませんでした）。この Ubuntu 23.10 はサポート期間が2024年7月までと短いですが、Raspberry Pi 5 を初めてサポートするなどと行った魅力的な点があります。

## OS インストール

Raspberry Pi は microSD に OS を書き込んでブートさせるのですが、そのために Raspberry Pi Imager というものが用意されています。

[Raspberry Pi Imager](https://www.raspberrypi.com/software/){:target="_blank"}

この Raspberry Pi Imager には、2024年4月時点では Ubuntu 23.10 をインストールするオプションが用意されています。

![スクリーンショット 2024-03-23 174152 - コピー](../../../assets/img/post/2024-04-06-raspberry-pi-5-kernel/スクリーンショット 2024-03-23 174152 - コピー.webp)

![スクリーンショット 2024-03-23 192322](../../../assets/img/post/2024-04-06-raspberry-pi-5-kernel/スクリーンショット 2024-03-23 192322.webp)

![スクリーンショット 2024-03-23 194154](../../../assets/img/post/2024-04-06-raspberry-pi-5-kernel/スクリーンショット 2024-03-23 194154.webp)

今回は上の「Ubuntu Desktop 23.10 (64-bit)」を選びました。

これを microSD に書き込みインストールしました。

## 起動

いざ、microSD を Raspberry Pi 5 に差し込んで起動します。

すると、CPU ファンが全力で回り始めました。うんうん、起動時なら CPU 使用率高いし仕方ないよね。

そう思いながら待ち続けると、Ubuntu の GUI インストール画面が起動しました。まだ回る？

インストールが完了し再起動しました。まだ回り続ける？

再起動後ログインし、デスクトップ画面が表示されました。しかし、未だにファンは全力で唸りを上げながら回り続けます。



CPU 使用率を確認しても数 % 程度。3月下旬、室温20度前後。CPU 温度を調べても50度前後で、そこまで全力で回り続ける必要性がわかりません。

もしや GUI のせいか？と思い、microSD を Ubuntu Server 23.10 に焼き直しました。しかし、状況は変わりません。

ではディストリビューションかカーネルのせいか？と思い、Raspberry Pi OS に焼き直して起動しました。すると、今度はファンが静かです。

# 原因と対処

というわけで、Ubuntu 23.10（というよりそのデフォルトのカーネルバージョン）と Raspberry Pi 5 との間に何らかの互換性の問題がありそうです。調べてみるとこんな記事が。

[Fan speed control on Raspberry Pi 5 not working In Ubuntu 23.10 - Ask Ubuntu](https://askubuntu.com/questions/1490462/fan-speed-control-on-raspberry-pi-5-not-working-in-ubuntu-23-10){:target="_blank"}

ファンの速度を制御する PWM fan control のバグのようです。

既に最新版のカーネルに fix マージされているようなので、カーネルをアップデートして再起動します。

```bash
$ sudo apt update
$ sudo apt upgrade
```

再起動後、起動直後は CPU 使用率からかファンが全力で回りますが、起動が完了すると驚くほどファンが静かになりました。

アップデート後のカーネルバージョンは下記のとおりです（アップデート前のバージョンは記録し忘れました…）。

```bash
$ uname -r
6.5.0-1013-raspi
```

