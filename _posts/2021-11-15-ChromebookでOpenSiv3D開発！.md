---
layout: post
title: ChromebookでOpenSiv3D開発！
tags: [Chrome OS, OpenSiv3D, C++]
excerpt_separator: <!--more-->
---

[前々回](../08/ChromebookでもTyporaが使いたい.html)の記事ではChromebookでTyporaを使えるようにしました。でも、やっぱりそれだけじゃ物足りない。敢えて軽量重視なChrome OSでゲーム開発がしたいと思ったので、今回は環境の導入からOpenSiv3Dのビルド、そしてサンプルプログラムのコンパイルまで実践してみます。

<!--more-->

# 目次

1. [実験環境](#env)
2. [方策](#plan)
3. [Crostiniの導入](#crostini)
4. [前準備](#before)
   1. [OpenCV4の導入](#opencv4)
   2. [Boostの導入](#boost)
   3. [gcc 9.3.0の導入](#gcc9.3.0)
5. [OpenSiv3Dのビルド](#opensiv3d)
6. [アプリのコンパイル](#compile)
7. [まとめ](#sum)
8. [参考文献](#cf)



<a id="env"></a>

# 実験環境

- 型番: HP ChromeBook x360
- OS: Google Chrome OS （64ビット）
  - バージョン: 94.0.4606.104
- CPU: Intel Pentium（詳細な型番は不明）
- 画面解像度: 1920×1280ピクセル
- RAM: 4GB

普段WindowsやMacのPCを触っている自分からすれば低スペックの部類に入ります。しかしこれはChromebookですので、Chromebookの中ではまあまあ標準的なスペックと言えるのではないでしょうか。  

  
インストールするものは以下の通り。  

- OpenSiv3D v0.6.3
- OpenCV 4.5.1
- Boost 1.71.0
- gcc 9.3.0
- その他、必要なライブラリなど
- （エディタ：Emacs）

<a id="plan"></a>

# 方策

さあ、早速ターミナルを起動して、OpenSiv3Dをビルドして…といきたいところですが、残念ながらそう簡単にはいきません。Ubuntuのような一般的なLinuxディストリビューションとは異なり、Chrome OSはライトユーザー層に向けて作られたブラウザが主役の軽量OSですので、わざわざこんな軽量OSでC++のコードをバリバリ書いてゲーム開発しようなんて考え出すような変態向けには作られておりません。  
しかし近年、寛大なるGoogle先生はLinux仮想環境（Crostini）をChrome OSに与えてくださりました。マイナーな開発環境のため環境の導入にひと手間かかりますが、これでChrome OSでもLinux開発が可能になりました。Google先生に感謝し、そして祈りましょう。ビルドが成功しますように。  

全体的な流れとしては以下のように行います。

1. Crostiniの導入

2. 前準備

   1. OpenCV4の導入

   1. Boost 1.71.0の導入

   1. gcc 9.3.0の導入

3. OpenSiv3Dのビルド

   1. ライブラリのインストール

   1. cmake

   1. ビルド

4. アプリのコンパイル

後半の「3. OpenSiv3Dのビルド」「4. アプリのコンパイル」は、ほぼリファレンス通りに進めればOKです。一方で前準備はかなり時間がかかります。

<a id="crostini"></a>

# Crostiniの導入

※導入済みの場合は飛ばしてください。  

Chrome OSにLinux仮想環境（Crostini）を導入します。導入は簡単で、設定画面から「Linux開発環境」をオンにするだけです。  
![](../../../assets/img/post/2021-11-15-ChromebookでOpenSiv3D開発！/Screenshot 2021-11-07 22.07.26-16369538565791.png)  
Crostini導入についての詳細は↓の記事の冒頭に書いてあります。  
[ChromebookでもTyporaが使いたい！ | 為せばnull](../08/ChromebookでもTyporaが使いたい.html)

<a id="before"></a>

# 前準備

なぜ前準備が必要かというと、OpenSiv3Dのリファレンス通りに進めてもOpenCV、Boost、gccの必要なバージョンがCrostiniに導入できないからです。CrostiniはDebianベースの仮想環境なので``apt install``でパッケージを導入できますが、``apt``で提供されているバージョンが古いのです。  
ではなぜUbuntuなら前準備無しで``apt install``で最新版が手に入るのかというと、``ubuntu-toolchain``が利用できるからです。Ubuntuであれば、最初に  

```bash
$ sudo add-apt-repository ppa:ubuntu-toolchain-r/test
```

を実行してやれば、その後はOpenCV4もBoost 1.71.0もgcc 9.3.0も普通に``apt``コマンドで導入できます。ところがこの``ubuntu-toolchain``、Crostiniでは利用できないのです。Ubuntuじゃないからね。  
ではどうするのかというと、これらのソースをダウンロードし、自前でビルドしてインストールします。面倒くさそうに思えるかもしれませんが、既に各パッケージをビルドからインストールまで自動で行ってくれるシェルスクリプトが用意されておりますので、今回はそれを利用します。必要なのはビルドにかかる時間のみです。  

ここからはターミナル上で作業を進めます。

<a id="opencv4"></a>

## OpenCV4の導入

まずはOpenCV4を導入します。前述の通り、Crostiniでは``apt``コマンドではOpenCV4は手に入りません。``sudo apt install libopencv-dev``ではOpenCV 3.2.0がインストールされます。古すぎい！  

最初に``apt``をupdate & upgrade  

```bash
$ sudo apt update
$ sudo apt upgrade
```

次に、OpenCVをインストールしてくれるシェルスクリプトをダウンロードします。  

```bash
$ cd ~
$ wget https://raw.githubusercontent.com/milq/milq/5b7c3332c78b3a2dd952321f247a72b13a3026db/scripts/bash/install-opencv.sh
$ chmod +x install-opencv.sh
$ ./install-opencv.sh
```

最後の``$ ./install-opencv.sh``で2～3時間くらいかかりました。何をしているかというと、必要なパッケージを集め、OpenCVのソースをダウンロードし、ビルドして配置するという一連のインストール作業をこれ一つで進めてくれます。  

完了したらバージョン確認。  

```bash
$ opencv_version
4.5.1
```

4.x.xが表示されたら成功です。今回は4.5.1がインストールされました。  

次に、cmake実行時に``pkg_check_modules``が``opencv4``を見つけてくれるように、``pkg-config``用のファイルを作成します。  
まずはエディタをインストール。  

```bash
$ sudo apt install emacs
```

pkgconfigのディレクトリに移動し（なければディレクトリを新規作成）、  

```bash
$ cd /usr/lib/pkgconfig
```

新たに``opencv4.pc``を作成します。  

```bash
sudo emacs opencv4.pc
```

``opencv4.pc``の中身は以下のように書き込みます。  

```
prefix=/usr/local
exec_prefix=${prefix}
includedir_old=${prefix}/include/opencv4/opencv2
includedir_new=${prefix}/include/opencv4
libdir=${exec_prefix}/lib

Name: opencv4
Description: The opencv library
Version: 4.5.1
Libs: -L${exec_prefix}/lib -lopencv_dnn -lopencv_gapi -lopencv_highgui -lopencv_ml -lopencv_objdetect -lope\
ncv_photo -lopencv_stitching -lopencv_video -lopencv_calib3d -lopencv_features2d -lopencv_flann -lopencv_vi\
deoio -lopencv_imgcodecs -lopencv_imgproc -lopencv_core
Libs.private: -ldl -lm -lpthread -lrt
Cflags: -I${includedir_old} -I${includedir_new}
```

opencvの場所は多分どの環境でも同じだと思いますが、もし違う場所に配置されていたら環境に合わせて適宜``prefix``の値を変更してください。``Version``はインストールされたバージョンをここに書き入れます。今回はOpenCV 4.5.1がインストールされたので``Version: 4.5.1``としました。  
完了したら``opencv4.pc``を保存して閉じ、以下のコマンドを実行。  

```bash
$ pkg-config –cflags opencv4
$ pkg-config –libs opencv4
```

これでcmakeがopencv4を見つけてくれるようになったはずです。以上でOpenCV4の導入は完了です。

<a id="boost"></a>

## Boostの導入

[OpenSiv3Dのリポジトリ](https://github.com/Siv3D/OpenSiv3D){:target="_blank"}のREADME.mdを見て、最新のLinux版が要求しているBoostのバージョンを確認します。今日時点（v0.6.3）では``Boost 1.71.0 - 1.73.0``となっておりましたので、今回はBoost 1.71.0を導入しました。  
ここで注意点ですが、必ずREADME.mdに記載されたバージョンを導入しましょう。現在はもっと新しいバージョンが出ていますが、最新の1.78.0ではOpenSiv3Dのビルド途中でビルドエラーが発生しビルドを完了できませんでした。  

まずはホームディレクトリにBoost 1.71.0のソースをダウンロードし解凍。  

```bash
$ wget https://boostorg.jfrog.io/artifactory/main/release/1.71.0/source/boost_1_71_0.tar.bz2
$ tar -jxvf boost_1_71_0.tar.bz2
```

解凍が完了したらBoostのインストールに移ります。  

```bash
$ cd boost_1_71_0
$ ./bootstrap.sh
$ ./b2 toolset=gcc-8 --prefix=/usr/local -j5
$ sudo ./b2 install toolset=gcc-8 --prefix=/usr/local -j5
```

最後の2つでかなり時間がかかりますので要注意。これでBoostの導入は完了です。

<a id="gcc9.3.0"></a>

## gcc 9.3.0の導入

実はC/C++コンパイラであるgccも自分でコンパイルして導入し直さなければなりません。  
[OpenSiv3Dのリポジトリ](https://github.com/Siv3D/OpenSiv3D){:target="_blank"}のREADME.mdによれば、v0.6.3のLinux版ではgcc 9.3.0が要求されていますが、例によってCrostiniの``apt install``では、古いバージョンであるgcc 8.3.0までしか手に入りません。  
OpenSiv3DではC++20の機能が多く活用されていますのでgcc8では対応していない部分が多いです。例えば、gcc 8.3.0では``char8_t``などといったOpenSiv3Dで多用されている型が定義されていません。  
実は既にgccは自動でインストールされています。しかし、当然バージョンは古くgcc 8.3.0でした。  

```bash
$ gcc --version
gcc (Debian 8.3.0-6) 8.3.0
Copyright (C) 2018 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

この古いgccは**まだ残したまま**、新しいgccである9.3.0のソースをダウンロードしてビルドします。何故残しておくかというと、新しいgccをビルドするために古いgccが必要だからです。  

まずはgccのソースをダウンロードして解凍。  

```bash
$ cd ~
$ wget http://ftp.tsukuba.wide.ad.jp/software/gcc/releases/gcc-9.3.0/gcc-9.3.0.tar.gz
$ tar xvf gcc-9.3.0.tar.gz
```

ビルド用の場所を用意。  

```bash
$ cd gcc-9.3.0
$ ./contrib/download_prerequisites
$ cd ..
$ mkdir build
$ cd build
```

今回はCとC++のコンパイラだけあれば十分なので、``enable-languages=c,c++``とします。  

```bash
$ ../gcc-9.3.0/configure --prefix=/usr/local/gcc-9.3.0 --enable-languages=c,c++ --disable-multilib --disable-bootstrap
$ make
$ sudo make install
```

自前の環境ではmakeに1時間ほど要しました。  

ビルドが完了したら、古いgccを削除します。  

```bash
$ sudo apt remove gcc
```

そして新しいgccへのシンボリックリンクを作成します。  

```bash
$ sudo ln -s /usr/local/gcc-9.3.0/bin/gcc /usr/bin/gcc
$ sudo ln -s /usr/local/gcc-9.3.0/bin/g++ /usr/bin/g++
$ sudo ln -s /usr/local/gcc-9.3.0/bin/c++ /usr/bin/c++
```

最後に動作確認。  

```bash
$ gcc --version
gcc (GCC) 9.3.0
Copyright (C) 2019 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

gcc 9.3.0が``gcc``コマンドで呼び出されるようになりました。  

これでコンパイラはgcc 9.3.0に置き換わりましたが、libstdc++が古いままです。OpenSiv3Dアプリでは実行する際に``GLIBCXX_3.4.26``が要求されますが、現時点では``GLIBCXX_3.4.26``が含まれた``libstdc++.so.6``が正しい位置に配置されていません。  

```bash
$ cd /usr/lib/x86_64-linux-gnu/
$ ls
...
libssl.so.1.1                           libxcb-shm.so.0
libstdc++.so.6                          libxcb-shm.so.0.0.0
libstdc++.so.6.0.25                     libxcb.so
libstemmer.so.0d                        libxcb.so.1
...
```

``strings``コマンドで``libstdc++.so.6``のバイナリの可読部分を表示してみます。  

```bash
$ strings libstdc++.so.6 | grep GLIBCXX
GLIBCXX_3.4
GLIBCXX_3.4.1
GLIBCXX_3.4.2
GLIBCXX_3.4.3
GLIBCXX_3.4.4
GLIBCXX_3.4.5
GLIBCXX_3.4.6
GLIBCXX_3.4.7
GLIBCXX_3.4.8
GLIBCXX_3.4.9
GLIBCXX_3.4.10
GLIBCXX_3.4.11
GLIBCXX_3.4.12
GLIBCXX_3.4.13
GLIBCXX_3.4.14
GLIBCXX_3.4.15
GLIBCXX_3.4.16
GLIBCXX_3.4.17
GLIBCXX_3.4.18
GLIBCXX_3.4.19
GLIBCXX_3.4.20
GLIBCXX_3.4.21
GLIBCXX_3.4.22
GLIBCXX_3.4.23
GLIBCXX_3.4.24
GLIBCXX_3.4.25
GLIBCXX_DEBUG_MESSAGE_LENGTH
```

ご覧の通り、従来（gcc 8.3.0）の``libstdc++.so.6``には``GLIBCXX_3.4.25``までしか含まれていません。gcc 9.3.0の``libstdc++.so.6``には``GLIBCXX_3.4.26``が同梱されますので、先程gccビルド時に生成された``libstdc++.so.6``に置き換えます。  

```bash
$ sudo rm libstdc++.so.6
$ sudo cp /usr/local/gcc-9.3.0/lib64/libstdc++.so.6 /usr/lib/x86_64-linux-gnu/
```

確認してみると、  

```bash
$ strings /usr/lib/x86_64-linux-gnu/libstdc++.so.6 | grep GLIBCXX
GLIBCXX_3.4
GLIBCXX_3.4.1
GLIBCXX_3.4.2
GLIBCXX_3.4.3
GLIBCXX_3.4.4
GLIBCXX_3.4.5
GLIBCXX_3.4.6
GLIBCXX_3.4.7
GLIBCXX_3.4.8
GLIBCXX_3.4.9
GLIBCXX_3.4.10
GLIBCXX_3.4.11
GLIBCXX_3.4.12
GLIBCXX_3.4.13
GLIBCXX_3.4.14
GLIBCXX_3.4.15
GLIBCXX_3.4.16
GLIBCXX_3.4.17
GLIBCXX_3.4.18
GLIBCXX_3.4.19
GLIBCXX_3.4.20
GLIBCXX_3.4.21
GLIBCXX_3.4.22
GLIBCXX_3.4.23
GLIBCXX_3.4.24
GLIBCXX_3.4.25
GLIBCXX_3.4.26
GLIBCXX_3.4.27
GLIBCXX_3.4.28
GLIBCXX_DEBUG_MESSAGE_LENGTH
(以下略)
```

``GLIBCXX_3.4.26``が追加されました。これでgcc 9.3.0の導入は完了です。

<a id="opensiv3d"></a>

# OpenSiv3Dのビルド

これでようやくOpenSiv3Dのビルド環境が整いました。ここからはOpenSiv3Dをビルドし、``libSiv3D.a``を生成します。ここからはほぼリファレンス通りです。  
まずはOpenSiv3Dの最新コードをダウンロード。  

```bash
$ cd ~
$ git clone https://github.com/Siv3D/OpenSiv3D.git
```

次に依存パッケージを導入。[リファレンス](https://github.com/Siv3D/OpenSiv3D/blob/main/.github/workflows/ci.yml#L26-L49){:target="_blank"}ではOpenCVとBoostのインストールも含まれていますが、これらは既に導入済みのため除いてあります。  

```bash
$ sudo apt install ninja-build
$ sudo apt install libasound2-dev
$ sudo apt install libavcodec-dev
$ sudo apt install libavformat-dev
$ sudo apt install libavutil-dev
$ sudo apt install libcurl4-openssl-dev
$ sudo apt install libgtk-3-dev
$ sudo apt install libgif-dev
$ sudo apt install libglu1-mesa-dev
$ sudo apt install libharfbuzz-dev
$ sudo apt install libmpg123-dev
$ sudo apt install libopus-dev
$ sudo apt install libopusfile-dev
$ sudo apt install libsoundtouch-dev
$ sudo apt install libswresample-dev
$ sudo apt install libtiff-dev
$ sudo apt install libturbojpeg0-dev
$ sudo apt install libvorbis-dev
$ sudo apt install libwebp-dev
$ sudo apt install libxft-dev
$ sudo apt install uuid-dev
$ sudo apt install xorg-dev
```

これを一つ一つ実行するのは面倒だという方は、依存パッケージを一括インストールするシェルスクリプトを用意しましたので、以下を実行して導入してください。  

```bash
$ wget https://yotiosoft.github.io/datacenter/sh/opensiv3d_packages_install.sh
$ chmod +x ./opensiv3d_packages_install.sh && ./opensiv3d_packages_install.sh
```


ここからビルド作業に入っていきます。 まずは``cmake``でCMakeListsを作成。 

```bash
$ cd ~/OpenSiv3D/Linux
$ mkdir build && cd build
$ cmake -GNinja -DCMAKE_BUILD_TYPE=RelWithDebInfo ..
$ cmake -DCMAKE_BUILD_TYPE=RelWithDebInfo ..
```

そしてOpenSiv3Dをビルド。Crostiniで並列コンパイルをやらせるとすぐにメモリ不足になってしまいますので、''cmake''に``-j 1``オプションを付け加えておきました。自分の環境ではこれがないと``SivGeometory2D.cpp.o``をビルドするときにメモリ不足で強制終了されてしまいます。

```bash
$ cd ../
$ cmake --build build -j 1
```

ここでも数時間ほどかかりますので気長に待ちましょう。

<a id="compile"></a>

# アプリのコンパイル

それでは、早速サンプルプログラム（Hello Siv3D）をビルドしてみましょう。とその前に、サンプルプログラムの``OpenSiv3D/Linux/App/Main.cpp``を少し修正します。  
実はLinux版のサンプルプログラムは、CIの動作確認のため初期状態では``Main.cpp``は空のMain関数が記述されており、このままビルドして実行するとウィンドウが表示されることなくすぐに実行終了します。  

```c++
/////////////////
//
//	Test code for CI
//	- 通常のアプリケーション開発時には除去してください
//
# include <Siv3D.hpp> // OpenSiv3D v0.6.3
SIV3D_SET(EngineOption::Renderer::Headless) // Non-graphical mode
void Main() { }
//
/////////////////

/*
# include <Siv3D.hpp> // OpenSiv3D v0.6.3
void Main()
{
	// 背景の色を設定 | Set background color
	Scene::SetBackground(ColorF{ 0.8, 0.9, 1.0 });
	// 通常のフォントを作成 | Create a new font
	const Font font{ 60 };
	// 絵文字用フォントを作成 | Create a new emoji font
	const Font emojiFont{ 60, Typeface::ColorEmoji };
	// `font` が絵文字用フォントも使えるようにする | Set emojiFont as a fallback
	font.addFallback(emojiFont);
	// 画像ファイルからテクスチャを作成 | Create a texture from an image file
	const Texture texture{ U"example/windmill.png" };
	// 絵文字からテクスチャを作成 | Create a texture from an emoji
	const Texture emoji{ U"🐈"_emoji };
	// 絵文字を描画する座標 | Coordinates of the emoji
	Vec2 emojiPos{ 300, 150 };
	// テキストを画面にデバッグ出力 | Print a text
	Print << U"Push [A] key";
	while (System::Update())
	{
		// テクスチャを描く | Draw a texture
		texture.draw(200, 200);
		// テキストを画面の中心に描く | Put a text in the middle of the screen
		font(U"Hello, Siv3D!🚀").drawAt(Scene::Center(), Palette::Black);
		// サイズをアニメーションさせて絵文字を描く | Draw a texture with animated size
		emoji.resized(100 + Periodic::Sine0_1(1s) * 20).drawAt(emojiPos);
		// マウスカーソルに追随する半透明な円を描く | Draw a red transparent circle that follows the mouse cursor
		Circle{ Cursor::Pos(), 40 }.draw(ColorF{ 1, 0, 0, 0.5 });
		// もし [A] キーが押されたら | When [A] key is down
		if (KeyA.down())
		{
			// 選択肢からランダムに選ばれたメッセージをデバッグ表示 | Print a randomly selected text
			Print << Sample({ U"Hello!", U"こんにちは", U"你好", U"안녕하세요?" });
		}
		// もし [Button] が押されたら | When [Button] is pushed
		if (SimpleGUI::Button(U"Button", Vec2{ 640, 40 }))
		{
			// 画面内のランダムな場所に座標を移動
			// Move the coordinates to a random position in the screen
			emojiPos = RandomVec2(Scene::Rect());
		}
	}
}
(以下略)
*/
```



冒頭部分（Test code for CIの部分）を削除し、下のMain関数のコメントアウトを外して保存します。  

```c++
# include <Siv3D.hpp> // OpenSiv3D v0.6.3
void Main()
{
	// 背景の色を設定 | Set background color
	Scene::SetBackground(ColorF{ 0.8, 0.9, 1.0 });
	// 通常のフォントを作成 | Create a new font
	const Font font{ 60 };
	// 絵文字用フォントを作成 | Create a new emoji font
	const Font emojiFont{ 60, Typeface::ColorEmoji };
	// `font` が絵文字用フォントも使えるようにする | Set emojiFont as a fallback
	font.addFallback(emojiFont);
	// 画像ファイルからテクスチャを作成 | Create a texture from an image file
	const Texture texture{ U"example/windmill.png" };
	// 絵文字からテクスチャを作成 | Create a texture from an emoji
	const Texture emoji{ U"🐈"_emoji };
	// 絵文字を描画する座標 | Coordinates of the emoji
	Vec2 emojiPos{ 300, 150 };
	// テキストを画面にデバッグ出力 | Print a text
	Print << U"Push [A] key";
	while (System::Update())
	{
		// テクスチャを描く | Draw a texture
		texture.draw(200, 200);
		// テキストを画面の中心に描く | Put a text in the middle of the screen
		font(U"Hello, Siv3D!🚀").drawAt(Scene::Center(), Palette::Black);
		// サイズをアニメーションさせて絵文字を描く | Draw a texture with animated size
		emoji.resized(100 + Periodic::Sine0_1(1s) * 20).drawAt(emojiPos);
		// マウスカーソルに追随する半透明な円を描く | Draw a red transparent circle that follows the mouse cursor
		Circle{ Cursor::Pos(), 40 }.draw(ColorF{ 1, 0, 0, 0.5 });
		// もし [A] キーが押されたら | When [A] key is down
		if (KeyA.down())
		{
			// 選択肢からランダムに選ばれたメッセージをデバッグ表示 | Print a randomly selected text
			Print << Sample({ U"Hello!", U"こんにちは", U"你好", U"안녕하세요?" });
		}
		// もし [Button] が押されたら | When [Button] is pushed
		if (SimpleGUI::Button(U"Button", Vec2{ 640, 40 }))
		{
			// 画面内のランダムな場所に座標を移動
			// Move the coordinates to a random position in the screen
			emojiPos = RandomVec2(Scene::Rect());
		}
	}
}
```

これでOKです。サンプルプログラムをビルドしてみます。

```bash
$ ./App
$ mkdir build && cd build
$ cmake -GNinja -DCMAKE_BUILD_TYPE=RelWithDebInfo ..
$ cmake --build build
```

しばらく待つと``Siv3DTest``が生成されます。これが実行ファイルです。実行してみます。  

```bash
$ ./Siv3DTest 
```

すると…  
![](../../../assets/img/post/2021-11-15-ChromebookでOpenSiv3D開発！/Screenshot 2021-11-15 12.00.00.png)  
おお！おなじみのサンプルプログラムが、ついにChrome OS上で動き始めました。カーソル操作もできるし、Simple GUIのボタンも使えます。  

v0.6から実装された3Dも問題なく動きます。  
<video src="../../../assets/img/post/chrome3d.webm" controls></video>    

画面録画しながら実行しているので動画はカクカクしていますが、実際はもっとヌルヌル動いてくれます。

<a id="sum"></a>

# まとめ

Chrome OSでもLinux仮想環境であるCrostiniを用いてOpenSiv3Dの導入からSiv3Dアプリのビルドまで実現できました。これでCities Boxやその他もろもろの開発がChromebookでもできます。CrostiniはDebianベースなので、Debianでも同様の手順で導入できるかと思います。  
次は開発用のエディタとしてChrome OSにVS Codeでも入れてみようかなと思います。

<a id="cf"></a>

# 参考文献

- OpenSiv3Dの導入
  - [Siv3D リファレンス v0.6.3](https://zenn.dev/reputeless/books/siv3d-documentation/viewer/setup#3.-linux-%E3%81%A7-siv3d-%E3%82%92%E5%A7%8B%E3%82%81%E3%82%8B){:target="_blank"}
- OpenCV4の導入
  - [Ubuntu 18.04へのOpenCV4.5.1の導入 \| RYoMa_0923 \| note](https://note.com/ryoma_0923/n/n381428d55b8f){:target="_blank"}
- Boostの導入
  - [ストレスフリーにUbuntuでc++20のために最新のg++, Cmake, Boostを導入する - Qiita](https://qiita.com/forno/items/11c4a0f8169d987f232b#boost%E3%81%AE%E3%83%80%E3%82%A6%E3%83%B3%E3%83%AD%E3%83%BC%E3%83%89%E3%81%A8%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB){:target="_blank"}
- gcc 9.3.0の導入
  - [Debian 10 (buster) - GCC 10.2.0 インストール（ソースビルド）！ - mk-mode BLOG](https://www.mk-mode.com/blog/2021/04/14/debian-installation-newest-gcc-by-src/){:target="_blank"}
