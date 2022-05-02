---
layout: post
title: "macOS MontereyでVirtualBoxがインストールできない"
tags: [Mac]
excerpt_separator: <!--more-->
---

MacにVirtualBoxをインストールしようとしたら、なぜかインストールに失敗。インストール時に「インストールを完了できませんでした」と言われてしまいました。  
![スクリーンショット 2022-04-11 14.15.33](../../../assets/img/post/スクリーンショット 2022-04-11 14.15.33-9655999.png)  

一応Virtualbox自体は起動できるものの、仮想マシンは起動できず。  
![スクリーンショット 2022-04-11 14.34.44](../../../assets/img/post/スクリーンショット 2022-04-11 14.34.44.png)  
**Kernel driver not installed (rc=-1908)**と出てしまう。カーネルドライバがインストールされていないということで、やはりVirtualboxのインストールが途中で失敗している様子が窺えます。

<!--more-->  

# 環境

- 実機環境
  - MacBook Pro 2016
  - macOS Monterey 12.2.1
  - CPU: intel i5 6360U
  - RAM: 8GB
- VirtualBox
  - ver.6.1.14



# 原因はMacのセキュリティ

どうやらMacの設定の「セキュリティとプライバシー」にて、Oracle製ソフトウェアのアップデートの実行を許可しないとインストールが成功しないようです。  
  
インストール時、「セキュリティとプライバシー」に「開発元"Oracle America, Inc."のシステムソフトウェアがアップデートされました」という表示が現れ、「許可」ボタンが表示されるはずなんですが…  
あれ？  
![スクリーンショット 2022-04-12 2.54.20](../../../assets/img/post/スクリーンショット 2022-04-12 2.54.20.png)  
ない。ない。どこにもない。  
許可ボタンが出てこない…  

これでは許可しようがありません。インストールできないじゃないか…

# 解決策:USサイトから最新版をダウンロード

[Changelog - Oracle VM VirtualBox](https://www.virtualbox.org/wiki/Changelog){:target="_blank"}  

> **VirtualBox 6.1.30** (released November 22 2021)
>
> - macOS host: fix multiple bugs specific to macOS Monterey in installer and startup of kernel extensions

調べてみると、VirtualBox 6.1.28以前にはmacOS Moterey特有のインストール関連のバグがあるらしく。  
昨年11月リリースの6.1.30で改善されたようです。  

**日本語のOracleのダウンロードページには最新で6.1.14までしか公開されていませんが、VirtualBoxのUSサイトでは6.1.32まで公開されています。**  
ということで、今日時点で最新の6.1.32を英語版VirtualBoxサイトからダウンロード。  
[Downloads - Oracle VM VirtualBox](https://www.virtualbox.org/wiki/Downloads){:target="_blank"}  

インストールを始める前に、前バージョン（6.1.14）のアンインストールも忘れずに。  
![スクリーンショット 2022-04-11 14.33.08](../../../assets/img/post/スクリーンショット 2022-04-11 14.33.08.png)  

最新版のインストールを始めたら、Macの設定画面「セキュリティとプライバシー」に許可ボタンが出現しました。  
![スクリーンショット 2022-04-11 14.30.43](../../../assets/img/post/スクリーンショット 2022-04-11 14.30.43.png)  
許可ボタンを押してインストールを進めたら、無事完了。仮想マシンも起動できました。  
![スクリーンショット 2022-04-11 14.29.38](../../../assets/img/post/スクリーンショット 2022-04-11 14.29.38.png)

# おわりに

VirtualBoxをダウンロードするときは[VirtualBoxのUSサイト](https://www.virtualbox.org/wiki/Downloads){:target="_blank"}から最新版をダウンロードしましょう。[日本語のOracleのページ](https://www.oracle.com/jp/virtualization/technologies/vm/downloads/virtualbox-downloads.html){:target="_blank"}は更新されておらず、未だにバージョン6.1.14（2年前のリリース）が最新版として扱われています。  

「virtualbox ダウンロード」とかでググると日本語のOracleのページが先に出てきてしまうので要注意。確かに、[別のページ]("https://www.oracle.com/jp/virtualization/technologies/virtualbox/downloads.html"){:target="_blank"}には「Oracle VM Virtualboxの最新版は、こちら（USサイト）からダウンロードをお願いします。」と書いてあった。しかし、肝心のダウンロードページには記載なし。こういうの、結構困るよなぁ…
