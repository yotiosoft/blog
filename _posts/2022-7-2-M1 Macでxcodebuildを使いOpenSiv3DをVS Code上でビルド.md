---
layout: post
title: "M1 Macでxcodebuildを使いOpenSiv3DをVS Code上でビルド"
tags: [Mac, OpenSiv3D]
excerpt_separator: <!--more-->
---

最近、GitHub Copilotを利用し始めた。しかしGitHub CopilotはXcode非対応。VS CodeやWindowsのVisual Studioなら使えるけども、OpenSiv3D for Macは現状Xcodeでのビルドが必須です。そこでエディタはVS Codeを使い、ビルドだけXcodeにやってもらうことに。しかし、ビルドのためにいちいちXcodeを開くのは面倒。  
というわけで今回は、``xcodebuild``コマンドでOpenSiv3Dアプリをビルドし、VS Codeでビルドから実行までできるようにしてみます。

<!--more-->  

# 注意

書き終わってから気付いたのですが、既に先に実現していた方がいらっしゃいました。  
実現方法は異なりますが、こちら↓のほうが実用的です。  
[Xcodeを開かずにOpenSiv3Dを使いたい - Qiita](https://qiita.com/makia/items/3188b08670f178104f6d){:target="_blank"}  

既に先人がいらっしゃったのか〜。まあいっか。せっかく書いたので投稿させていただきます。

# 概要

## 動作確認環境

- MacBook Air 2020 (13-inch)
  - chip: Apple M1
  - RAM: 8GB
  - VS Code、Xcode、Xcode CommandLineTools、Rosetta2 インストール済み
- OpenSiv3D v0.6.4

# 前準備

## Command Line Toolsの指定

``xcodebuild``コマンドを使う前に、Command Line Toolsの選択が必要です。Xcodeを開き、メニューバーの[Xcode] > [Preferences]を開いて、[Locations]タブを開きます。  
ここで、[Command Line Tools:]が空欄になっている場合は、プルダウンメニューをクリックし、Xcodeのバージョンを指定します。  
![スクリーンショット 2022-06-30 22.19.32](../../../assets/img/post/2022-7-1-XcodebuildでOpenSiv3DをVS Code上でビルド/スクリーンショット 2022-06-30 22.19.32.png)

## xcodebuildコマンドについて

Xcodeプロジェクトをビルドするためのコマンドラインツールで、ターミナルから利用可能です。事前にCommandLineToolsをインストールしておく必要があります。  

利用方法として、Xcodeの現在の設定でビルドするのであれば、ターミナルのカレントディレクトリをOpenSiv3DのXcodeプロジェクトファイル（empty.xcodeproj）がある場所に移動して

```zsh
% xcodebuild
```

を実行すればOK。  

ただし、M1やM2などのApple Siliconの場合はこれではビルドできません。現時点でOpenSiv3DはApple Siliconには非対応で、そのままビルドしようとするとライブラリがarm64に対応していない云々のエラーが出て失敗してしまいます。  
![スクリーンショット 2022-06-30 22.06.21](../../../assets/img/post/2022-7-1-XcodebuildでOpenSiv3DをVS Code上でビルド/スクリーンショット 2022-06-30 22.06.21.png)  

そこで、ビルドターゲットのアーキテクチャを指定します。  

普段どおりにXcodeでビルドするのであれば、画面上部の「My Mac」を「My Mac (Rosetta2)」に変更してやればx86_64アーキテクチャでビルドされ、Rosetta2で実行が可能です。  
![スクリーンショット 2022-06-30 22.09.50](../../../assets/img/post/2022-7-1-XcodebuildでOpenSiv3DをVS Code上でビルド/スクリーンショット 2022-06-30 22.09.50.png)  
![スクリーンショット 2022-06-30 22.10.04](../../../../../Desktop/sc/スクリーンショット 2022-06-30 22.10.04.png)  


これをコマンドラインツールである ``xcodebuild``で指定するにはどうすればいいかというと、``ARCHS``オプションに``x86_64``を指定してやります。  

```zsh
% xcodebuild ARCHS=x86_64
```

これでx86_64アーキテクチャ向けにビルドされますので、Rosetta2を使えばApple Silicon上でもビルド＆動作が可能です。  
![スクリーンショット 2022-06-30 22.20.43](../../../assets/img/post/2022-7-1-XcodebuildでOpenSiv3DをVS Code上でビルド/スクリーンショット 2022-06-30 22.20.43.png)

## コマンドでappファイルを開くには？

``open``コマンドを使います。  
例）

```zsh
% open App/empty.app
```



# 手順

ここからが本題です。先に示したコマンドを活用して、VS Code上に一発でビルドできるためのタスクを登録します。  

まずはVS CodeでXcodeプロジェクトファイル（empty.xcodeproj）がある場所を開きます。  
そしてCmd + shift + Pで[Tasks: Run Tasks]を選択。  
![スクリーンショット 2022-06-30 22.51.23](../../../assets/img/post/2022-7-1-XcodebuildでOpenSiv3DをVS Code上でビルド/スクリーンショット 2022-06-30 22.51.23.png)  

さらに「タスクを構成する」を選び、  
![スクリーンショット 2022-06-30 22.52.22](../../../assets/img/post/2022-7-1-XcodebuildでOpenSiv3DをVS Code上でビルド/スクリーンショット 2022-06-30 22.52.22.png)    
「テンプレートからtasks.jsonを生成」を選択。  
![スクリーンショット 2022-06-30 22.52.26](../../../assets/img/post/2022-7-1-XcodebuildでOpenSiv3DをVS Code上でビルド/スクリーンショット 2022-06-30 22.52.26.png)  

![スクリーンショット 2022-06-30 22.54.41](../../../assets/img/post/2022-7-1-XcodebuildでOpenSiv3DをVS Code上でビルド/スクリーンショット 2022-06-30 22.54.41.png)  
「Others」を選びます。  

tasks.jsonの内容はこんな感じにしておきます。  

```json
{
    // See https://go.microsoft.com/fwlink/?LinkId=733558
    // for the documentation about the tasks.json format
    "version": "2.0.0",
    "tasks": [
        {
            "label": "build",
            "type": "shell",
            "command": "xcodebuild",
            "args": [
                "ARCHS=x86_64"
            ],
        },
        {
            "label": "run",
            "type": "shell",
            "command": "open",
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "dependsOn": [
                "build"
            ],
            "args": [
                "App/empty.app"
            ]
        }
    ]
}
```

``build``では``xcodebuild``を使ってプロジェクトをビルドします。``run``では、さらにビルドしたappファイルを実行します。  
``run``をビルドのショートカット（Cmd + shift + B）が押された際に実行するようにしています。  
OpenSiv3Dのデフォルトの実行ファイル名（``App/empty.app``）に合わせて作成していますので、ファイルパスは適宜自身のファイル名に変更してください。

# 実行

<video src="../../../assets/img/post/2022-7-1-XcodebuildでOpenSiv3DをVS Code上でビルド/画面収録-2022-06-30-23.02.02.mp4" controls></video>

とりあえずビルドと実行が自動でできました。  
ライブラリのパスが通ってないのと補完できないのが気になります。まだまだ工夫の余地はありそうです。
