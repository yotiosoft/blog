---
layout: post
title: "Rustでバイナリとライブラリでdependenciesを分ける"
tags: [Rust, rusimg]
excerpt_separator: <!--more-->
---

半年ほど前に記事にした画像変換ツール rusimg ですが、以前はバイナリのアプリケーションのみ開発していました。開発が進むにつれて各機能の利便性も上がり、「ライブラリにしたらもっと便利じゃね？」と思い、最近は rusimg のライブラリ化にも着手しています。

現状、rusimg はバイナリ（アプリケーション、つまりツールそのもの）とライブラリ（他のプロジェクトから rusimg の各機能が使えるようにしたもの）が混在するプロジェクトで、Rust の様式通り、バイナリは main.rs、ライブラリは lib.rs に実装しています。

ただ、アプリケーション側は rusimg ライブラリ（以下、librusimg）の基本機能に加え、

- clap による引数解析

- regex による（引数に渡された）正規表現の解析

- glob による正規表現に合致するファイルパスの一覧の取得

- viuer によるコマンドラインでの画像表示

- colored によるコマンドライン文字の色付け

…といった外部クレートを利用しています。これらのクレートはアプリケーションの実装には必要ですが、librusimg には不要です。

しかしながら、Cargo.toml に dependencies としてこれらのクレートを含めると、たとえ librusimg の利用者が cargo install で**ライブラリだけ**をインストールしようとしても、バイナリに必要な余分な外部クレートまでインストールしてしまいます。これでは容量とリソースの無駄遣いです。

よって今回は、これら外部クレートについてはバイナリをインストールしたい場合だけインストールし、ライブラリだけをインストールしたい場合はこれらのクレートを cargo のインストール対象から外すような依存関係を作っていきたいと思います。

<!--more-->

# 結論

以下の手順で行います。

- バイナリを bin として Cargo.toml に定義する

- バイナリ用の feature を定義する

- バイナリだけが使う外部クレートを optional（一分機能だけが使うクレート）にする

- 定義した feature をバイナリに対して適用する

## 1. バイナリを bin として Cargo.toml に定義する

Rust では、main.rs は bin クレート、ライブラリは lib クレートとして識別されます。これは Cargo.toml に定義しなくとも自動的に定義されています。なので定義する、というよりかは明文化するといったほうが正しいでしょうか。

なぜ明文化する必要があるかというと、後ほど外部クレートの依存関係をバイナリのみに対して定義するためです。

Cargo.toml に下記を追記します。

```toml
[[bin]]
name = "rusimg"
path = "src/main.rs"
required-features = ["app"]
```

## 2. バイナリ用の feature を定義する

cargo には feature という概念があります。これはクレートをコンパイルするとき、同じクレート内の特定の機能だけを有効にしたり、機能の有効化・無効化に沿って別々の動作をサせたりすることができるというものです。

- [Features - The Cargo Book](https://doc.rust-lang.org/cargo/reference/features.html#the-features-section){:target="_blank"}

- [[Rust] フィーチャーフラグの使い方 #Rust - Qiita](https://qiita.com/osanshouo/items/43271813b5d62e89d598){:target="_blank"}

feature を定義すると何が嬉しいかというと、feature ごとに依存クレートを定義することができます。

rusimg で例えるなら、rusimg は bmp、jpeg、png、webp の4形式に対応しており、それぞれ次の外部クレートに依存しています。

- bmp : image 0.24.5

- jpeg : image 0.24.5, mozjpeg 0.9.4

- png : image 0.24.5, oxipng 8.0.0

- webp : image 0.24.5, webp 0.2.2

もしクレート利用者が bmp の機能だけを使いたい場合、mozjpeg、oxipng、webp は無駄になってしまいます。よって、bmp の実装に対して bmp という feature を予め定義しておき、利用者が bmp feature だけを有効にすることで、mozjpeg、oxipng、webp といった不必要なクレートのコンパイルを回避できます。

これはバイナリに対しても同様に定義可能です。まずは ``[features]`` に、バイナリ用の feature として ``app`` を定義しておきます（名前は何でも OK です）。

```toml
[features]
app = ["clap", "regex", "viuer", "glob", "colored"]
```

このとき、clap、regex、viuer、glob、colored といった、先程述べた、バイナリだけが使用する外部クレートを ``[]`` 内に列挙しておきます。

## 3. バイナリだけが使う外部クレートを optional にする

Cargo.toml の ``[dependencies]`` に定義したクレートは、通常、無条件に依存関係に組み込まれ、プロジェクトのビルド時に同時にコンパイルされます。

これを防止し、必要な feature が要求されているときだけ依存関係に組み込むように設定するのが ``optional`` です。

``[dependencies]`` のうち、バイナリだけが使用する外部クレートに対して、``optional = true`` を定義します。

```toml
clap = { version = "4.1.8", features = ["derive"], optional = true }
regex = { version ="1.7.2", optional = true }
viuer = { version ="0.6.2", optional = true }
glob = { version = "0.3.1", optional = true }
colored = { version = "2.0.4", optional = true }
```

これにて、``[features]`` でこれらのクレートが要求された feature、すなわり ``app`` feature のインストール時のみ、これらのクレートが依存関係に組み込まれるようになりました。

## 4. 定義した feature をバイナリに対して適用する

最初に定義した bin に対し、``app`` feature を適用します。

```toml
[[bin]]
name = "rusimg"
path = "src/main.rs"
required-features = ["app"]
```

これにて完了です。

# 動作確認

まずは ``cargo install --git https://github.com/yotiosoft/rusimg`` でライブラリのみをインストールした場合のビルド内容を見てみます。

※ 現状、app は optional な feature として定義されているので、勝手にバイナリがインストールされることはありません

```bash
$ cargo install --git https://github.com/yotiosoft/rusimg
...
$ cargo tree
rusimg v0.1.0 (/home/ytani/git/rusimg)
├── image v0.24.5
│   ├── ...
├── mozjpeg v0.9.4
│   ├── ...
├── oxipng v8.0.0
│   ├── ...
└── webp v0.2.2
    ├── ...
```

行数の関係上、表示内容を省略していますが、rusimg が直接依存するパッケージとしては image, mozjpeg, oxipng, webp のみがインストールされていることが確認できます。

次に、``features="app"`` を付け加えてバイナリも一緒にインストールしてみます。

```bash
$ cargo install --git https://github.com/yotiosoft/rusimg --features="app"
...
$ cargo tree
rusimg v0.1.0 (/home/ytani/git/rusimg)
├── clap v4.5.1
│   ├── ...
├── colored v2.1.0
│   └── lazy_static v1.4.0
├── glob v0.3.1
├── image v0.24.9
│   ├── ...
├── mozjpeg v0.9.8
│   ├── ...
├── oxipng v8.0.0
│   ├── ...
├── regex v1.10.3
│   ├── ...
├── viuer v0.6.2
│   ├── ...
└── webp v0.2.6
    ├── ...
```

こちらでは clap, colored, glob, regex, viuer といった、バイナリだけが依存する外部クレートも同時にコンパイルされていることが確認できました。

# app をデフォルトでインストールさせる

ただ、バイナリをインストールする場合にいちいち ``features="app"`` と付け加えるのは面倒です。その場合は default features に ``app`` を含めてしまいます。

```toml
[features]
default = ["app"]
app = ["clap", "regex", "viuer", "glob", "colored"]
```

ただ、こうすると、下記のようにdependencies に rusimg を含めた場合に、

```toml
[dependencies]
rusimg = { git = "https://github.com/yotiosoft/rusimg.git" }
```

デフォルトでバイナリ用の依存クレートまで一緒にコンパイルされてしまいます。

これを回避するためには、わざわざ

```toml
[dependencies]
rusimg = { git = "https://github.com/yotiosoft/rusimg.git", default-features = false}
```

…と表記して、default feature を無効化しておく必要があります。

どちらが便利かは微妙です。バイナリに重きを置くなら default に含めて、ライブラリに重くを置くなら含めない、といった形になりますかね。

## 余談

``cargo install`` では ``--bin`` オプションでインストールするバイナリを指定することもできます。ただ、``--bin`` オプションを指定しても、そのバイナリが依存する feature は自動ではインストールされないようです。

```bash
$ cargo install --git https://github.com/yotiosoft/rusimg --bin rusimg
error: failed to compile `rusimg v0.1.0 (https://github.com/yotiosoft/rusimg#5dee37ca)`, intermediate artifacts can be found at `/tmp/cargo-installzyLqh2`.
To reuse those artifacts with a future compilation, set the environment variable `CARGO_TARGET_DIR` to that path.

Caused by:
  target `rusimg` in package `rusimg` requires the features: `app`
  Consider enabling them by passing, e.g., `--features="app"`
```

やはり ``--features="app"`` を明記しろ、と言ってきます。必要なことは明白なのに面倒ですね。

これに関しては過去にも、「bin が要求した feature は自動でインストールするべきか」という議論がされていたようですが…

- [Automatically enable required-features when running an example · Issue #4663 · rust-lang/cargo · GitHub](https://github.com/rust-lang/cargo/issues/4663){:target="_blank"}

ここでは、バイナリをインストールする時、自動的に feature を有効化する ``enable-features`` オプションが提案されています。ただ、現時点では実現はしていないようですし、issue が立てられてから6年経った今も open のままです。

# 参考文献

- [rust - How can I specify binary-only dependencies? - Stack Overflow](https://stackoverflow.com/questions/35711044/how-can-i-specify-binary-only-dependencies){:target="_blank"}

- [Features - The Cargo Book](https://doc.rust-lang.org/cargo/reference/features.html#the-features-section){:target="_blank"}

- [[Rust] フィーチャーフラグの使い方 #Rust - Qiita](https://qiita.com/osanshouo/items/43271813b5d62e89d598){:target="_blank"}

一番上の Stack Overflow によれば、他にも ``[dev-dependencies]`` を用いる方法や、ライブラリプロジェクトの下にバイナリプロジェクトを作成し、ライブラリをクレートとして依存関係に含める方法もあるようです。
