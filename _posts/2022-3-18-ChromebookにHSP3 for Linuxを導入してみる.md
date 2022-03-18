---
layout: post
title: "ChromebookにHSP3 for Linuxを導入してみる"
tags: [Chrome OS, HSP3]
feature-img: "/assets/img/feature-img/220318.png"
thumbnail: "/assets/img/thumbnails/feature-img/220318.png"
excerpt_separator: <!--more-->
---

久々のChromebook関連記事。今回はHSP、Hot Soup ProcessorのLinux版である「HSP3 for Linux」を、Chrome OSで動作するLinux仮想環境Crostiniに入れて動かしてみたいと思います。

<!--more-->  

# 使用環境

- HP ChromeBook x360
- Google Chrome OS (64bit)
  - version 99.0.4844.57
- CPU: Intel Pentium（詳細な型番は不明）
- 画面解像度: 1920x1280 px
- RAM: 4GB
- SSD: 64GB
- Crostiniのディスクサイズ: 20GB

# 導入環境

- Hot Soup Processor 3.6 (OpenHSP / HSP3 for Linux)
- OpenGLES
  - libgles2-mesa-dev 19.2.8
- SDL2
  - libsdl2-dev 2.0.9
  - libsdl2-ttf-dev 2.0.15
  - libsdl2-image-dev 2.0.4
  - libsdl2-mixer-dev 2.0.4
- gtk+-2.0
  - libgtk2.0-dev

# 1. Crostiniのインストール

以前別の記事に書いたのでこちらをご参照ください。  
[ChromebookでもTyporaが使いたい！](https://blog.yotiosoft.com/2021/11/08/Chromebook%E3%81%A7%E3%82%82Typora%E3%81%8C%E4%BD%BF%E3%81%84%E3%81%9F%E3%81%84.html)



# 2. HSP3 for Linuxをclone

以下、HSP3公式ガイドを参考に進めていきます。  
[HSP3 for Linux/Raspberry Pi](https://hsp.tv/make/hsp3linux_pi.html){:target="_blank"}  

まずはGitHubからOpenHSPのリポジトリをクローンします。  
予め作業用のディレクトリを生成しておきます。今回はホームディレクトリ直下に作りました。  

```bash
~$ mkdir openhsp
~$ cd openhsp
```

次にリポジトリをclone。  

``` bash
~/openhsp/$ git clone http://github.com/onitama/OpenHSP
```



# 3. 環境の導入

HSP3 for Linuxでは下記の環境が必要になります。  

- OpenGLES2.0以降 / EGL
- SDL2
- gtk+-2

SDLに関しては、公式ガイドにはSDL1.2と記載されているものの、OpenHSPのビルド時にSDL2が要求されたのでSDL2を導入しました。  
以下、これらを1つずつ導入していきます。

## OpenGLES

```bash
$ sudo apt update
$ sudo apt install libgles2-mesa-dev
```

## SDL2.0  

```bash
$ sudo apt install libsdl2-dev libsdl2-ttf-dev libsdl2-image-dev libsdl2-mixer-dev
```

## gtk+-2.0

```bash
$ sudo apt install libgtk2.0-dev  
```

これでライブラリの導入は完了です。

# 4. OpenHSPのmake

いよいよOpenHSPをmakeしてみます。  

まずはOpenHSPのディレクトリへ移動して

```bash
~/openhsp/$ cd OpenHSP/
```

makeコマンドを実行。

```bash
~/openhsp/OpenHSP$ make
```

makeを実行すればあとは勝手にやってくれます。自分の環境では7分ほど要しました。これでHSP3 for Linuxの導入は完了です。



# サンプルプログラムを動かしてみる

``OpenHSP/samples/block3.hsp``がブロック崩しのサンプルプログラムですので、これをCrostini上でコンパイルして動かしてみます。  

まずはhspcmpで``block3.hsp``から``block3.ax``を生成。

```bash
~/openhsp/OpenHSP $./hspcmp -d -i -u ./sample/block3.hsp -o./block3.ax
#HSP script preprocessor ver3.6 / onion software 1997-2021(c)
#Use file [hspdef.as]
#Use file [dish_enhance.as]
#Use file [dish_enhance.as]
#HSP code generator ver3.6 / onion software 1997-2021(c)
#use UTF-8 strings.
#Uninitalized variable (key).
#Uninitalized variable (bsize).
#Code size (1860) String data size (307) param size (0)
#Vars (38) Labels (8) Modules (0) Libs (0) Plugins (3)
#No error detected. (total 3593 bytes)
```

これをhsp3dishに渡し、ランタイムを実行。  

```bash
~/openhsp/OpenHSP$ ./hsp3dish ./block3.ax
Init:HGIOScreen(640,480)
```

すると…おおっと？  
![image-20220318132722249](../../../assets/img/post/2022-2-14-ChromebookにHSP3 for Linuxを導入してみる　/image-20220318132722249.png)

一応起動しました。が、ウィンドウサイズが合わず、表示の一部が途切れてしまっています。  

もう一度コマンドラインを見てみます。すると、  

```bash
~/openhsp/OpenHSP$ ./hsp3dish ./block3.ax
Init:HGIOScreen(640,480)
```

ウィンドウのサイズが(640, 480)に設定されていることが確認できます。  
[HSP3Dishのドキュメント](http://www.onionsoft.net/hsp/v35/doclib/hsp3dish_prog.htm#INIFILE){:target="_blank"}によれば、``hsp3dish.ini``で画面サイズが設定できるようです。  
なぜかsampleディレクトリ内に``hsp3dish.ini``が置いてあったので、これを``hsp3dish``の実行可能ファイルが置かれている場所にコピーし、もう一度実行してみます。

![image-20220318150750076](../../../assets/img/post/2022-2-14-ChromebookにHSP3 for Linuxを導入してみる　/image-20220318150750076.png)  
なお、``hsp3dish.ini``は初期で画面サイズが(360,640)に設定されています。  

```
; hsp3dish settings
wx=360
wy=640
autoscale=0
```


これでもう一度``hsp3dish``でランタイムを実行。すると、  

```bash
~/openhsp/OpenHSP$ ./hsp3dish ./block3.ax
Init:HGIOScreen(360,640)
```

![image-20220318151001990](../../../assets/img/post/2022-2-14-ChromebookにHSP3 for Linuxを導入してみる　/image-20220318151001990.png)  
キター！  
無事、縦長の画面サイズでHSP3のブロック崩しが起動しました。  

<video src="../../../assets/img/post/2022-2-14-ChromebookにHSP3 for Linuxを導入してみる　/Screen-recording-2022-03-18-15.11.45.mp4" controls></video>

Crostini上でも快適に動作します。

# エディタを起動

せっかくなのでスクリプトエディタも起動してみます。OpenHSPディレクトリにある``./hsed``がスクリプトエディタです。

```bash
~/openhsp/OpenHSP$ ./hsed
```

![image-20220318155930052](../../../assets/img/post/2022-3-18-ChromebookにHSP3 for Linuxを導入してみる/image-20220318155930052.png)  
こちらも無事起動しました。せっかくなので、先程のブロック崩しとは別のプログラムを動かします。sampleフォルダの``button_test.hsp``をエディタにコピペし実行してみます。  
![image-20220318160315264](../../../assets/img/post/2022-3-18-ChromebookにHSP3 for Linuxを導入してみる/image-20220318160315264.png)  
``HSP -> コンパイル+実行``を選ぶとエディタに書いたプログラムが実行されます。  
![image-20220318160343121](../../../assets/img/post/2022-3-18-ChromebookにHSP3 for Linuxを導入してみる/image-20220318160343121.png)  
すると、  
![image-20220318160625827](../../../assets/img/post/2022-3-18-ChromebookにHSP3 for Linuxを導入してみる/image-20220318160625827.png)  
こちらも実行できました。

![image-20220318160550357](../../../assets/img/post/2022-3-18-ChromebookにHSP3 for Linuxを導入してみる/image-20220318160550357.png)  
ボタンやダイアログも利用できます。URLは開けませんでした。

# おわりに

HSP3 for LinuxがChrome OSのCrostini上でも動作することが確認できました。HSP for Linuxは従来のHSP3に比べると若干機能に制約がありますが、HSP3Dish向けにプログラムを作ればWindows、AndroidやiOS、ブラウザ、Respberry Piと同様のソースコードでChromebook上でも動作できるのは素晴らしいことです。今回はトラブルも少なく、すんなり動いてくれました。手軽に実現できるので、Chrome OS環境をお持ちの方はぜひ試してみてください。  
wineを使ってWindows版のHSP3を動かす方法もあるので、気が向いたらそれも後日試してみます。

