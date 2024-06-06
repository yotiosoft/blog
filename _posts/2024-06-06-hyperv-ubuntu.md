---
layout: post
title: "Hyper-VのUbuntuの仮想ハードディスクサイズを拡張する"
tags: [Windows, Ubuntu, Hyper-V]
excerpt_separator: <!--more-->
---

Hyper-V にインストールした Ubuntu で空き容量が足りなくなり、仮想ハードディスクの拡張が必要になったのでそのときのメモです。

単純に「VHD のサイズを変えとけばいいだろ～」と思っていたのですが、そうじゃなかったので少し手間取りました。

<!--more-->

# 概要

ホスト環境:

- メーカー：HP
- 型番：Envy te02
- OS：Windows 11 Pro 64bit
- CPU：Intel Core i7-13700 @ 2.10GHz
- RAM：32GB
- SSD：1TB
- Hyper-V は有効化済み

ゲスト環境:

- OS：Ubuntu 22.04.1
- RAM：16GB

# 作業内容

## 1. VHD を拡張する

Hyper-V の右側のメニュー「ディスクの編集」を開き、

![image-20240606114219883](../../../assets/img/post/2024-06-06-hyperv-ubuntu/image-20240606114219883.webp)

拡張したい仮想ハードディスクのファイルを選択して次へ。

![image-20240606114310452](../../../assets/img/post/2024-06-06-hyperv-ubuntu/image-20240606114310452.webp)

「拡張」を選び、

![image-20240606114333222](../../../assets/img/post/2024-06-06-hyperv-ubuntu/image-20240606114333222.webp)

拡張後のサイズを設定しておきます。

![image-20240606114353170](../../../assets/img/post/2024-06-06-hyperv-ubuntu/image-20240606114353170.webp)

## 2. Ubuntu 側の設定

まずは Ubuntu 側のターミナルから、ハードディスクサイズを再スキャン。

```bash
$ sudo sh -c "echo 1 > /sys/class/block/sda/device/rescan"
```

これにて Ubuntu 側で空き領域が認識されます。

![Screenshot from 2024-06-06 11-51-17](../../../assets/img/post/2024-06-06-hyperv-ubuntu/Screenshot from 2024-06-06 11-51-17.webp)



次に、``parted`` でパーティション拡張の設定を行っていきます。

```bash
$ sudo parted /dev/sda
GNU Parted 3.4
/dev/sda を使用
GNU Parted へようこそ！ コマンド一覧を見るには 'help' と入力してください。
(parted)
```

まずはパーティション情報を見ておきます。

このとき、一部の領域が利用されていないので GPT を修正するか？という警告が表示されたので「F」で Fix を選んでおきました。

```bash
(parted) print free                                                       
警告: /dev/sda で利用可能な領域の一部が利用されていません。GPT を修正して全ての領域を利用可能にするか(104857600
ブロック増えます)、このままで続行することができますが、どうしますか？ 
修正/Fix/無視(I)/Ignore? F	# ← 入力
モデル: Msft Virtual Disk (scsi)
ディスク /dev/sda: 376GB
セクタサイズ (論理/物理): 512B/512B
パーティションテーブル: gpt
ディスクフラグ: 

番号  開始    終了    サイズ  ファイルシステム  名前  フラグ
      17.4kB  1049kB  1031kB  空き容量
14    1049kB  5243kB  4194kB                          bios_grub
15    5243kB  116MB   111MB   fat32                   boot, esp
 1    116MB   322GB   322GB   ext4
      322GB   376GB   53.7GB  空き容量
```

一番下の「空き容量」と表示されている領域が拡張した領域です。これに ``/dev/sda`` を割り当てていきます。

``/dev/sda`` に該当すると思われるのが下から2番目の番号「1」の領域。よってこの番号で拡張先を指定します。

```bash
(parted) resizepart 1                                                     
警告: パーティション /dev/sda1 は使用中です。それでも実行しますか？
はい(Y)/Yes/いいえ(N)/No?  y		# ← 入力
終了?  [322GB]? 100% 			  # ← 入力
```

「100%」で空き容量すべてを割り当てることを示します。

もう一回 ``print free`` で拡張されたか確認します。

```bash
(parted) print free                                                       
モデル: Msft Virtual Disk (scsi)
ディスク /dev/sda: 376GB
セクタサイズ (論理/物理): 512B/512B
パーティションテーブル: gpt
ディスクフラグ: 

番号  開始    終了    サイズ  ファイルシステム  名前  フラグ
      17.4kB  1049kB  1031kB  空き容量
14    1049kB  5243kB  4194kB                          bios_grub
15    5243kB  116MB   111MB   fat32                   boot, esp
 1    116MB   376GB   376GB   ext4
```

きちんと拡張されてました。これにて完了です。

parted 終了後（``(parted) q`` で抜ける）、パーティション設定が反映されました。

![Screenshot from 2024-06-06 11-59-02](../../../assets/img/post/2024-06-06-hyperv-ubuntu/Screenshot from 2024-06-06 11-59-02.webp)

## 3. ファイルシステムサイズを反映する

これにて容量が拡張された…はずなのですが、まだファイルシステム側は認識していないようでした。

![Screenshot from 2024-06-06 12-04-42](../../../assets/img/post/2024-06-06-hyperv-ubuntu/Screenshot from 2024-06-06 12-04-42.webp)

よって、別途 ``resize2fs`` でファイルシステムにサイズの拡張を反映します。

```bash
$ sudo resize2fs /dev/sda1
resize2fs 1.46.5 (30-Dec-2021)
Filesystem at /dev/sda1 is mounted on /; on-line resizing required
old_desc_blocks = 38, new_desc_blocks = 44
The filesystem on /dev/sda1 is now 91721979 (4k) blocks long.
```

今度こそ反映されました。お疲れ様でした。

![Screenshot from 2024-06-06 12-05-52](../../../assets/img/post/2024-06-06-hyperv-ubuntu/Screenshot from 2024-06-06 12-05-52.webp)

# 参考文献

- [VMware上のUbuntu 22.04 ファイルシステム拡張 #parted - Qiita](https://qiita.com/mcyang/items/a32b914db073f308a3cb){:target="_blank"}

- [Hyper-V の Ubuntu 18.04.1 LTS のディスク容量を拡張する #Ubuntu - Qiita](https://qiita.com/nakat-t/items/87d0ae049a5e0b57e469){:target="_blank"}
