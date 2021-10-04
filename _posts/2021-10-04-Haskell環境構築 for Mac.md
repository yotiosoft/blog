---
layout: post
title: Haskell環境構築 for Mac
tags: [Haskell, Mac]
excerpt_separator: <!--more-->
---

必要に迫られHaskellをMacにインストールしたので、ついでに書き残しておきます。

<!--more-->

# 使用環境

- MacBook Pro 2016 Late（13-inch / TouchBarなし）
- OS: macOS Big Sur 11.5.2
- RAM: 8GB



# stackについて

Haskellの環境導入にはstackというツールを用います。stackでは以下のことができます：

- Haskellコンパイラのインストール
- Haskellプロジェクトの作成＆ビルド
- Haskellで作られたツールをビルド＆インストール


インストール手順としては大まかにはこんな感じです。

1. stackをインストールする
2. stackでHaskellのコンパイラや基本パッケージをインストールする



# インストール手順

## 1. stackをインストールする

Homebrewを用います。

```bash
$ brew install stack
```

特に指定がなければ最新版がインストールされます。今回はhaskell-stack--2.7.3（52.6MB）がインストールされました。

## 2. Haskellコンパイラ＆基本パッケージのインストール

stackのインストールに成功したらHaskellのコンパイラ等をインストールします。

```bash
$ stack setup
```

を実行すればあとはHaskellのコンパイラなど勝手にインストールしてくれます。今回はghc-8.10.7（261.34MB）がインストールされました。  
インストールにかかった時間はダウンロード時間も含めて5分ほど。  

```bash
Installed GHC.    
stack will use a sandboxed GHC it installed
For more information on paths, see 'stack path' and 'stack exec env'
To use this GHC and packages outside of a project, consider using:
stack ghc, stack ghci, stack runghc, or stack exec
```

``stack path``や``stack exec env``でパスが確認できます。

# プロジェクト作成と実行

## プロジェクト作成

stackではコマンド一発でプロジェクトファイルを作成してくれます。  
今回はプロジェクト名は``test``としました。

```bash
$ stack new test
```

![スクリーンショット 2021-10-04 21.58.00](../../../assets/img/post/スクリーンショット 2021-10-04 21.58.00.png)   


初期状態では``someFunc``が呼び出されるよう記述されています。  
App/Main.hs:

```haskell
module Main where

import Lib

main :: IO ()
main = someFunc
```

``someFunc``は``src/Lib.hs``に定義されており、"someFunc"と表示するよう記述されています。  
src/Lib.hs:

```haskell
module Lib
    ( someFunc
    ) where

someFunc :: IO ()
someFunc = putStrLn "someFunc"
```

## ビルド

次にビルドしてみます。

```bash
$ cd ./test
$ stack build
```

```bash
test> build (lib + exe)
Preprocessing library for test-0.1.0.0..
Building library for test-0.1.0.0..
Preprocessing executable 'test-exe' for test-0.1.0.0..
Building executable 'test-exe' for test-0.1.0.0..
[2 of 2] Compiling Main
Linking .stack-work/dist/x86_64-osx/Cabal-3.2.1.0/build/test-exe/test-exe ...
test> copy/register
Installing library in /Users/[...]/Git/haskell_practice/test/.stack-work/install/x86_64-osx/66d30bce50b3ff51749694c7852ae701264b0db40a65be86f7a302b7a608b522/8.10.7/lib/x86_64-osx-ghc-8.10.7/test-0.1.0.0-3THkSfOydX69Whsh6Y7VyR
Installing executable test-exe in /Users/[...]/Git/haskell_practice/test/.stack-work/install/x86_64-osx/66d30bce50b3ff51749694c7852ae701264b0db40a65be86f7a302b7a608b522/8.10.7/bin
Registering library for test-0.1.0.0..
```

実行ファイルはtest-exeという名前でビルドされているようです。

## 実行

最後に実行してみます。

```bash
$ stack exec test-exe
```

```bash
someFunc
```

"someFunc"と表示されました。正常に動作しているようです。



# おわりに

まだHaskellどころか関数型言語自体入門したばかりで、これまで触れてきた命令型言語とは大きく異なります。ある程度習得したらなにか作ってみたいなと思ってます。  

後でWindowsにもインストールしますが、一つの記事にまとめてしまうと読みづらくなりそうなので別記事にします。

