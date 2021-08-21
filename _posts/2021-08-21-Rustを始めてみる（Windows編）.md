---
layout: post
title: Rustを始めてみる（Windows編）
tags: [Rust]
excerpt_separator: <!--more-->
---

[前回](../20//Rust%E3%82%92%E5%A7%8B%E3%82%81%E3%81%A6%E3%81%BF%E3%82%8B-Mac%E7%B7%A8.html)に引き続き、Rustの環境構築、今回はWindows編です。

<!--more-->



# エディタについて

Macと同じくVisual Studio Code（以下VS Code）を利用します。



# Rustの環境構築に必要なモノ

- rustc  
  

  コンパイラ

- cargo  
  

  パッケージマネージャ

- rustup  
  
  
  Rustのインストーラ

# 参考文献

[Windows で Rust 用の開発環境を設定する \| Microsoft Docs](https://docs.microsoft.com/ja-jp/windows/dev-environment/rust/setup){:target="_blank"}

# Windowsでの環境構築

今回はRustインストーラを利用します。  
手順はだいたいこんな感じ。

1. C++ビルドツールをインストールする
2. rustupでRustをインストールする
3. VS Codeに拡張機能「rust-analyzer」をインストールする
4. デバッグのため、VS Codeに拡張機能「CodeLLDB」をインストールする



## 1.C++ビルドツールをインストールする

Microsoft Docsによれば、Visual StudioをインストールすればC++ビルドツールがついてくるとのこと。  
あるいは、Microsoft C++ Build Toolsをインストールする必要あり。  
今回は既にVisual Studioをインストール済みなのでここは省略。



## 2. rustupでRustをインストールする

[Rust公式サイト](https://www.rust-lang.org/ja/tools/install){:target="_blank"}からrustup_init.exeをダウンロードしインストールする。  
32bit版と64bit版がある。  
基本的に指示に従って進めればOK。  
このとき、cargoへのパスも自動で通してくれます。



## 3. VS Codeに拡張機能「rust-analyzer」をインストールする

rust-analyzerはRustのコード補完をしてくれる拡張機能。  
![SnapCrab_NoName_2021-8-21_11-51-18_No-00](../../../assets/img/post/2021-08-21-Rustを始めてみる（Windows編）/SnapCrab_NoName_2021-8-21_11-51-18_No-00.png)


VS Codeを開いて、rust-analyzerを検索しインストール。



## 3. デバッグのため、VS Codeに拡張機能「CodeLLDB」をインストールする

同様にして、VS CodeにCodeLLDBをインストール。  
今回は使いません。



# Hello World!

まずはCargoでRustプロジェクトを作成します。  
VS Codeでターミナル（Power Shell）を開いて  
![SnapCrab_ようこそ - rust_practice - Visual Studio Code_2021-8-21_11-57-51_No-00](../../../assets/img/post/2021-08-21-Rustを始めてみる（Windows編）/SnapCrab_ようこそ - rust_practice - Visual Studio Code_2021-8-21_11-57-51_No-00.png)  
ターミナルが表示されたら、以下のコマンドを入力。

```powershell
> cargo init <プロジェクト名>
```

今回はプロジェクト名はhello_worldとしました。  
![SnapCrab_NoName_2021-8-21_11-58-23_No-00](../../../assets/img/post/2021-08-21-Rustを始めてみる（Windows編）/SnapCrab_NoName_2021-8-21_11-58-23_No-00.png)  


例によって自動的にHello Worldプログラムが用意されます。  
![SnapCrab_NoName_2021-8-21_11-58-44_No-00](../../../assets/img/post/2021-08-21-Rustを始めてみる（Windows編）/SnapCrab_NoName_2021-8-21_11-58-44_No-00.png)  
あとは「Run」をクリックするとコンパイルして実行してくれます。  
![SnapCrab_NoName_2021-8-21_11-58-44_No-00](../../../assets/img/post/2021-08-21-Rustを始めてみる（Windows編）/SnapCrab_NoName_2021-8-21_11-58-44_No-00-16295149098553.png)  

実行されました。WindowsでもRustの実行環境の構築が完了しました。  
![SnapCrab_NoName_2021-8-21_11-58-56_No-00](../../../assets/img/post/2021-08-21-Rustを始めてみる（Windows編）/SnapCrab_NoName_2021-8-21_11-58-56_No-00-16295150163085.png)



# まとめ

Windows版VS CodeでRustが実行できた。  
Windowsでの環境構築は面倒なイメージがあるけど、意外に簡単でした。  
次回からはRustの学習に入っていきたいと思います。
