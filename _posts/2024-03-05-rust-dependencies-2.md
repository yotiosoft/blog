---
layout: post
title: "dev-dependenciesでRustのexampleとライブラリの依存関係を分ける"
tags: [Rust, rusimg]
excerpt_separator: <!--more-->
---

前回の記事では、feature を用いてバイナリクレートとライブラリクレートの dependencies を分けました。

[Rustでバイナリとライブラリでdependenciesを分ける \| 為せばnull](https://blog.yotiosoft.com/2024/02/23/rust-dependencies.html)

今回はもう一つの手法として、dev-dependencies を用いる方法を書き残したいと思います。

<!--more-->

なお、前回はバイナリクレートでしたが、今回の方法は、アプリケーション（ライブラリではない方）を**バイナリクレートとしてではなく example として**扱わないと利用できない点に注意してください。便宜上、以降はアプリケーションと呼ぶことにします。

正直言うと筆者はバイナリクレートと example との違いはよく分かっていないのですが、バイナリはクレートのメインアプリケーションのことを、example はライブラリクレートの使用例としてのデモプログラムのことを示すようです。

ですので、アプリケーションが中心で、あくまでもライブラリはおまけ程度であるクレートならばバイナリクレート、ライブラリが中心でアプリケーションがおまけ程度であるクレートならば examples として扱うのが良いのかな？と思っていますが、このあたりの位置づけは正直よくわかりません。

結局どちらもアプリケーションとしてのビルドが可能で、``cargo install`` でコマンドとしてインストールすることが可能ですので、動作上はほとんど変化はないと思います。ただ、アプリケーションを example として扱った場合、クレートのメインはライブラリとなります。ユーザは ``cargo install`` 実行時に ``--example`` オプションで example をインストールすることを明記する必要があります。

# 結論

アプリケーション（バイナリ）側だけが使用する依存関係を ``[dev-dependencies]`` として Cargo.toml に定義します。

`[dev-dependencies]` はテスト実行、examples のビルド、ベンチマーク実行時のみ依存関係に含まれることを示すもので、ライブラリのビルド時には依存関係には含まれません。よって、ライブラリから不要な外部クレートを除外することができます。

手順は以下の通りです。

- アプリケーションを examples ディレクトリに移動する

- アプリケーションを Cargo.toml に example として定義する

- アプリケーションだけが使用する依存関係を ``[dependencies]`` ではなく ``[dev-dependencies]`` に定義する

## 1. アプリケーションを examples ディレクトリに移動する

アプリケーションのソースコードである ``main.rs`` を ``examples/[example名]`` に移動することで、アプリケーションは examples として扱われます。example 名はアプリケーションの名前でも付けておきましょう。

すなわち、

```
rust-app
├ src
│ ├ main.rs
│ └ lib.rs
└ Cargo.toml  
```

これを

```
rust-app
├ examples
│ └ rust-app
│ 　 └ main.rs
├ src
│ └ lib.rs
└ Cargo.toml    
```

こうします。

## 2. アプリケーションを Cargo.toml に example として定義する

```toml
[[examples]]
name = "rust-app"
```

Cargo.toml に ``[[examples]]`` を追記しておき、``name`` に先ほどのアプリケーション名を設定しておきます。

## 3. アプリケーションだけが使用する依存関係を `[dev-dependencies]` に定義する

もともと ``[dependencies]`` に定義されていた、アプリケーションだけが依存してライブラリが依存しない外部クレートを、``[dev-dependencies]`` に移動します。

例えば、``[dependencies]`` が

```toml
[dependencies]
image = "0.24.5"
mozjpeg = { version = "0.9.4", optional = true }
oxipng = { version = "8.0.0", optional = true }
dep_webp = { version = "0.2.2", optional = true, package = "webp" }
clap = { version = "4.1.8", features = ["derive"] }
regex = { version ="1.7.2" }
viuer = { version ="0.6.2" }
glob = { version = "0.3.1" }
colored = { version = "2.0.4" }
```

このようになっていて、下5つ、すなわち clap、regex、viuer、glob、colored がアプリケーションでしか使用しない外部クレートであった場合、

```toml
[dependencies]
image = "0.24.5"
mozjpeg = { version = "0.9.4", optional = true }
oxipng = { version = "8.0.0", optional = true }
dep_webp = { version = "0.2.2", optional = true, package = "webp" }

[dev-dependencies]
clap = { version = "4.1.8", features = ["derive"] }
regex = { version ="1.7.2" }
viuer = { version ="0.6.2" }
glob = { version = "0.3.1" }
colored = { version = "2.0.4" }
```

このようにします。

これで完了です。

# 動作確認

ここからは、自分が現在開発しているアプリケーション「[rusimg](https://github.com/yotiosoft/rusimg){:target"_blank"}」を例に動作確認してみます。

まずは ``cargo install --git https://github.com/yotiosoft/rusimg`` でインストールしようとした場合の結果を見てみます。すると、

```bash
$ cargo install --git https://github.com/yotiosoft/rusimg
    Updating git repository `https://github.com/yotiosoft/rusimg`
error: there is nothing to install in `rusimg v0.1.0 (https://github.com/yotiosoft/rusimg#18a34951)`, because it has no binaries
`cargo install` is only for installing programs, and can't be used with libraries.
To use a library crate, add it as a dependency to a Cargo project with `cargo add`.
```

このようなエラーが現れました。``cargo install`` はライブラリインストール用のコマンドではないから、ライブラリを追加したいなら ``cargo add`` を使え、という趣旨のエラーです。

それでは、空の Rust のクレートディレクトリに移動して、rusimg を外部クレートとして追加してみます。すると、

```bash
$ cargo add --git https://github.com/yotiosoft/rusimg
    Updating git repository `https://github.com/yotiosoft/rusimg`
    Updating git repository `https://github.com/yotiosoft/rusimg`
      Adding rusimg (git) to dependencies.
             Features:
             + bmp
             + dep_webp
             + jpeg
             + mozjpeg
             + oxipng
             + png
             + webp
    Updating git repository `https://github.com/yotiosoft/rusimg`
    Updating crates.io index
```

こちらでは、clap などといった、``[dev-dependencies]`` に登録したクレートを除き、ライブラリが必要とする外部クレートだけがインストールされたようするが確認できました。

続いて、examples をインストールしてみます。``cargo install --git https://github.com/yotiosoft/rusimg --examples rusimg`` でインストールできます。

```bash
$ cargo install --git https://github.com/yotiosoft/rusimg --examples rusimg
    Updating git repository `https://github.com/yotiosoft/rusimg`
  Installing rusimg v0.1.0 (https://github.com/yotiosoft/rusimg#18a34951)
    Updating crates.io index
  Downloaded mio v0.8.11
  Downloaded tempfile v3.10.1
  Downloaded half v2.4.0
  Downloaded regex-automata v0.4.6
  Downloaded rayon v1.9.0
  Downloaded 5 crates (982.6 KB) in 1.21s
   Compiling libc v0.2.153
 ...
```

今度は examples がビルドされている様子が確認できました。インストール完了後、コマンドとして rusimg を実行できました。

```bash
$ rusimg
 2 images are detected.
[1/2] Processing: test.bmp
[2/2] Processing: test_save.bmp

✅ All images are processed.
```

# 前回の features で dependencies を分ける方法との違い

クレート開発者側の手数で言えば、別途 features を定義する必要がない分、今回の examples を利用する方法のほうが手軽だと思います。

ただ、examples に指定するとライブラリが中心のクレートとして扱われますので、``cargo install`` 時に ``--example [アプリケーション名]`` の指定が必要になります。

一方、アプリケーションをバイナリクレートして扱い、features で dependencies を分けた場合も ``--features=app`` といった、アプリケーション用の外部クレートの feature 名を指定する必要があります。

利用者側の視点で言えばどちらも利便性に差異はないでしょう。

# 参考文献

- [rust - How can I specify binary-only dependencies? - Stack Overflow](https://stackoverflow.com/questions/35711044/how-can-i-specify-binary-only-dependencies){:target="_blank"}

- [開発中の依存関係 - Rust By Example 日本語版](https://doc.rust-jp.rs/rust-by-example-ja/testing/dev_dependencies.html){:target="_blank"}
