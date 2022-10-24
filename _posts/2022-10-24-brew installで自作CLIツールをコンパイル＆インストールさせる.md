---
layout: post
title: "brew installで自作CLIツールをコンパイル＆インストールさせる"
tags: [Rust, Mac, Homebrew]
excerpt_separator: <!--more-->
---

最近、Rust で CLI ツールを作っており、ある程度できてきたのでそろそろリリースしようかなと思い。今までと同じような単なる実行ファイル配布ではなく、パッケージマネージャからインストールできたら導入が便利かなと思い、Mac 版と Linux 版は Homebrew での配布を検討しています。  

今回は Homebrew の公式リポジトリに上げるのではなく、``brew tap``で自分のリポジトリからインストールできるようにしています。 

<!--more-->  

この記事では、自分のリポジトリに Formula を置いておき、brew コマンドでソースファイルをダウンロード＆``cargo install``でコンパイルとインストールができるようにするまで実践してみます。  
事前に開発者側でコンパイルしておき、バイナリで配布して brew にインストールさせることも可能ですが、今回はとりあえずユーザ側でコンパイルさせる方法でいきます（アーキテクチャごとにコンパイルしなきゃいけないのが面倒だもの）。

# 実験環境

- MacBook Air 2020
  - Chip: Apple M1
  - RAM: 8GB
  - Homebrew 3.6.6-34-gfaa9950


# リリースするもの

dptran という Rust 製の自作 CLI ツール。DeepL API をコマンドラインから使えるようにしたツールです。今記事が初登場ですが、まだリリース前だし本題から逸れるので dptran についての解説はまた後日。

# 1. GitHub でリリース

リリースする物が置いてある GitHub リポジトリの [Releases] から [Create a new release] を選び、リリース画面を開く。  
そしてリリース。すると、ソースファイルが入った tar.gz ファイルが自動で生成されます。  
![スクリーンショット 2022-10-24 16.37.56](../../../assets/img/post/2022-10-24/スクリーンショット 2022-10-24 16.37.56.png)

# 2. Formula 用のリポジトリ作成

Homebrew のバージョン管理用のリポジトリを新たに GitHub に作成しておきます。  
名前は「homebrew-[自作ツール名]」（今回はhomebrew-dptran）とします。  
![スクリーンショット 2022-10-22 0.35.51](../../../assets/img/post/2022-10-24/スクリーンショット 2022-10-22 0.35.51.png)

# 3 Formula 用のファイルを作成

先に、リリースした zip ファイルの URL を GitHub のリリースページからリンクをコピペして取得しておきます。  
![スクリーンショット 2022-10-24 16.15.23](../../../assets/img/post/2022-10-24/スクリーンショット 2022-10-24 16.15.23.png)  
ローカルにも homebrew-dptran のリポジトリを作り、Formula の設定ファイルを作成します。  
``brew create [ファイルのURL]``で GitHub リポジトリの情報を基に自動生成してくれます。

```zsh
% mkdir homebrew-dptran && cd homebrew-dptran
% brew create --rust --tap yotiosoft/dptran -v https://github.com/YotioSoft/dptran/archive/refs/tags/ver.0.1.0.tar.gz  
```

すると、自動作成された Formula ファイルが開かれます。  
必要に応じて修正します（自分の場合は修正不要でした）。

```ruby
# Documentation: https://docs.brew.sh/Formula-Cookbook
#                https://rubydoc.brew.sh/Formula
# PLEASE REMOVE ALL GENERATED COMMENTS BEFORE SUBMITTING YOUR PULL REQUEST!
class Dptran < Formula
  desc "A tool to run DeepL translations on command line (for my Rust studies)."
  homepage ""
  url "https://github.com/YotioSoft/dptran/archive/refs/tags/ver.0.1.0.tar.gz"
  sha256 "eb5d154e0b2c76b1542647d21e3802653c8715bdae82f1a847ae9a1d7010afdd"
  license ""

  depends_on "rust" => :build

  def install
    # ENV.deparallelize  # if your formula fails when building in parallel
    system "cargo", "install", *std_cargo_args
  end

  test do
    # `test do` will create, run in and delete a temporary directory.
    #
    # This test will fail and we won't accept that! For Homebrew/homebrew-core
    # this will need to be a test that verifies the functionality of the
    # software. Run the test with `brew test dptran`. Options passed
    # to `brew install` such as `--HEAD` also need to be provided to `brew test`.
    #
    # The installed folder is not in the path, so use the entire path to any
    # executables being tested: `system "#{bin}/program", "do", "something"`.
    system "false"
  end
end
```

このファイルは``/opt/homebrew/Library/Taps/homebrew/homebrew-core/Formula/``に保存されます。  
ローカルリポジトリの Formula ディレクトリに移動しておきましょう。  

```zsh
% mkdir Formula
% mv /opt/homebrew/Library/Taps/homebrew/homebrew-core/Formula/dptran.rb Formula
```

最終的にはこんな構成になります。  

```
homebrew-dptran
|- Formula
   |- dptran.rb
```


これでOK。git push しておきます。

```zsh
% git remote add origin GitHub:YotioSoft/dptran.git
% git add .
% git commit -m "add Formula file"
% git push origin main
```

# 4 動作確認

実際に brew install してみます。まずはリポジトリの tap から。

```zsh
% brew tap yotiosoft/dptran
Running `brew update --auto-update`...
==> Auto-updated Homebrew!
==> Updated Homebrew from 2bec76052 to 3.6.6 (faa995022).
Updated 2 taps (homebrew/core and homebrew/cask).

You have 46 outdated formulae installed.
You can upgrade them with brew upgrade
or list them with brew outdated.


The 3.6.0 release notes are available on the Homebrew Blog:
  https://brew.sh/blog/3.6.0
The 3.6.6 changelog can be found at:
  https://github.com/Homebrew/brew/releases/tag/3.6.6
==> Tapping yotiosoft/dptran
Cloning into '/opt/homebrew/Library/Taps/yotiosoft/homebrew-dptran'...
remote: Enumerating objects: 32, done.
remote: Counting objects: 100% (32/32), done.
remote: Compressing objects: 100% (18/18), done.
remote: Total 32 (delta 7), reused 30 (delta 5), pack-reused 0
Receiving objects: 100% (32/32), done.
Resolving deltas: 100% (7/7), done.
Tapped 1 formula (13 files, 9.8KB).

```

次に install。  

```
% brew install dptran
==> Downloading https://ghcr.io/v2/homebrew/core/ca-certificates/manifests/2022-
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/ca-certificates/blobs/sha256:1b
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sh
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/rust/manifests/1.64.0
######################################################################## 100.0%
==> Downloading https://ghcr.io/v2/homebrew/core/rust/blobs/sha256:6fb6dbcd5c3ec
==> Downloading from https://pkg-containers.githubusercontent.com/ghcr1/blobs/sh
######################################################################## 100.0%
==> Downloading https://github.com/YotioSoft/dptran/archive/refs/tags/ver.0.1.0.
Already downloaded: /Users/ytani/Library/Caches/Homebrew/downloads/6c6de106c93fe7f70e371c823629aede32c582a167fef6c08309fd40af4b79f6--dptran-ver.0.1.0.tar.gz
==> Installing dptran from yotiosoft/dptran
==> Installing dependencies for yotiosoft/dptran/dptran: ca-certificates and rust

（後略）
```

依存環境のインストールとコンパイルが始まります。  
しばらく待ち、インストールが完了したら``dptran``コマンドを実行。  

```
% dptran
Welcome to dptran!
First, please set your DeepL API-key:
  $ dptran -c api-key [YOUR_API_KEY]
You can get DeepL API-key for free here:
  https://www.deepl.com/ja/pro-api?cta=header-pro-api/
(base) ytani@ytaninoMacBook-Air homebrew-dptran % dptran 1c664a9f-4696-d92d-1caa-b4a3634ec562:fx
Welcome to dptran!
First, please set your DeepL API-key:
  $ dptran -c api-key [YOUR_API_KEY]
You can get DeepL API-key for free here:
  https://www.deepl.com/ja/pro-api?cta=header-pro-api/
```

うまくいきました。パスも通ってます。homebrew で導入できるとかなり楽ですね。  

## 問題点

ただ、コンパイルのために依存環境の導入も必要で、Python だの ffmpeg だの emacs だの余計なものまでインストールさせられ、インストール開始から完了までかかった時間は30分ほど。長すぎぃ！  

ユーザ側でコンパイルさせると依存環境の導入に異常に時間がかかると分かったので、可能な限りリリース時に開発者側でコンパイルしておいて、brewにはバイナリファイルのダウンロードとインストールだけしてもらうようにしておいたほうが現実的です。  
この方法については後日記事に書こうかと思います。
