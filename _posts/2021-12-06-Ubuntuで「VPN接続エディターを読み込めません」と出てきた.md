---
layout: post
title: "Ubuntuで「VPN接続エディターを読み込めません」と出てきた"
tags: [Ubuntu]
excerpt_separator: <!--more-->
---

UbuntuではVPN設定に初期状態でL2TPのオプションがないので、L2TPのマネージャをインストール必要があります。

```bash
$ sudo apt install network-manager-l2tp
```

上記のコマンドを実行すると、VPNの設定画面にL2TPの設定が追加されます。  
 ![スクリーンショット 2021-12-06 21-40-48](../../../assets/img/post/2021-12-06-Ubuntuで「VPN編集エディターを読み込めません」と出てきた/スクリーンショット 2021-12-06 21-40-48.png) 
さあこれで設定できる！と開いたら、L2TPの設定画面でこんなエラーが。  
![スクリーンショット 2021-12-06 21-43-15](../../../assets/img/post/2021-12-06-Ubuntuで「VPN編集エディターを読み込めません」と出てきた/スクリーンショット 2021-12-06 21-43-15.png)  

```
エラー:VPN接続エディターを読み込めません
```

うおお、これではVPNの設定ができないではないか…

<!--more-->  

# 解決策

上記示した``network-manager-l2tp``の他にもう一つ、``network-manager-l2tp-gnome``をインストールする必要があったようです。  

```bash
sudo apt install network-manager-l2tp-gnome
```

![スクリーンショット 2021-12-06 21-57-23](../../../assets/img/post/2021-12-06-Ubuntuで「VPN編集エディターを読み込めません」と出てきた/スクリーンショット 2021-12-06 21-57-23.png)  
これで解決。
