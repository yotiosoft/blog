---
layout: post
title: "Docker Windowsのコンテナの保管場所を移動する"
tags: [Docker, Windows]
excerpt_separator: <!--more-->
---

空き容量が足りないよ～泣

![cdrive.png](..\..\..\assets\img\post\2023-11-19\cdrive.png)

Docker を日常的に利用しているんですが、C ドライブに置かれているコンテナが問答無用で C ドライブをガバガバ食っていくので空き容量が足りません。（そもそも C ドライブが 128GB なのが悪い）

よって今回は、容量を巣食うコンテナを別の HDD に移動します。

<!--more-->

# 実行環境など

- OS : Windows 10

- Docker Desktop 4.10.1

# コンテナデータの正体はコイツだ

Docker コンテナの保存場所は ``%LOCALAPPDATA%`` 下にあります。

```powershell
> echo %LOCALAPPDATA%
C:\Users\<ユーザ名>\AppData\Local
```

筆者の環境ではユーザディレクトリ下の ``AppData\Local\Docker\`` でした。

ここの更に下の ``wsl\data`` にヤツが潜んでいます。

![SnapCrab_NoName_2023-11-18_12-33-31_No-00.png](..\..\..\assets\img\post\2023-11-19\SnapCrab_NoName_2023-11-18_12-33-31_No-00.png)

サイズはおよそ 12GB ほど。Docker は各種コンテナのデータを ext4.vhdx に保存しているんですが、これが肥大化の原因です。

# シンボリックリンクで手っ取り早く対応する

Docker ではコンテナ保管場所の変更オプションが存在しないので、今回はシンボリックリンクを生成して対応します。

## Docker Desktop を終了する

まずは、コンテナデータを移動するために Docker Desktop を終了します。「Quit Docker Desktop」をクリック。

![SnapCrab_NoName_2023-11-18_12-47-23_No-00.png](..\..\..\assets\img\post\2023-11-19\SnapCrab_NoName_2023-11-18_12-47-23_No-00.png)

もし「Docker Desktop Sercive」が裏で動いている場合は、こちらも止めておきます。

![SnapCrab_NoName_2023-11-18_12-50-29_No-00.png](..\..\..\assets\img\post\2023-11-19\SnapCrab_NoName_2023-11-18_12-50-29_No-00.png)

## ext4.vhdx を移動する

次に、別の HDD や SSD 等に退避先のフォルダを作成します。今回は ``D:\Docker\data\`` としました。ここに `Docker\wsl\data\` にある `ext4.vhdx` を移動します。

![SnapCrab_NoName_2023-11-18_12-56-37_No-00.png](..\..\..\assets\img\post\2023-11-19\SnapCrab_NoName_2023-11-18_12-56-37_No-00.png)

## シンボリックリンクを作成する

まずは管理者権限でコマンドプロンプトを開きます。次に、``Docker\wsl\data\ext4.vhdx`` を 移動先（``D:\docker\data\ext4.vhdx``）へのシンボリックリンクとして生成します。

```powershell
>mklink C:\Users\<ユーザ名>\AppData\Local\Docker\wsl\data\ext4.vhdx D:\docker\data\ext4.vhdx
C:\Users\<ユーザ名>\AppData\Local\Docker\wsl\data\ext4.vhdx <<===>> D:\docker\data\ext4.vhdx のシンボリック リンクが作成され ました
```



これで完了です。

以降、問題なく各コンテナが利用できました。
