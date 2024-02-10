---
layout: post
title: "Wingetで{Vim|Emacs}を入れてgitのデフォルトエディタに設定する"
tags: [git, Windows]
excerpt_separator: <!--more-->
---

Git for Windows のデフォルトエディタが MinTTY という使い慣れてないエディタだったので変更しようと思います。

<!--more-->

# 概要

少し前まで Windows には公式のパッケージマネージャが存在せず、インストーラをダウンロードして、インストーラを開いて、…という煩わしい操作が必要だったのですが、近年は winget の登場により大分便利になりました。とはいっても全部やってくれるわけではないですが（後述）。

apt や homebrew などに比べるとまだまだですが、有名どころのパッケージはだいたい winget で入手できます。入手可能なパッケージは search コマンドで検索できます。

```powershell
PS C:\Users\ytani> winget search vim
名前                          ID                          バージョン  一致                 ソース
--------------------------------------------------------------------------------------------------
Vim Cheat Sheet               9WZDNCRDMCWR                Unknown                          msstore
Vimar By-me App List Exporter 9PFSSC3DTHK8                Unknown                          msstore
Vimar VIEW Pro                9NNG9QXL8W3Z                Unknown                          msstore
Vim                           vim.vim.nightly             9.1.0083                         winget
Vim                           vim.vim                     9.1.0004                         winget
qutebrowser                   qutebrowser.qutebrowser     3.1.0       Tag: vim             winget
win-vind                      pit-ray.win-vind            5.10.0      Tag: vim             winget
Neovim Nightly                Neovim.Neovim.Nightly       0.10.0      Tag: vim             winget
Neovim                        Neovim.Neovim               0.9.5       Tag: vim             winget
Vieb                          Jelmerro.Vieb               11.0.0      Tag: vim             winget
Helix                         Helix.Helix                 23.10       Tag: vim             winget
lf                            gokcehan.lf                 r31         Tag: vim             winget
4K Video Downloader           OpenMedia.4KVideoDownloader 4.29.0.5640 Tag: vimeo           winget
Kaku                          Chia-Lung.Kaku              2.0.2       Tag: vimeo           winget
Replit                        Replit.Replit               1.0.6       Tag: desenvolvimento winget
PS C:\Users\ytani> winget search emacs
名前      ID        バージョン 一致           ソース
----------------------------------------------------
GNU Emacs GNU.Emacs 29.2       Moniker: emacs winget
```

# Vim

## インストール＆PATH を通す

```powershell
> winget install vim
見つかりました Vim [vim.vim] バージョン 9.1.0004
このアプリケーションは所有者からライセンス供与されます。
Microsoft はサードパーティのパッケージに対して責任を負わず、ライセンスも付与しません。
ダウンロード中 https://github.com/vim/vim-win32-installer/releases/download/v9.1.0004/gvim_9.1.0004_x64.exe
  ██████████████████████████████  10.4 MB / 10.4 MB
インストーラーハッシュが正常に検証されました
パッケージのインストールを開始しています...
インストーラーは管理者として実行するように要求するため、プロンプトが表示されます。
インストールが完了しました
```

これで行けるはずなのですが…

```powershell
> vim search emacsd
vim : 用語 'vim' は、コマンドレット、関数、スクリプト ファイル、または操作可能なプログラムの名前として認識されません。
名前が正しく記述されていることを確認し、パスが含まれている場合はそのパスが正しいことを確認してから、再試行してください
。
発生場所 行:1 文字:1
+ vim search emacsd
+ ~~~
    + CategoryInfo          : ObjectNotFound: (vim:String) [], CommandNotFoundException
    + FullyQualifiedErrorId : CommandNotFoundException
```

これだけでは PATH が通ってないため使えません。

むしろそこを自動でやってくれるのがパッケージマネージャの醍醐味だろう、という個人的な見解は置いといて、さっさとパスを通します。

…とはいえ、インストールした先がわからない…

インストール中もインストール先の表示はありませんでした。

ということで、検索をかけます（管理者権限が必要）。

```powershell
> Get-ChildItem -Recurse -Filter "Vim" -Directory "C:\"


    ディレクトリ: C:\Program Files


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----        2024/02/10     22:57                Vim
> Get-ChildItem "C:\Program Files\Vim"


    ディレクトリ: C:\Program Files\Vim


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----        2024/02/10     22:57                vim91
-a----        2024/02/10     22:57           1309 _vimrc
```

``C:\Program Files``の下、``C:\Program Files\Vim\vim91``にありました（``vim``の後の数字はバージョンにより異なるかと思います）。これを PATH に追加します。

```powershell
> $ENV:Path+=";C:\Program Files\Vim\vim91"
```

ようやっと vim が使えるようになりました。

![SnapCrab_ytani@ytani-Virtual-Machine ~_2024-2-3_18-22-42_No-00.webp](..\..\..\assets\img\post\2024-02-11\SnapCrab_ytani@ytani-Virtual-Machine%20~_2024-2-3_18-22-42_No-00.webp)

## git のデフォルトエディタに設定する

```powershell
> git config --global core.editor vim
```

これだけで OK です。

試しに ``git commit`` してみると

![スクリーンショット 2024-02-10 232035.webp](..\..\..\assets\img\post\2024-02-11\スクリーンショット%202024-02-10%20232035.webp)

vim が立ち上がったことが確認できました。

# Emacs

次は Emacs です。同様に winget でインストールします。

```powershell
> winget install emacs
見つかりました GNU Emacs [GNU.Emacs] バージョン 29.2
このアプリケーションは所有者からライセンス供与されます。
Microsoft はサードパーティのパッケージに対して責任を負わず、ライセンスも付与しません。
ダウンロード中 https://ftp.gnu.org/gnu/emacs/windows/emacs-29/emacs-29.2-installer.exe
  ██████████████████████████████  69.5 MB / 69.5 MB
インストーラーハッシュが正常に検証されました
パッケージのインストールを開始しています...
インストールが完了しました
```

それで、こちらも当然のごとく PATH が通っていません。

```powershell
> emacs
emacs : 用語 'emacs' は、コマンドレット、関数、スクリプト ファイル、または操作可能なプログラムの名前として認識されませ
ん。名前が正しく記述されていることを確認し、パスが含まれている場合はそのパスが正しいことを確認してから、再試行してくだ
さい。
発生場所 行:1 文字:1
+ emacs
+ ~~~~~
    + CategoryInfo          : ObjectNotFound: (emacs:String) [], CommandNotFoundException
    + FullyQualifiedErrorId : CommandNotFoundException
```

検索をかけてみると、

```powershell
> Get-ChildItem -Recurse -Filter "Emacs" -Directory "C:\"


    ディレクトリ: C:\Program Files


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----        2024/02/10     23:22                Emacs
> Get-ChildItem "C:\Program Files\Emacs"


    ディレクトリ: C:\Program Files\Emacs


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----        2023/12/03      2:16                emacs-29.1
d-----        2024/02/10     23:23                emacs-29.2
-a----        2024/02/10     23:23         124642 Uninstall.exe
> Get-ChildItem "C:\Program Files\Emacs\emacs-29.2"


    ディレクトリ: C:\Program Files\Emacs\emacs-29.2


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----        2024/02/10     23:22                bin
d-----        2024/02/10     23:22                include
d-----        2024/02/10     23:23                lib
d-----        2024/02/10     23:23                libexec
d-----        2024/02/10     23:23                share
```

こちらも ``C:\Program Files``にありました。（バージョンが2つ入っちゃってるのは筆者が以前も同様にインストールしたからです）

vim と違い、こちらは ``emacs-[version]\bin`` ディレクトリの下にバイナリファイルがありました。最新版の方の PATH を通しておきます。

```powershell
> $ENV:Path+=";C:\Program Files\Emacs\emacs-29.2\bin"
```

``emacs -nw`` で CUI モードで起動してみます。

![スクリーンショット 2024-02-10 233149.webp](..\..\..\assets\img\post\2024-02-11\スクリーンショット%202024-02-10%20233149.webp)

こちらも起動できました。

## git のデフォルトエディタに設定する

```powershell
> git config --global core.editor emacs
```

でいけるかと思ったのですが、

```powershell
> git commit
hint: Waiting for your editor to close the file... error: cannot spawn emacs: No such file or directory
error: unable to start editor 'emacs'
Please supply the message using either -m or -F option.
```

うーん… emacs コマンドだとデフォルトで GUI が立ち上がるので、もしかしてこれが駄目だったりするのかな？

というわけで、``emacs -nw``で登録しておきましょう。

```powershell
> git config --global core.editor 'emacs -nw'
```

ちなみに、既に git で Vim をデフォルトエディタに設定している状態で Emacs に改宗（あるいはその逆）したい場合は ``--replace-all`` を使います。

```powershell
> git config --replace-all --global core.editor 'emacs -nw'
```

これにて Emacs でのコミットメッセージ編集が可能になりました。

![スクリーンショット 2024-02-10 234110.webp](..\..\..\assets\img\post\2024-02-11\スクリーンショット%202024-02-10%20234110.webp)
