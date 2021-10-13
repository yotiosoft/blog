---
layout: post
title: Haskell環境構築 for Windows
tags: [Haskell, Windows]
excerpt_separator: <!--more-->
---

10日間空いてしまいましたが、今回は[前々回](../04/2021-10-04-Haskell環境構築 for Mac.md)のMacにHaskellをインストールした記事に引き続き、HaskellをWindowsにインストールしていきたいと思います。

<!--more-->

# 使用環境

- OS: Windows10 Home 21H1(19043.1237 / 64bit)
- CPU: Intel Core i5-6500 @ 3.20GHz
- RAM: 16GB



# stackについて

Haskellの環境導入にはstackというツールを用います（他の手法もあります）。stackでは以下のことができます：

- Haskellコンパイラのインストール
- Haskellプロジェクトの作成＆ビルド
- Haskellで作られたツールをビルド＆インストール


インストール手順としては大まかにはこんな感じです。

1. stackをインストールする
2. stackでHaskellのコンパイラや基本パッケージをインストールする



# インストール手順

## 1. stackをインストールする

stackをダウンロードしインストールします。  
[https://github.com/commercialhaskell/stack/releases](https://github.com/commercialhaskell/stack/releases){:target="_blank"}  
今回は``stack-2.7.3-windows-x86_64-installer.exe``をダウンロード。特に理由がなければ最新版を選びましょう。  
インストーラに従って進めていけばすぐにインストールは終わります。パスも通してくれました。

## 2. Haskellコンパイラ＆基本パッケージのインストール

stackのインストールに成功したらHaskellのコンパイラ等をインストールします。  
コマンドプロンプトで実行します。

```powershell
> stack setup
```

を実行すればあとはHaskellのコンパイラなど勝手にインストールしてくれます。今回はghc-8.10.7（414.01MiB）がダウンロードされました。  
インストールにかかった時間はダウンロード時間も含めて4分ほど。  
途中、空き容量不足で失敗したのでやり直し。ファイル展開とインストール含め6GBほど要します（インストールされるコンパイラなどのサイズは3GBほど）。  
![SnapCrab_NoName_2021-10-14_0-25-29_No-00](../../../assets/img/post/2021-10-14-Haskell環境構築 for Windows/SnapCrab_NoName_2021-10-14_0-25-29_No-00.png)

# プログラム実行

プロジェクトを作成して、ビルドして…という動作は[前々回](../04/2021-10-04-Haskell環境構築 for Mac.md)の記事でやったので、今回はghciを試してみます。ghciとはHaskellの対話形式のコンパイラで、1行ずつコマンドを実行できます。  

```powershell
> stack ghci
```

```powershell
Prelude>
```

試しに式を入力。  

```haskell
Prelude> (*) 167 2
```

```
334
```

うまく動いているようです。



# おわりに

今まさにHaskellを講義で習っていますが、Rustを独学したときによく分からなかった点（例えば束縛の概念）が理解できるようになって面白い。Rustは命令形言語ですが、部分的に関数型プログラミングの考え方も取り入れているんだなと分かりました。今後は並行して勉強していきたいと思います。

