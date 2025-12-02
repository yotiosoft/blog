---
layout: post
title: "sambaで運用しているNASにiPadから書き込めるようにする"
tags: [Linux, samba]
excerpt_separator: <!--more-->
---

自宅の Raspberry Pi サーバで samba で NAS を運用しているんですが、iPad からだけなぜか**「読み取り専用」となってしまって書き込めない**問題が起きていました。Windows からでも macOS からでも普通に書き込めるのに、iPad だけ書き込めません。

<img src="../../../assets/img/post/2025-11-24-ipad-samba-nas/IMG_3423.webp" alt="IMG_3423" style="zoom:20%;" />

というわけで、今回はこれをサクッと解決しましょう。

<!--more-->	

# 実行環境

- サーバ
  - Raspberry Pi 5
  - Ubuntu 24.04.3 LTS (Linux 6.8.0)
  - samba 4.19.5-Ubuntu
  - ファイルシステム：ext4 (on dm-crypt on dm-linear)
- クライアント
  - iPad Air m3
  - iPadOS 18.6.2
  - 標準のファイルアプリ

# 解決策

## samba に vfs_fruit と vfs_streams_xattr を追加する

macOS 端末との互換性に向けられた `vfs_fruit` モジュールと `vfs_streams_xattr` モジュールを有効にします。

`vfs_fruit` モジュールは samba の VFS モジュールで、下記の機能を有効にします。

- AFP 互換メタデータへの対応（AppleDouble など）

- macOS の拡張属性（com.apple.\*）の保存

- Finder のリソースフォークへの対応

- Time Machine SMB

結果的には Apple SMB との互換性が確保される効果が得られると。

また、`streams_xattr` はクライアント側からの代替データストリーム（ADS）を POSIX の xattrs に格納できるようにするための機能です。主に NTFS 用と説明されていますが、iPad 側から書き込むにも ADS が書き込めないと「書き込み不可」になってしまうようで、`vfs_fruit` とセットで使う必要があります（少なくとも手元の環境ではそうでした）。

### 進め方

まずはお好みのエディタで `/etc/samba/smb.conf` を開き、`[global]` セクションの下に下記の行を追加します。

```toml
[global]
vfs objects = fruit streams_xattr
```

（今回は使わないのでスキップしますが）TimeMachine を使いたい場合は、TimeMachine サポートも有効にしておきましょう。

```toml
fruit:time machine = yes
```

`smb.conf` を保存したら、設定を再読込して完了です。

```bash
$ sudo smbcontrol all reload-config
```



これで完了です。簡単ですね。

手元の環境でも、iPad からファイル書き込みやディレクトリ作成ができるようになりました。

<img src="../../../assets/img/post/2025-11-24-ipad-samba-nas/IMG_3428.webp" alt="IMG_3428" style="zoom:40%;" />

# 参考文献

- [vfs_fruit (samba manpages)](https://www.samba.gr.jp/project/translation/current/htmldocs/manpages/vfs_fruit.8.html){:target="_blank"}
- [vfs_streams_xattr (samba manpages)](https://www.samba.gr.jp/project/translation/3.6/htmldocs/manpages-3/vfs_streams_xattr.8.html)(https://www.samba.gr.jp/project/translation/current/htmldocs/manpages/vfs_fruit.8.html){:target="_blank"}
- [1.13. MacOS クライアント向けの Samba の設定 \| ネットワークファイルサービスの設定および使用 \| Red Hat Enterprise Linux \| 9 \| Red Hat Documentation](https://docs.redhat.com/ja/documentation/red_hat_enterprise_linux/9/html/configuring_and_using_network_file_services/assembly_configuring-samba-for-macos-clients_assembly_using-samba-as-a-server){:target="_blank"}
