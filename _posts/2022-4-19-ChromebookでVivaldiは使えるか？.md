---
layout: post
title: "ChromebookでVivaldiは使えるか？"
tags: [Chrome OS]
feature-img: "/assets/img/feature-img/220419.png"
thumbnail: "/assets/img/thumbnails/feature-img/220419.png"
excerpt_separator: <!--more-->
---

少し前、ブラウザをChromeからVivaldiに乗り換えました。2段タブスタックやウェブパネルなど、便利な機能が目白押しで今や手放せません。ブックマークなどはVivaidi側で同期しているので、当然、他の機種でもVivaudiが使いたくなるもの。  
残念ながらiOSやiPadOSには現時点で未対応なのでこちらは諦めるしかありませんが、Google Chromeブラウザが主体のChromebookでは使えるのでしょうか？  
結論から言うと使えます。機種のサポート状況にもよりますが、Android版とLinux版の両方がChrome OS上でも利用可能です。

<!--more-->

# 目次

- [実験環境](#env)
- [方法1: Android版をインストールする](#android)
- [方法2: Linux版をインストールする](#linux)
- [おわりに](#conclusion)

<a id="env"></a>

# 実験環境

- HP ChromeBook x360
  - Google Chrome OS (64bit)
  - version 99.0.4844.94
  - CPU: Intel Pentium（詳細な型番は不明）
  - 画面解像度: 1920x1280 px
  - RAM: 4GB
  - SSD: 64GB
  - Crostiniのディスクサイズ: 20GB

<a id="android"></a>

# 方法1: Android版をインストールする

## 前提条件

- Playストアが利用可能であること。機種によっては使えないものもあります

## 手順

Playストアを開き、Vivaidiを検索。  
![](../../../assets/img/post/2022-4-10-ChromebookでVivaldiは使えるか？/Screenshot 2022-04-10 11.40.12.png)
出てきたら「インストール」を選択。これだけで完了です。  

インストールが完了すると起動できます。まずは初期設定。  
![](../../../assets/img/post/2022-4-10-ChromebookでVivaldiは使えるか？/Screenshot 2022-04-10 11.44.10.png)  
初期設定が完了すると、いつものスタートページが現れました。  
![](../../../assets/img/post/2022-4-10-ChromebookでVivaldiは使えるか？/Screenshot 2022-04-10 11.46.22.png)  
ただ、やはりAndroid版ということあってか、出てくるサイトは皆モバイル端末向けのサイトです。  
![image-20220410114904966](../../../assets/img/post/2022-4-10-ChromebookでVivaldiは使えるか？/image-20220410114904966.png)

動画は正常に再生できました。  

ちなみにモバイル版サイトについてですが、設定画面からデスクトップ版サイトの読み込みに切替可能です。  
![image-20220410115429458](../../../assets/img/post/2022-4-10-ChromebookでVivaldiは使えるか？/image-20220410115429458.png)

## タブスタックは使える？

タブスタック機能はAndroid版にも実装されていました。  
![](../../../assets/img/post/2022-4-10-ChromebookでVivaldiは使えるか？/Screenshot 2022-04-10 11.58.26.png)  
＋を長押し→「新しいタブスタックを作成する」で利用可能です。  
![image-20220410120050407](../../../assets/img/post/2022-4-10-ChromebookでVivaldiは使えるか？/image-20220410120050407.png)  

## ウェブパネルは使える？

サイドパネル自体はあるのですが、ウェブパネルは使えないっぽい？  
ウェブパネルを追加する項目が見当たりませんでした。  
![image-20220410120203756](../../../assets/img/post/2022-4-10-ChromebookでVivaldiは使えるか？/image-20220410120203756.png)

ブックマークはちゃんと同期できてました。

## Android版の良い点

- Playストアから簡単に導入できる
- 広告ブロック、タブスタックなど主要な機能は使える
- デスクトップ版表示に切替可能
- Chrome OSの日本語IMEが利用できる

## Android版の欠点

- ときどきチラつく（たぶんChrome OS側の問題）
- タッチ操作前提の設計なので、マウス操作だと若干戸惑うかも（長押し操作など）
- PC版に比べると機能は少ない

たまにチラつくのが少し気になりました。タブバーあたりが一瞬暗くなることが多いです。基本的には快適に利用できます。

<a id="linux"></a>

# 方法2: Linux版をインストールする

## 前提条件

- Linux開発環境（Crostini）が利用可能であること。機種によっては使えないものもあります
- Linux用の日本語入力環境を導入していること。導入していない場合、日本語入力が使えません

Linux開発環境と日本語入力環境の導入手順については↓の記事をご覧ください。  
[ChromebookでもTyporaが使いたい！ \| 為せばnull](https://blog.yotiosoft.com/2021/11/08/Chromebook%E3%81%A7%E3%82%82Typora%E3%81%8C%E4%BD%BF%E3%81%84%E3%81%9F%E3%81%84.html)

## 手順

まずはChrome OSの設定→「詳細設定」→「デベロッパー」から「Linux開発環境」を有効化します。  

有効化できたら、[Vivaldiのホームページ]へ行き、Linuxペンギンのアイコンをクリックします。このとき、トップページで「Vivaldiをダウンロード」を選ぶとWindows版がダウンロードされてしまうので注意。  
ダウンロードページが開かれたら、Linux版を選択します。Crostiniは中身がDebianなので、DEB版を選びます。インテルCPUなら「Linux DEB 64bit」を、ARM CPUなら「Linux DEB Arm64」を選ぶといいと思います。  
![image-20220410121224658](../../../assets/img/post/2022-4-10-ChromebookでVivaldiは使えるか？/image-20220410121224658.png)　　

ダウンロードが完了したら.debファイルを開きインストール。  

インストール完了後、Vivaidiを開いてみます。  
![image-20220410121512593](../../../assets/img/post/2022-4-10-ChromebookでVivaldiは使えるか？/image-20220410121512593.png)  

起動したら、まずは初期設定。  
![](../../../assets/img/post/2022-4-19-ChromebookでVivaldiは使えるか？/Screenshot 2022-04-10 12.15.53.png)  
設定完了後にブラウジング。  
![image-20220419213500417](../../../assets/img/post/2022-4-19-ChromebookでVivaldiは使えるか？/image-20220419213500417.png)  
YouTubeも普通に開けました。動画も再生できます。  
![image-20220410121932076](../../../assets/img/post/2022-4-10-ChromebookでVivaldiは使えるか？/image-20220410121932076.png)  

## タブスタックは使える？

使えました。~~ただ、Windows版やMac版のようにタブを別のタブにドラッグ＆ドロップするだけでは二段目のタブバーは出現しませんでした。~~  
~~Linux版でタブスタックを利用する場合、タブバーの＋ボタンを右クリック→「二段目のタブバーを固定」をクリック→二段目のタブバーに何らかのタブを追加する→もう一度「二段目のタブバーを固定」をクリックして解除、という手順を踏まなければならず、少し面倒です。~~  
![](../../../assets/img/post/2022-4-19-ChromebookでVivaldiは使えるか？/Screenshot 2022-04-19 21.10.51.png)  
![](../../../assets/img/post/2022-4-19-ChromebookでVivaldiは使えるか？/Screenshot 2022-04-19 21.11.06-16503708591331.png)  
![](../../../assets/img/post/2022-4-19-ChromebookでVivaldiは使えるか？/Screenshot 2022-04-19 21.11.18.png)  
~~Ubuntuで試したときも同様だったので、Linux版自体がこのような仕様なのかもしれません。~~  
**追記：**  
WindowsやMacと同様の操作でタブスタックが利用できました。操作方法を少し勘違いしてました。  
タブを持ち上げ、別のタブの上にドラッグ＆ドロップすることで二段目が出現します。

## ウェブパネルは使える？

使えました。Windows版やMac版と同様に利用可能です。  
![](../../../assets/img/post/2022-4-10-ChromebookでVivaldiは使えるか？/Screenshot 2022-04-10 12.21.21.png)  

## Linux版の良い点

- PC版のほぼすべての機能が利用可能
- カスタマイズ性が抜群
- PC向けなので他OS版のVivaldiと遜色なく使える
- マウス操作しやすい

## Linux版の欠点

- 重い。結構重い（多分Crostini上で動かしているのが原因）
- 導入が難しい
- Chrome OSのIMEは使えない（Linux仮想環境上に別途、日本語入力環境の導入が必須）

<a id="conclusion"></a>

# おわりに

速さ重視ならAndroid版、機能重視ならLinux版といった感じだと思います。どちらの方法でも仮想環境上で動かす必要がある以上、ネイティブアプリであるChromeブラウザほどの動作の速さ・不具合の少なさ・手軽さの実現は難しいというのが現状かと思われます。Vivaldi自体は優れたブラウザですが、Chrome OS上で日常的に動かすには仮想環境の機能面での課題点がいくつか見受けられました。 

