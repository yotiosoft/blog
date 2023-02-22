---
layout: post
title: "C++で音声出力できる「SoLoud」の導入メモ（Visual C++編）"
tags: [C/C++, Windows, SoLoud]
excerpt_separator: <!--more-->
---

Albus Box により高度なサウンドエフェクトを導入したいと思い、ここ最近はオーディオ関連のライブラリを調査しています。まずは OpenSiv3D でも採用されている [SoLoud](http://solhsa.com/soloud/index.html){:target="_blank"} という C++ 向けのライブラリを試してみました。（前回、OpenSiv3D の Audio クラスに OpenAL が採用されているとか書きましたが、あれは勘違いです）  

リファレンスをざっと読んだ限り OpenSiv3D にまだ実装されていない機能、例えばイコライザ※ や 3D サウンド、FFTFilter などもサポートしています（※イコライザは開発版のみサポート）。  
Albus Box は OpenSiv3D 製ですが、今回は SoLoud 自体がサポートしている機能を求めているため、 OpenSiv3D を介さずに直接 SoLoud を弄る方向で検討しています。  

一つ難点なのが、SoLoud に関する日本語の情報が全然ありません。それどころか英語でも公式リファレンス以外の情報源があまりなく、リファレンスに掲載された少ない情報とサンプルコードを頼りに実装していくしかなさそうです。

<!--more-->

# 今回やること

- SoLoud で文章読み上げ（simplest sample）
- SoLoud で mp3 ファイルを再生
- ↑ これらを Visual C++ で実装

今回は Visual Studio で実装します。

# 参考文献

- [SoLoud](http://solhsa.com/soloud/index.html){:target="_blank"}  
  ほぼ唯一と言っていい情報源。C++ での利用方法が掲載されています。

# 利用方法

## ダウンロード

[こちら](http://solhsa.com/soloud/downloads.html){:target="_blank"} から最新の安定版をダウンロードできます。安定版にはビルド済みのサンプルプログラム（megademo など）が付属します。  

開発版は [GitHub](https://github.com/jarikomppa/soloud){:target="_blank"} から入手可能です。開発版では安定版に実装されていない機能も利用可能です。

## ファイルの include

SoLoud をダウンロードすると、中に include というディレクトリと src というディレクトリがあります。include は各種ヘッダファイルが含まれており、src には各種機能のソースファイルが含まれています。ビルドする際は両方プロジェクトに含めておく必要があります。  

include 内のヘッダファイルは機能ごとに分かれており、基本的には ``soloud.h`` と、自分が使用しようとしている機能（例えば音声ファイルの再生なら ``soloud_wav.h`` を include します。  

src 内のソースファイルは機能ごとにディレクトリが分かれており、① SoLoud のコアエンジンのソース、② 自分が使用するバックエンド用のソース、③ 自分が使用する機能のソースの3種類が必要になります。

### バックエンド

で、バックエンドとはなんぞやということなんですが、ここでは SoLoud が音声を再生するために使用する API やライブラリのことを指すようです。SoLoud は単体で音声を再生しているわけではなく、SDL や OpenAL、Windows multimedia、Linux OSS、OSX CoreAudio などなど、各種 OS やプラットフォームに実装されているバックエンドを使用して音声を再生します。つまり、これらを抽象化して簡単に使えるようにし、さらにマルチプラットフォームにてプログラムを共通化させるためのインターフェイスの役割を果たしているわけですね。  

バックエンドによって機能に違いがあるか否かは不明ですが、リファレンスを見た限り特にそのような記述は見受けられません。ただし、バックエンドごとに遅延が大きかったり（OpenAL）、まだ実験段階だったり（WASAPI）という違いはあるようです。  

それで今回は何を採用したかというと、Windows で動けばいいのでライブラリの導入が不要な Windows multimedia を採用しました。  

バックエンドを指定するには、そのバックエンドを表す定数を定義しておきます。これは [Quick Start](https://solhsa.com/soloud/quickstart.html){:target="_blank"} に書かれていますが、Windows multimedia なら ``WITH_WINMM`` となります。これを ``プロジェクト(P) → <プロジェクト名> のプロパティ(P) → C/C++ → プリプロセッサ`` にある「プリプロセッサの定義」に追加しておきます。  

![SnapCrab_NoName_2023-2-22_15-31-9_No-00](../../../assets/img/post/2023-02-22/SnapCrab_NoName_2023-2-22_15-31-9_No-00.png)

### _CRT_SECURE_NO_WARNINGS の定義

Visual C++ では ``sprintf()`` の使用は非推奨で、代わりに `sprintf_s()` を使用せよとされていますが、SoLoud はバリバリ使用しています。このままビルドしようとするとエラー扱いになります。SoLoud 側のプログラムを修正してもいいんですが、面倒なので今回はコンパイラに `sprintf()` の使用を許可させました。  

先程の手順と同様に、「プリプロセッサの定義」に ``_CRT_SECURE_NO_WARNINGS`` を追加しておけば OK です。

## インクルードディレクトリの追加

利用するに当たり、SoLoud のヘッダファイルが置かれているディレクトリをインクルードパスに追加する必要があります。`プロジェクト(P) → <プロジェクト名> のプロパティ(P) →VC++ ディレクトリ → インクルードディレクトリ` にて include ディレクトリを追加しておきましょう。  

![SnapCrab_NoName_2023-2-22_15-31-9_No-00](../../../assets/img/post/2023-02-22/SnapCrab_NoName_2023-2-22_17-54-45_No-00.png)

# 喋らせてみる

[Examples](https://solhsa.com/soloud/examples.html){:target="_blank"} にある simplest というサンプルプログラムを動かしてみます。  
自分の用途上、Visual C++ で動かす必要があるため Visual Studio で新たにプロジェクトを作成し、サンプルプログラムをコピペして動かしてみました。  

```cpp
#include "soloud/include/soloud.h"
#include "soloud/include/soloud_speech.h"
#include "soloud/include/soloud_thread.h"

// Entry point
int main(int argc, char* argv[])
{
  // Define a couple of variables
  SoLoud::Soloud soloud;  // SoLoud engine core
  SoLoud::Speech speech;  // A sound source (speech, in this case)

  // Configure sound source
  speech.setText("1 2 3   1 2 3   Hello world. Welcome to So-Loud.");

  // initialize SoLoud.
  soloud.init();

  // Play the sound source (we could do this several times if we wanted)
  soloud.play(speech);

  // Wait until sounds have finished
  while (soloud.getActiveVoiceCount() > 0)
  {
    // Still going, sleep for a bit
    SoLoud::Thread::sleep(100);
  }

  // Clean up SoLoud
  soloud.deinit();

  // All done.
  return 0;
}
```

## 実装方法

Visual C++ で実装する上で必要となった操作をまとめておきます。

### 使用するファイル

まず、このサンプルプログラムで必要なファイルは以下のとおりです。

- ヘッダファイル（*.h）
  
  - include/soloud.h
  
  - include/soloud_speech.h
  
  - include/soloud_thread.h
  
  - src/audiosource/speech 下のすべての h ファイル

- ソースファイル（*.cpp）
  
  - src/core 下のすべての cpp ファイル（for コアエンジン）
  - src/audiosource/speech 下のすべての cpp ファイル（for 読み上げ機能）
  - src/backend/winmm/soloud_winmm.cpp（for バックエンド）

これらを Visual C++ のプロジェクトに追加しておきます。

## 実行結果

実行してみたときの様子がこちら。  

<video src="../../../assets/img/post/2023-02-22/soloud1.mp4" controls></video>

かなり機械的な声で「1 2 3 1 2 3 Hello world. Welcome to So-Loud.」と読み上げられました。

# オーディオファイルを再生してみる

SoLoud では wav、flac、mp3、ogg などをサポートしており、これらは ``SoLoud::Wav`` クラスで再生できます。プログラムも非常にシンプルです。動作テスト用に自分で組んだプログラムがこちら。  

```cpp
#include "soloud/include/soloud.h"
#include "soloud/include/soloud_wav.h"

// Entry point
int main(int argc, char* argv[])
{
  // Define a couple of variables
  SoLoud::Soloud soloud;  // SoLoud engine core

  // initialize SoLoud.
  soloud.init();

  SoLoud::Wav wav;
  wav.load("littleidea.mp3");
  soloud.play(wav); // Play the wave : OK!

  getchar();

  // Clean up SoLoud
  soloud.deinit();

  // All done.
  return 0;
}
```

ここでは「[littleidea.mp3](https://www.bensound.com/royalty-free-music/track/little-idea-positive-music-for-youtube){:target="_blank"}」というフリー音源を再生しています。  

SoLoud エンジンを init して ``SoLoud::Wav`` クラスでファイルを読み込み、それを ``play()`` に渡すことで音声ファイルの再生が可能です。  

SoLoud は用が済んだら ``deinit()`` でメモリを解放してやる必要があります。このプログラムでは Enter キーを押したら ``deinit()`` を呼び出して終了するようにしています。

## 使用するファイル

このプログラムで必要なファイルは以下のとおりです。

- ヘッダファイル（*.h）
  
  - include/soloud.h
  
  - include/soloud_wav.h
  
  - src/audiosource/wav 下のすべての h ファイル

- ソースファイル（*.cpp）
  
  - src/core 下のすべての cpp ファイル（for コアエンジン）
  - src/audiosource/wav 下のすべての cpp ファイル（for ファイル再生機能）
  - src/backend/winmm/soloud_winmm.cpp（for バックエンド）

## 実行結果

実行してみたときの様子がこちら。  

<video src="../../../assets/img/post/2023-02-22/soloud2.mp4" controls></video>

# 感想

高機能かつシンプルで、イコライザやストリーミング再生など実装したい機能もサポートされているので、特に問題がなければ（かつ実装する気力が持続すれば）たぶんこれを採用すると思います。Rust や Python でも利用可能なので、別言語で利用してみるのも面白そうですね。ただ、まだ文章読み上げとファイルを再生しただけなので検証が必要そうです。もうちょっとだけ続くんじゃ～
