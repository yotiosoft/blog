---
layout: post
title: "自宅の外から遠隔でWindowsマシンをオンオフする"
tags: [Windows, RaspberryPi]
excerpt_separator: <!--more-->
---

自宅の外から遠隔でWindowsマシンの電源をオンオフできるようにしたときの記録です。

<!--more-->

# 概要

対象マシン:

- メーカー：HP
- 型番：Envy te02
- OS：Windows 11 Pro 64bit
- CPU：Intel Core i7-13700 @ 2.10GHz
- RAM：32GB
- SSD：1TB

サーバ:

- Raspberry Pi 5
- OS：Ubuntu 23.10
- RAM：8GB

# 起動させる

## どうやって起動させるか？

Wake-on-LAN を使います。簡単に説明すると、Wake-on-LAN は LAN 内のマシンを遠隔で起動するための技術です。起動したいマシンの MAC アドレスが含まれたマジックパケットと呼ばれるものを PC に対して送出し、LAN 内に該当するマシンがあればそれが起動するというものです。

## 前準備：Windows 側で Wake-on-LAN を有効にする

Windows 側では標準では Wake-on-LAN が無効化されていますので、まずは有効化します。

デバイスマネージャを開き、「ネットワーク アダプター」の中から NIC アダプタを選択します。イーサネット接続の場合は Realtek PCIe ...、Wi-Fi 接続の場合は Intel(R) Wi-Fi... の場合が多いかと思います。

![image-20240628013428526](../../../assets/img/post/2024-06-28-wake-on-lan/image-20240628013428526.webp)



「詳細設定」タブから「Wake on magic なんとか」を選択し、「値」を「有効」にします。

![image-20240628013532323](../../../assets/img/post/2024-06-28-wake-on-lan/image-20240628013532323.webp)



次に MAC アドレスを確認します。これは Wake-on-LAN で起動する上で必須です。

設定を開き、「ネットワークとインターネット」から「イーサネット」（Wi-Fi 接続の場合は「Wi-Fi」）の項目を選択します。

![image-20240628014355349](../../../assets/img/post/2024-06-28-wake-on-lan/image-20240628014355349.webp)

下にスクロールし、「物理アドレス (MAC)」に記載されている値が MAC アドレスです。この値は後で使うのでメモるなり写メるなりしておいてください。

![image-20240628014340656](../../../assets/img/post/2024-06-28-wake-on-lan/image-20240628014340656.webp)

## （非推奨）Wake-on-LAN のポートを開放する

ここから先は二通りの方法があります。1つはサーバがいらないけどちょっと危険な方法、もう1つはサーバがいるけど安全な方法です。



まずは1つ目のサーバがいらない方法から。ルータ側で Wake-on-LAN 用のポートを開放する方法です。

ポートフォワーディングで設定しておくと、WAN 側から送られてきたマジックパケットが起動先の PC に中継されますので、ルータを越えて Wake-on-LAN が利用可能です。

まずはルータの設定を開きます。ルータ設定のアドレスはルータのメーカによりますが、筆者の環境では「192.168.0.1」です。ここからの設定方法はルータによって異なりますので、あくまでのご参考程度に。

Wake-on-LAN は一般的に、ポート番号 9（UDP）が用いられます [1] ので、これを開放しておきます。

![image-20240628014837756](../../../assets/img/post/2024-06-28-wake-on-lan/image-20240628014837756.webp)

転送先の IP アドレス（上記画像では「LAN側ホスト」）は PC のローカル IP アドレスです。こちらは先程の Windows の「ネットワークとインターネット」設定の画面から確認できます。



これで設定は完了です。以降、下記のような wakeonlan コマンドで Wake-on-LAN できるようになります。

```bash
$ wakeonlan -i [自宅のグローバルIPアドレス] -p 9 [PCのMACアドレス]
```



…と、これでできるはずなんですが、自分の環境では成功しませんでした。

wakeonlan の送信元が LAN 内ならこれで起動できるんですが、LAN 外だと起動できず。うーん、ルータがブロックしているんですかね。ただ、ルータのパケットフィルタリング設定を見ても9番ポートのパケットフィルターは見つかりません。

### おすすめしない理由

というわけで、この方法をおすすめしない理由は下記の3つです。

- ルータによってはマジックパケットがルータ越えできない場合がある [2]。
- そもそも危険。グローバル IP アドレスと MAC アドレスがバレてしまえば、誰でも認証せずに PC を起動できてしまうため。
- このようにポートフォワーディングしてしまうと、他の機器で Wake-on-LAN できなくなる。ポートフォワーディングによって9番ポートが1台の PC に専有されてしまうため。

## （推奨）Wake-on-LAN 用サーバを用意する

自宅の（起動したい PC と同じ）LAN 内に Wake-on-LAN 用のサーバを用意しておき、そのサーバに代わりに Wake-on-LAN をしてもらうという方法もあります。

この方法であれば9番ポートの開放が不要で、なおかつ SSH なり VPN なりの認証が通ったユーザだけが Wake-on-LAN できますので、ポート開放によって外部の人間に勝手に PC を起動される心配はありません。



筆者の場合は Raspberry Pi 5 をサーバとして利用しております。実現方法はいくらかありますが、SSH サーバを用意し、SSH ログインして ``wakeonlan`` を実行。これがオーソドックスな方法かと思います。

```bash
$ ssh raspi		# SSH でログイン
ytani@raspberry-pi-5:~$ wakeonlan [PCのMACアドレス]		# SSH サーバ上で wake-on-lan を実行
```



シェルスクリプトとして定義しておくとなお便利です。例えば：

```bash
#!/bin/sh
wakeonlan [PCのMACアドレス]
```

を ``wakeonlan.sh`` として home ディレクトリに配置しておけば、

```bash
$ ssh raspi
ytani@raspberry-pi-5:~$ ./wakeonlan.sh
```

で遠隔起動できます。

# シャットダウンさせる

シャットダウンは Wake-on-LAN ではできませんので、Windows 側のコマンドを用います。仕組みとしては、SSH サーバから Windows の ``Shutdown.exe`` を実行させるという方法です。

少し面倒ですが、SSH サーバから Windows に SSH ログインできるようにしておきます。Windows での OpenSSH の設定方法については下記をご参照ください。

なお、パスワード認証ではなく key 認証にすることを強く推奨します（セキュリティ上の理由もありますが、一番の理由は SSH サーバからコマンド一発でログインできるため）。

[Windows 用 OpenSSH の概要 \| Microsoft Learn](https://learn.microsoft.com/ja-jp/windows-server/administration/openssh/openssh_install_firstuse?tabs=gui){:target="_blank"}

[Windows 用 OpenSSH でのキーベースの認証 \| Microsoft Learn](https://learn.microsoft.com/ja-jp/windows-server/administration/openssh/openssh_keymanagement){:target="_blank"}



SSH サーバに Windows への SSH 鍵を登録しておいたら（仮に鍵の名前を「Windows」とします）、次に下記のシェルスクリプトを作成します。

```bash
#!/bin/sh
timeout 10 ssh Windows Shutdown.exe -s
```

やっていることは以下の通り。

- SSH で Windows マシンにログインする。
- Windows マシン上で ``Shutdown.exe`` を実行させる。
- 10秒経ったらタイムアウトする。

タイムアウトさせている理由としては、Windows 側がシャットダウンしてしまうと SSH から抜け出せなくなってしまうためです。Linux ではマシンがシャットダウンすると自動で SSH 接続が切断されるのですが、Windows はどうもそうではないようです（少なくとも筆者の環境ではそうでした）。



``Shutdown.exe`` の実行後、シャットダウンしたか確認するために、Windows PC に対して``ping``させてみるとなお安心感が高まります。

```bash
#!/bin/sh
timeout 10 ssh Windows Shutdown.exe -h
timeout 5 ping [Windows PCのローカルIPアドレス]
```



``Shutdown.exe`` は引数によって他の操作も可能です。例えば ``-h`` では休止状態に、``-r`` では再起動にできます。

参考：[shutdown \| Microsoft Learn](https://learn.microsoft.com/ja-jp/windows-server/administration/windows-commands/shutdown){:target="_blank"}

# 参考文献

[1] [スマホアプリで外出先から自宅のPCを遠隔起動してみる「Wol Wake on Lan Wan」 - 自動化厨のプログラミングメモブログ │ CODE:LIFE](https://www.aterm.jp/support/qa/qa_external/00227/win11.html){:target="_blank"}

[2] [「立ちはだかるルーターの壁」の巻：第8回 \| IT Leaders](https://it.impress.co.jp/articles/-/7797){:target="_blank"}

[3] [Windows 用 OpenSSH の概要 \| Microsoft Learn](https://learn.microsoft.com/ja-jp/windows-server/administration/openssh/openssh_install_firstuse?tabs=gui){:target="_blank"}

[4] [Windows 用 OpenSSH でのキーベースの認証 \| Microsoft Learn](https://learn.microsoft.com/ja-jp/windows-server/administration/openssh/openssh_keymanagement){:target="_blank"}

[5] [shutdown \| Microsoft Learn](https://learn.microsoft.com/ja-jp/windows-server/administration/windows-commands/shutdown){:target="_blank"}
