---
layout: post
title: "MacのVSCodeでGitHub Copilotを使ってOpenSiv3D開発したい"
tags: [Mac, OpenSiv3D, VS Code]
excerpt_separator: <!--more-->
---

最近、主にWindowsでの開発時にGitHub Copilotを利用するようになって、Macでもこれを使いたい。しかしGitHub CopilotはXcodeは非対応。VS Codeなら使えるけども、OpenSiv3DはXcodeでのビルドが前提条件です。  
そこで、エディタはVS Codeを使い、ビルドだけXcodeにやってもらうことに。ビルドのためにいちいちXcodeを開くのは面倒なので、VS CodeにOpenSiv3Dアプリをビルドするタスクを実行してもらうことで、VS Codeでビルドから実行まで自動的にできるようにしてみます。

<!--more-->  

# 概要

## 動作確認環境

- MacBook Air 2020 (13-inch)
  - chip: Apple M1
  - RAM: 8GB
  - VS Code、Xcode、Xcode CommandLineTools、Rosetta2 インストール済み
- OpenSiv3D v0.6.4
- Visual Studio Code ver.1.68.1
  - C/C++, C/C++ Extension Pack, C/C++ Themes, GitHub Copilot インストール済み


# 実現方法

Macには``xcodebuild``コマンドがあり、このコマンドを用いてXcodeのプロジェクトをビルドできます。これを実行するタスクをVS Codeに登録しておき、さらに、ビルドしたアプリを自動で開くタスクを実行させれば、VS Codeでビルドからタスクまでの一通りの処理がVS Code上でも可能なはずです。  
VS Codeはinclude pathを登録することで補完もできますので、include pathの設定もしておきます。最後に、GitHub Copilotを試用してみます。

# 前準備

## Command Line Toolsの指定

``xcodebuild``コマンドを使う前に、Command Line Toolsの選択が必要です。Xcodeを開き、メニューバーの[Xcode] > [Preferences]を開いて、[Locations]タブを開きます。  
ここで、[Command Line Tools:]が空欄になっている場合は、プルダウンメニューをクリックし、Xcodeのバージョンを指定します。  
![スクリーンショット 2022-06-30 22.19.32](../../../assets/img/post/2022-7-1/スクリーンショット 2022-06-30 22.19.32.png)

## xcodebuildコマンドについて

事前にCommandLineToolsをインストールしておく必要があります。  

利用方法として、Xcodeの現在の設定でビルドするのであれば、ターミナルのカレントディレクトリをOpenSiv3DのXcodeプロジェクトファイル（empty.xcodeproj）がある場所に移動して

```zsh
% xcodebuild
```

を実行すればOK。  

ただし、M1やM2などのApple Siliconの場合はこれではビルドできません。現時点でOpenSiv3DはApple Siliconには非対応で、そのままビルドしようとするとライブラリがarm64に対応していない云々のエラーが出て失敗してしまいます。  
![スクリーンショット 2022-06-30 22.06.21](../../../assets/img/post/2022-7-1/スクリーンショット 2022-06-30 22.06.21.png)  

そこで、ビルドターゲットのアーキテクチャを指定します。  

普段どおりにXcodeでビルドするのであれば、画面上部の「My Mac」を「My Mac (Rosetta2)」に変更してやればx86_64アーキテクチャでビルドされ、Rosetta2で実行が可能です。  
![スクリーンショット 2022-06-30 22.09.50](../../../assets/img/post/2022-7-1/スクリーンショット 2022-06-30 22.09.50.png)  
![スクリーンショット 2022-06-30 22.10.04](../../../assets/img/post/2022-7-1/スクリーンショット 2022-06-30 22.10.04.png)  


これをコマンドラインツールである ``xcodebuild``で指定するにはどうすればいいかというと、``ARCHS``オプションに``x86_64``を指定してやります。  

```zsh
% xcodebuild ARCHS=x86_64
```

これでx86_64アーキテクチャ向けにビルドされますので、Rosetta2を使えばApple Silicon上でもビルド＆動作が可能です。  
![スクリーンショット 2022-06-30 22.20.43](../../../assets/img/post/2022-7-1/スクリーンショット 2022-06-30 22.20.43.png)  

また、DebugビルドとReleaseビルドの指定も可能で、それぞれオプションとして``-configuration Debug``、``-configuration Release``をコマンドライン引数に追加します。

## コマンドでappファイルを開くには？

``open``コマンドでappファイルのパスを指定することで開けますが、これだけでは普通にアプリを実行するだけなので、アプリのコンソール出力の内容（OpenSiv3DのDebugやConsoleなどで表示させるもの）がVS Codeからは見えません。  
appファイルの実態はディレクトリなので、Xcodeでビルドしたときのようにコンソール出力を表示したい場合は、appファイルの中に置かれているバイナリファイルを直接開きます。  

例）empty.appのバイナリファイルを開く

```zsh
% ./App/empty.app/Contents/MacOS/empty
```



# VS Codeでのビルドタスクの設定

ここからが本題です。先に示したコマンドを活用して、VS Code上に一発でビルドできるためのタスクを登録します。  

まずはVS CodeでXcodeプロジェクトファイル（empty.xcodeproj）がある場所を開きます。  
そしてCmd + shift + Pで[Tasks: Run Tasks]を選択。  
![スクリーンショット 2022-06-30 22.51.23](../../../assets/img/post/2022-7-1/スクリーンショット 2022-06-30 22.51.23.png)  

さらに「タスクを構成する」を選び、  
![スクリーンショット 2022-06-30 22.52.22](../../../assets/img/post/2022-7-1/スクリーンショット 2022-06-30 22.52.22.png)    
「テンプレートからtasks.jsonを生成」を選択。  
![スクリーンショット 2022-06-30 22.52.26](../../../assets/img/post/2022-7-1/スクリーンショット 2022-06-30 22.52.26.png)  

「Others」を選びます。  
![スクリーンショット 2022-06-30 22.54.41](../../../assets/img/post/2022-7-1/スクリーンショット 2022-06-30 22.54.41.png)  

自動でtasks.jsonが生成されます。  
tasks.jsonの内容はこんな感じにしておきます。  

```json
{
    // See https://go.microsoft.com/fwlink/?LinkId=733558
    // for the documentation about the tasks.json format
    "version": "2.0.0",
    "tasks": [
        {
            "label": "debug-build",
            "type": "shell",
            "command": "xcodebuild",
            "args": [
                "ARCHS=x86_64",
                "-configuration",
                "Debug"
            ],
        },
        {
            "label": "debug-run",
            "type": "shell",
            "command": "open",
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "dependsOn": [
                "debug-build"
            ],
            "args": [
                "App/empty.app"
            ]
        },
        {
            "label": "release-build",
            "type": "shell",
            "command": "xcodebuild",
            "args": [
                "ARCHS=x86_64",
                "-configuration",
                "Release"
            ],
        },
        {
            "label": "release-run",
            "type": "shell",
            "command": "open",
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "dependsOn": [
                "release-build"
            ],
            "args": [
                "App/empty.app"
            ]
        }
    ]
}
```

Debugビルド用とReleaseビルド用にそれぞれbuildタスクとbuild-runタスクを作成しました。buildタスクは``xcodebuild``を呼び出し、build-runタスクはbuildタスクに加え、ビルドしたバイナリを開き実行します。  

``build-run``をビルドのショートカット（Cmd + shift + B）が押された際に実行するようにしています。  
このとき、debug用（debug-run）とrelease用（release-run）の2つの選択肢から選んでビルドします。  
![スクリーンショット 2022-07-01 18.56.25](../../../assets/img/post/2022-7-1/スクリーンショット 2022-07-01 18.56.25.png)

## IntelliSenseの設定

現時点ではライブラリのヘッダにパスが通っておらず、補完ができないので、ライブラリへのinclude pathをIntelliSenseに設定しておきます。赤い波線が表示されているincludeのコード部分をクリックして電球マークを選択肢、「\"includePath\"設定の編集」を選択。  
![スクリーンショット 2022-07-01 0.25.57](../../../assets/img/post/2022-7-2/スクリーンショット 2022-07-01 0.25.57.png)

  

「コンパイラ パス」の入力欄にXcodeで使用されるコンパイラ、つまりclangのファイルパスを入力します。  
このファイルパスは``% which clang``で得られます。  

```zsh
% which clang
/usr/bin/clang
```


![スクリーンショット 2022-07-01 19.01.15](../../../assets/img/post/2022-7-1/スクリーンショット 2022-07-01 19.01.15.png)  

次に、「パスを含める」の入力欄にOpenSiv3Dのヘッダファイルが置かれている2つのディレクトリパスを追記。  

```
${workspaceFolder}/../../include/
${workspaceFolder}/../../include/ThirdParty/
```

![スクリーンショット 2022-07-01 19.01.45](../../../assets/img/post/2022-7-1/スクリーンショット 2022-07-01 19.01.45.png)


これで補完ができるはずです。試しにコードを追加してみます。  
![スクリーンショット 2022-07-01 20.09.29](../../../assets/img/post/2022-7-1/スクリーンショット 2022-07-01 20.09.29.png)

できました。Summaryも表示されています。また、ヘッダファイルのエラー表示も消えました。  
![スクリーンショット 2022-07-01 19.02.15](../../../assets/img/post/2022-7-1/スクリーンショット 2022-07-01 19.02.15.png)


ただ、PrintやConsoleのオペランド(<<)に赤い波線が出現。  
![スクリーンショット 2022-07-01 18.41.02](../../../assets/img/post/2022-7-1/スクリーンショット 2022-07-01 18.41.02.png)  
うーん、IntelliSenseのコード解析がヘッダファイルからPrintやConsoleのオペランドを認識できていないのだろうか。とりあえず動作に影響はないので置いておきます。

# ビルドしてみる

<video src="../../../assets/img/post/2022-7-1/画面収録-2022-07-01-19.55.54.mp4" controls></video>

Ctrl+shift+Bを押してdebug-build-runを選択すると、ビルドが始まりアプリが起動しました。コンソール出力もできています。

# GitHub Copilotを試す

最後に、GitHub Copilotが動くか見てみます。  
<video src="../../../assets/img/post/2022-7-1/画面収録-2022-07-01-20.13.55.mp4" controls></video>

動きました。ときどきOpenSiv3Dに存在しない関数を提案してきたりしますが、思った以上に精度は良さげです。

# おわりに

PrintやConsoleなどにエラーが表示されてしまう点が少し気になりますが、動作上は問題なく、それ以外に特に問題点はなさそうでした。また何か進展があったら追記します。
