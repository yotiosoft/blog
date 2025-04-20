---
layout: post
title: "GRUBのsaved_entryを設定してGRUBが既定で起動するOSを変更する"
tags: [Ubuntu, Linux, Windows]
excerpt_separator: <!--more-->
---

以前、こんな記事を書きました。

- [GRUBがデフォルトで起動するOSを変更する \| 為せばnull](https://blog.yotio.jp/2021/10/16/GRUB%E3%81%8C%E3%83%87%E3%83%95%E3%82%A9%E3%83%AB%E3%83%88%E3%81%A7%E8%B5%B7%E5%8B%95%E3%81%99%E3%82%8BOS%E3%82%92%E5%A4%89%E6%9B%B4%E3%81%99%E3%82%8B.html)（2021年10月16日投稿）

今回やりたいのは上記の記事と同じことなんですが、久々にやろうとしたらなんか様子が変わっていたので、対処法をメモしておきます。

<!--more-->	

# 環境

- Ubuntu 20.04 LTS
- Linux 5.17.3
- GRUB 2.06

（あ、もう ubuntu のサポート今月で切れますね。普段使ってない環境とはいえ、そろそろアップデートせねば…）

# 背景

## ``sudo update-grub`` しても GRUB の default が変わらない

はい。上記の記事と同じ方法でデフォルトで起動する OS を変えようとしても、``/boot/grub/grub.cfg`` の値が書き換わりません。

### やったこと

まずは GRUB に登録されているカーネルの順序を確認。

```bash
$ grep -e "^\(menuentry\)\|\(submenu\)" /boot/grub/grub.cfg
menuentry 'Ubuntu' --class ubuntu --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-simple-3d25245f-880e-4743-b276-6a80ae867393' {
submenu 'Advanced options for Ubuntu' $menuentry_id_option 'gnulinux-advanced-3d25245f-880e-4743-b276-6a80ae867393' {
menuentry 'Windows Boot Manager (on /dev/sda1)' --class windows --class os $menuentry_id_option 'osprober-efi-A66D-E0B6' {
menuentry 'UEFI Firmware Settings' $menuentry_id_option 'uefi-firmware' {
```

今回デフォルトに設定したいのは ``Windows Boot Manager (on /dev/sda1)`` です。0から数えて上から2番目。

よって、``/etc/default/grub`` の ``GRUB_DEFAULT`` を2に設定します。

```bash
$ sudo vi /etc/default/grub
# If you change this file, run 'update-grub' afterwards to update
# /boot/grub/grub.cfg.
# For full documentation of the options in this file, see:
#   info -f grub -n 'Simple configuration'

GRUB_DEFAULT=2                # ←ここ
GRUB_TIMEOUT_STYLE=hidden
GRUB_TIMEOUT=10
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
GRUB_CMDLINE_LINUX=""
...
```

この内容で書き換え、設定を反映させます。

```bash
$ sudo update-grub
Sourcing file `/etc/default/grub'
Sourcing file `/etc/default/grub.d/99-use-saved-entry.cfg'
Sourcing file `/etc/default/grub.d/init-select.cfg'
Generating grub configuration file ...
Linux イメージを見つけました: /boot/vmlinuz-5.17.3
Found initrd image: /boot/initrd.img-5.17.3
Linux イメージを見つけました: /boot/vmlinuz-5.17.3.old
Found initrd image: /boot/initrd.img-5.17.3
Linux イメージを見つけました: /boot/vmlinuz-5.17.2
Found initrd image: /boot/initrd.img-5.17.2
Linux イメージを見つけました: /boot/vmlinuz-5.15.0-122-generic
Found initrd image: /boot/initrd.img-5.15.0-122-generic
Linux イメージを見つけました: /boot/vmlinuz-5.15.0-46-generic
Found initrd image: /boot/initrd.img-5.15.0-46-generic
Found Windows Boot Manager on /dev/sda1@/EFI/Microsoft/Boot/bootmgfw.efi
Adding boot menu entry for UEFI Firmware Settings
完了
```

``/boot/grub/grub.cfg`` を開き、反映されたか確認します。すると…

```bash
$ grep "set default" /boot/grub/grub.cfg
   set default="${next_entry}"
   set default="${saved_entry}"
```

あ、あれー？

``set default="2"`` となっていなければならないところが、``set default="${saved_entry}"`` になってしまっています。

# 原因

手元の環境で ``"${saved_entry}"`` になって変更できなくなってしまった経緯は、ちょっと分かりませんでした…

前回との違いとして、

- カーネルのバージョンが変わった（5.4.0→5.17.3）
- GRUB のバージョンが変わった（2.04→2.06）

などがあります。ただ、カーネルのバージョンは GRUB には関係ないはずです。

- GRUB のアップデートのときに何らかのはずみでこうなったのか？
- そういえば、なんで GRUB がバージョンアップされたんだ？

- そういえば、Ubuntu 環境をインストールし直した気もするし、し直してない気もする…
- あれ、Windows も入れ直さなかったっけ？

などなど、筆者の記憶が曖昧すぎて原因が不明です。いかんせん、もう現役引退して普段使っていない古い PC なのです。すみません…

# 対処

原因は不明ですが、とりあえず対処法はあります。``${saved_entry}`` の値を書き換えてしまえばよいのです。

この値を編集するのに便利なコマンドが ``grub-set-default`` です。

こちらは ``grub2-common`` パッケージに含まれていますので、まずはインストールしておきます。

```bash
$ sudo apt install grub2-common
...
以下のパッケージはアップグレードされます:
  grub-common grub-pc grub-pc-bin grub2-common
アップグレード: 4 個、新規インストール: 0 個、削除: 0 個、保留: 380 個。
3,558 kB のアーカイブを取得する必要があります。
この操作後に追加で 0 B のディスク容量が消費されます。
続行しますか? [Y/n] y
...
```

そして ``grub-set-default`` コマンドで既定値を設定してあげます。このとき、sudo が必要なので注意です。

```bash
$ sudo grub-set-default 2
```

実行後何も表示されることはありませんが、反映されているはずです。



以降、再起動後に GRUB の2番目の OS である Windows Bootloader がデフォルトで選択されるようになりました。

# まとめ

``${saved_entry}`` を変更するには ``grub-set-default`` コマンドを使いましょう。
