---
layout: post
title: "Hyper-VにUbuntuをインストール～SSHサーバの設定まで"
tags: [Windows, Ubuntu, Hyper-V]
feature-img: "/assets/img/feature-img/240506.webp"
thumbnail: "/assets/img/thumbnails/feature-img/240506.jpg"
excerpt_separator: <!--more-->
---

少し前に Hyper-V に Ubuntu を導入したときの手順を書き残しておきます。

<!--more-->

# 概要

導入の目的は修論の実験環境用途です。以前はデュアルブートで対応していましたが、いちいち再起動するのが面倒なのと、仮想環境でも十分な性能を発揮する PC が手に入ったことで Hyper-V 環境への移行を決意しました。

ホスト環境はこんな感じです。

- メーカー：HP

- 型番：Envy te02

- OS：Windows 11 Pro 64bit

- CPU：Intel Core i7-13700 @ 2.10GHz

- GPU：NVIDIA RTX 4070Ti (VRAM 12GB)

- RAM：32GB

- SSD：1TB

- イーサネット（Realtek PCIe GbE Family Controller）接続

- Hyper-V は有効化済み

大前提として、Hyper-V を使用するには Windows の Pro 版である必要があります（ググったら一応 Windows Home にも Hyper-V をインストールする術はあるみたいですが…その場合は自己責任で）。

また、Hyper-V ロールを既に有効化していることが前提です。デフォルトでは無効になっています。

[Windows 10 での Hyper-V の有効化 \| Microsoft Learn](https://learn.microsoft.com/ja-jp/virtualization/hyper-v-on-windows/quick-start/enable-hyper-v){:target="_blank"}

（Windows 11 の公式記事が見つかりませんでしたが、手順はほぼ同じだったと思います）

今回導入するのは Ubuntu 22.04 LTS です。もうすぐ 24.04 LTS が出るかと思いますが、執筆時点での最新版はこれです。

# Ubuntu のインストール

まずは Ubuntu の iso をダウンロードして…といった手順は必要ありません。

Hyper-V にはクイック作成というツールがあり、こちらから Windows や Ubuntu の仮想環境を簡単に導入できます。

というより、iso で導入してしまうと拡張セッションが使えず、画面解像度の調整やクリップボードの同期といった便利な連携機能が使えなくて不便なんですよね。クイック作成の存在を知らずに iso からインストールし、無理やり拡張セッションを使えるようにしようとすると遠回りすることになります。（経験者は語る）

まずは Hyper-V クイック作成を開きます。スタートメニューで検索すれば出てきます。

![SnapCrab_検索_2024-2-3_3-11-13_No-00.webp](..\..\..\assets\img\post\2024-02-03\SnapCrab_検索_2024-2-3_3-11-13_No-00.webp)

これを開き、

![SnapCrab_仮想マシンの作成_2024-2-3_3-13-54_No-00.webp](..\..\..\assets\img\post\2024-02-03\SnapCrab_仮想マシンの作成_2024-2-3_3-13-54_No-00.webp)

最新版の Ubuntu を選択して「仮想マシンの作成(V)」をクリックします。

するとダウンロードが始まりますので、しばらく待機します。

![SnapCrab_仮想マシンの作成_2024-2-3_3-15-53_No-00.webp](..\..\..\assets\img\post\2024-02-03\SnapCrab_仮想マシンの作成_2024-2-3_3-15-53_No-00.webp)

それで、ダウンロードが完了するとハードディスクの作成やら仮想環境の作成やらの処理を全自動でやってくれます。文字通りワンクリックで仮想環境を生成できました。

![SnapCrab_仮想マシンの作成_2024-2-3_3-20-31_No-00.webp](..\..\..\assets\img\post\2024-02-03\SnapCrab_仮想マシンの作成_2024-2-3_3-20-31_No-00.webp)

「接続」を選び、仮想マシン接続のウィンドウが出現したら「起動」をクリックします。

![SnapCrab_DESKTOP-6Q8U12V 上の Ubuntu 2204 LTS  - 仮想マシン接続_2024-2-3_3-21-30_No-00.webp](..\..\..\assets\img\post\2024-02-03\SnapCrab_DESKTOP-6Q8U12V%20上の%20Ubuntu%202204%20LTS%20%20-%20仮想マシン接続_2024-2-3_3-21-30_No-00.webp)

ここからは Ubuntu の通常通りの初期設定です。

![SnapCrab_DESKTOP-6Q8U12V 上の Ubuntu 2204 LTS  - 仮想マシン接続_2024-2-3_3-22-45_No-00.webp](..\..\..\assets\img\post\2024-02-03\SnapCrab_DESKTOP-6Q8U12V%20上の%20Ubuntu%202204%20LTS%20%20-%20仮想マシン接続_2024-2-3_3-22-45_No-00.webp)

ユーザ設定を終えたらインストール完了を待ちます。

![SnapCrab_DESKTOP-6Q8U12V 上の Ubuntu 2204 LTS  - 仮想マシン接続_2024-2-3_3-24-14_No-00.webp](..\..\..\assets\img\post\2024-02-03\SnapCrab_DESKTOP-6Q8U12V%20上の%20Ubuntu%202204%20LTS%20%20-%20仮想マシン接続_2024-2-3_3-24-14_No-00.webp)

再起動するとログイン画面が出現します。ネイティブ環境での Ubuntu 使いにとっては見慣れないログイン画面ですが、これは xrdp なるリモートデスクトップ接続のログイン画面で、初期設定で設定したユーザ名とパスワードをそのまま入力すれば OK です。

![SnapCrab_DESKTOP-6Q8U12V 上の Ubuntu 2204 LTS  - 仮想マシン接続_2024-2-3_3-29-30_No-00.webp](..\..\..\assets\img\post\2024-02-03\SnapCrab_DESKTOP-6Q8U12V%20上の%20Ubuntu%202204%20LTS%20%20-%20仮想マシン接続_2024-2-3_3-29-30_No-00.webp)

初回起動時に出てくるプライバシー設定等を終えて、おなじみのデスクトップ画面が出現しました。これにて Ubuntu の導入は完了です。

![SnapCrab_DESKTOP-6Q8U12V 上の Ubuntu 2204 LTS  - 仮想マシン接続_2024-2-3_3-31-44_No-00.webp](..\..\..\assets\img\post\2024-02-03\SnapCrab_DESKTOP-6Q8U12V%20上の%20Ubuntu%202204%20LTS%20%20-%20仮想マシン接続_2024-2-3_3-31-44_No-00.webp)

# SSH サーバの導入

これにて Ubuntu は導入できましたが、Windows アプリと同時に Ubuntu の ターミナルを使いたいとき、例えば Windows 版 VS Code から Ubuntu のターミナルにアクセスしたいときなどは、いちいち Hyper-V の Ubuntu と Windows との間でカーソルを行き来するのは面倒です。

よって、Ubuntu 側に SSH サーバを導入し、Windows 側のターミナルからアクセスできるようにしてしまいます。ついでに、外部の PC からもアクセスできるようにしてしまいましょう。手順は大きく分けて次の3つです。

- 他の PC から仮想環境にアクセスできるようにする

- SSH サーバをインストールする

- SSH key の生成＆登録

## 1. 他の PC から仮想環境にアクセスできるようにする

現状、仮想環境はネットワークアダプタとしてホストマシンの Default switch を使用しています。今のままでも Ubuntu から Windows ホストマシンを通してネットワークにアクセスすることはできますが、実際には WinNAT を通してアクセスしています。NAT (NAPT) は特定のポートを別の IP アドレスに変換する技術ですので、接続先の外部から見れば仮想環境との通信は「ホスト環境と通信」している状態であり、仮想環境を外部から識別することはできません。

よって Default switch のままでは他の PC からはアクセスできませんが、ホスト環境からであれば仮想環境にアクセスすることができます。もし他の PC から SSH 接続する必要がないのであれば、本項はスキップしても問題ありません。

外部 PC から仮想環境にアクセスできるようにする、つまり LAN で仮想環境を認識させるには、仮想スイッチを新たに生成しホスト環境と同じネットワークアダプタに接続させます。

まずは現状確認です。自身の環境では、初期状態では仮想環境のローカル IP アドレスはeth0 に対して「172.20.44.35」となっていました。

![SnapCrab_DESKTOP-6Q8U12V 上の Ubuntu 2204 LTS  - 仮想マシン接続_2024-2-3_3-57-26_No-00.webp](..\..\..\assets\img\post\2024-02-03\SnapCrab_DESKTOP-6Q8U12V%20上の%20Ubuntu%202204%20LTS%20%20-%20仮想マシン接続_2024-2-3_3-57-26_No-00.webp)

これが Default switch によって割り当てられたローカル IP アドレスです。仮想スイッチを生成するとルータから直接 IP アドレスが割り当てられます。一般的な自宅の LAN 環境であれば 192.168.xxx.xxx になるかと思います。

一旦仮想環境はオフにし、Hyper-V の「操作」メニューから「仮想スイッチ マネージャー」を選択します。

![SnapCrab_Hyper-V マネージャー_2024-2-3_4-13-5_No-00.webp](..\..\..\assets\img\post\2024-02-03\SnapCrab_Hyper-V%20マネージャー_2024-2-3_4-13-5_No-00.webp)

「外部」を選択した状態で「仮想スイッチの作成(S)」を選択。

![SnapCrab_DESKTOP-6Q8U12V の仮想スイッチ マネージャー_2024-2-3_4-14-9_No-00.webp](..\..\..\assets\img\post\2024-02-03\SnapCrab_DESKTOP-6Q8U12V%20の仮想スイッチ%20マネージャー_2024-2-3_4-14-9_No-00.webp)

「接続の種類」で「外部ネットワーク(E)」を選び、ホスト環境のイーサネットアダプタを選びます。名前も分かりやすい名前にしておきましょう。完了したら「OK」をクリックし仮想スイッチの作成を完了します。

（筆者の環境では既に作成済みのため実際には作成しません。同じ外部ネットワークに対して二重追加することはできないっぽいです）

![SnapCrab_DESKTOP-6Q8U12V の仮想スイッチ マネージャー_2024-2-3_17-52-28_No-00.webp](..\..\..\assets\img\post\2024-02-03\SnapCrab_DESKTOP-6Q8U12V%20の仮想スイッチ%20マネージャー_2024-2-3_17-52-28_No-00.webp)

次に、仮想環境の名前を右クリックして「設定(E)」を開きます。

![SnapCrab_Hyper-V マネージャー_2024-2-3_17-56-37_No-00.webp](..\..\..\assets\img\post\2024-02-03\SnapCrab_Hyper-V%20マネージャー_2024-2-3_17-56-37_No-00.webp)

「Default Switch」となっているところを先程作成した仮想スイッチに変更します。

![SnapCrab_DESKTOP-6Q8U12V 上の Ubuntu 2204 LTS の設定_2024-2-3_17-57-52_No-00.webp](..\..\..\assets\img\post\2024-02-03\SnapCrab_DESKTOP-6Q8U12V%20上の%20Ubuntu%202204%20LTS%20の設定_2024-2-3_17-57-52_No-00.webp)

また、「メモリ」の項目から「動的メモリを有効にする(E)」のチェックを外しておきます（ここのチェックを外しておかないと起動できません）。

![SnapCrab_DESKTOP-6Q8U12V 上の Ubuntu 2204 LTS の設定_2024-2-3_18-0-17_No-00.webp](..\..\..\assets\img\post\2024-02-03\SnapCrab_DESKTOP-6Q8U12V%20上の%20Ubuntu%202204%20LTS%20の設定_2024-2-3_18-0-17_No-00.webp)

以上で仮想スイッチの設定は完了です。仮想環境を立ち上げ、``$ ip a``でローカル IP アドレスを確認してみましょう。

![SnapCrab_DESKTOP-6Q8U12V 上の Ubuntu 2204 LTS  - 仮想マシン接続_2024-2-3_18-25-49_No-00.webp](..\..\..\assets\img\post\2024-02-03\SnapCrab_DESKTOP-6Q8U12V%20上の%20Ubuntu%202204%20LTS%20%20-%20仮想マシン接続_2024-2-3_18-25-49_No-00.webp)

先程と違い、eth0 に対して「192.168.0.88」が割り当てられました。これが LAN のルータから与えられたローカル IP アドレスです。これで LAN 内の他の PC からも仮想環境を認識することができます。

なお、現在のままではローカル IP アドレスは DHCP から動的に割り当てられており、アドレスが変化する場合もあります。必要に応じて IP アドレスの固定も行ってください。

## 2. SSH サーバをインストールする

仮想環境側を SSH サーバとするため、ここからは「ホスト」の立場が入れ替わりますのでご注意。仮想環境が SSH ホストに、Hyper-V のホスト環境が SSH クライアントになります。

まずは ``openssh-server`` をインストールします。

```bash
$ sudo apt install openssh-server
```

仮想環境側のポートを開放します。SSH のポート番号は 22 なので、

```bash
$ sudo ufw allow 22
$ sudo ufw reload
```

…とすると、``$ sudo ufw reload`` で ``Firewall not enabled (skipping reload)`` と言われてしまいました。というわけでファイアウォールを有効にします。

```bash
$ sudo ufw enable
Firewall is active and enabled on system startup
$ sudo ufw reload
Firewall reloaded
```

以上で SSH サーバの設定は完了です。

先程 ``$ ip a`` で確認した IP アドレスをもとに、port 22 にホスト環境側の Windows からアクセスしてみましょう。

```powershell
> ssh [仮想環境のローカル IP アドレス] -p [ポート番号:22]
```

現時点では SSH-key は設定していないのでパスワード認証になります。

![SnapCrab_ytani@ytani-Virtual-Machine ~_2024-2-3_18-22-50_No-00.webp](..\..\..\assets\img\post\2024-02-03\SnapCrab_ytani@ytani-Virtual-Machine%20~_2024-2-3_18-22-50_No-00.webp)

このように、ホスト環境の PowerShell からアクセスすることができました。

次に、LAN 内の他の PC からもアクセスしてみます。

![スクリーンショット 2024-02-03 18.30.13.webp](..\..\..\assets\img\post\2024-02-03\スクリーンショット%202024-02-03%2018.30.13.webp)

このように、LAN 内の他の PC（MacBook）からもアクセスできることが確認できました。

なお、現時点では LAN 内のみアクセス可能です。もし外部ネットワークからアクセスしたい場合はルータ側でポートの開放、およびポートフォワーディング等の設定が必要になります。

## 3. SSH key の生成＆登録

現状ではパスワード認証なので、公開鍵認証に切り替えます。

### クライアント側（ホスト環境 etc.）①

クライアント側、すなわち SSH サーバにアクセスする側の端末では、ssh-keygen 等を用いて SSH key を生成します。

鍵の種類は何種類かありますが、ed25519 にしたければ

```powershell
> ssh-keygen -t ed25519
```

を実行し、SSH key を生成しておきます。

``Enter file in which to save the key`` では出力先パスを設定します。キーの生成完了時に出力先が表示されます。

SSH key の生成時、2つのファイルが生成されます。1つは拡張子のないファイル、もう1つは「*.pub」形式のファイルです。前者が秘密鍵、後者が公開鍵です。

## ホスト側（仮想環境）

クライアント側で生成した SSH key のうち、「*.pub」のほうをホスト側（＝仮想環境）に設定しておきます。

具体的には、``~/.ssh/authorized_keys`` を作成し、「*.pub」の中身をコピペ（追記）します。

## クライアント側（ホスト環境 etc.）②

ホスト側への公開鍵の登録が完了したので、SSH config（``~/.ssh/config``; Windows の場合は ``C:\Users\[ユーザ名]\.ssh``）に設定を追加しておきます。``.ssh`` フォルダがなければ作成しておきます。

中身はこんな感じに：

```
Host [お好みの SSH サーバ名, 例:Ubuntu-2204]
  HostName [仮想環境の IP アドレス]
  User [仮想環境のユーザ名]
  Port [SSH サーバのポート:22]
  UserKnownHostsFile /dev/null
  PreferredAuthentications publickey
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile [先程生成した秘密鍵のファイルパス]
  IdentitiesOnly yes
  LogLevel FATAL
```

以上で SSH key の設定は完了です。

設定した SSH サーバ名でクライアント側（Windows 側）からアクセスしてみます。

![SnapCrab_ytani@ytani-Virtual-Machine ~_2024-2-3_20-0-10_No-00.webp](..\..\..\assets\img\post\2024-02-03\SnapCrab_ytani@ytani-Virtual-Machine%20~_2024-2-3_20-0-10_No-00.webp)

公開鍵を使用して ``ssh [SSH サーバ名]`` でアクセスできることが確認できました。

# 参考文献

- [Windows10にHyper-VでUbuntu仮想マシンを構築 #Ubuntu - Qiita](https://qiita.com/YuheiTani/items/b62c365cffa5b0088c7d){:target="_blank"}

- [仮想スイッチの種別と用途：Windows 10 Hyper-V入門 - ＠IT](https://atmarkit.itmedia.co.jp/ait/articles/2008/14/news018.html){:target="_blank"}

- [ssh公開鍵認証を実装する #Linux - Qiita](https://qiita.com/HamaTech/items/21bb9761f326c4d4aa65){:target="_blank"}