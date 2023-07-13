---
layout: post
title: "自作画像変換ツール「rusimg」の紹介"
tags: [Rust, お知らせ, 開発日記]
excerpt_separator: <!--more-->
---

コマンドラインから画像を変換するツールは既にありますが、特定の画像フォーマット専用のコマンドであったり（例：``cwebp/dwebp``）と、いちいち形式に合わせてコマンド体系を覚えるのは面倒です。また、こういったコマンドは一枚ずつの変換にしか対応していなかったりして、複数枚を一度に変換したい場合は for 文ループをコマンドで書いたり、それを自分でシェルスクリプト等で定義しなければなりません。

そこで、webp のようなナウい形式から jpeg, png といった従来の形式まで、複数の形式に対応した画像変換 CLI ツールを作ろうと企み、3月頃から細々と開発しております。

だいぶ形になってきており、ベータ版公開直前くらいの段階まで来ているので、ここらへんで一旦開発日記でも付けておこうかなと思います。（**あ、まだリリースはしてないです。スミマセン**）

<!--more-->

# 使用言語など

例に漏れず Rust です。Rust バンザイ。

今回はきちんと選定理由があって、各画像フォーマットに対応したクレートが豊富であること、便利なコマンドライン引数処理クレート (clap) があること、依存関係の導入が楽であること、マルチプラットフォームへの対応が容易であること、高速であること、ライブラリ化が簡単であること、ドキュメントが自動生成できること、メモリ安全であることが理由として挙げられます。

画像変換クレートは以下のものを使用しています。

- image

- mozjpeg

- oxipng

- webp

# 名前

名前は「rusimg」です。名前から連想されるように、Rust で作った画像処理ツールなので rusimg。安直な名前なので、正式版の公開までにはもうちょっとカッコイイ名前に変えようかなと思っています。

# 対応画像形式

現時点では jpeg, png, webp, bmp の4つです。いずれも相互に変換が可能です。

画像形式ごとにモジュール化してあるので、対応形式の追加は容易です。今後は gif や RAW 形式にも対応していきたいなと思っています。

# システム構成

rusimg は大きく分けて2段階のモジュールから構成されます。

- rusimg.rs：各画像形式モジュールのインターフェースとなるモジュール
  
  - bmp.rs, jpeg.rs, png.rs, webp.rs：各画像形式特有のモジュール

他にも rusimg.rs の各関数を呼び出す main.rs、コマンドラインから引数を受け取る parse.rs があります。

現時点では CLI ツールのみでライブラリ化はしていませんが、ライブラリ化するとしたらほぼ rusimg.rs の各関数を呼び出すだけのラッパ関数になるかなと思います。

# 機能

- 画像変換機能 (-c)
  
  - jpeg, png, webp, bmp の間で相互に画像変換できるよ

- 画像圧縮機能 (-q)（bmp 除く）
  
  - 画像変換時に、画像の品質を 0～100 % の間で指定できるよ
  
  - パーセンテージが低いほど圧縮率が高くなるよ
  
  - ただし bmp は画像圧縮なんて関係ないので圧縮できないよ

- ソースファイル削除機能 (-d)
  
  - 画像変換時に、変換元の画像を自動で削除できるよ

- 一括変換機能
  
  - ディレクトリ内の画像を一括変換できるよ
  
  - ワイルドカードでのファイル名指定にも対応

- 出力先ファイル名・ディレクトリ指定機能 (-o)
  
  - 画像出力先を指定できるよ

- トリミング機能 (-t)
  
  - 画像の一部を切り出して出力できるよ

- リサイズ機能 (-r)
  
  - 画像を任意のサイズにリサイズできるよ（パーセンテージでの指定必須）

- グレースケール化機能 (-g)
  
  - 画像をモノクロにできるよ

- ヘルプ機能 (-h)
  
  - コマンドのヘルプを表示するよ

- プレビュー機能 (-v)
  
  - 変換後の画像をコマンドライン上でプレビューできるよ（意味不明）



自分なりにあったら良いなと思った機能を揃えています。

実は動作テストを兼ねて、このブログで使う画像を jpeg や png から webp に変換するときには rusimg を使っているのですが、特に一括変換と変換元ファイルの自動削除は自分でもよく使います。そもそも rusimg の開発を始めた大元の動機が、ブログを書いてるときの画像変換や画像圧縮の煩雑さにあります。

![SnapCrab_Windows PowerShell_2023-7-13_2-37-23_No-00.png](..\..\..\assets\img\post\2023-07-13\SnapCrab_Windows%20PowerShell_2023-7-13_2-37-23_No-00.png)

（もちろんこの画像↑も rusimg で変換しています）

基本、画像変換後は変換元の画像ファイルには用がありませんので、自分のユースケースでは普段から容量削減のためにさっさと消してしまいます。自分で言うのもなんですが、画像の一括変換から削除までコマンド一つで済むので結構便利です。

一番最近に追加したのはワイルドカードでのファイル指定機能です。すなわち、「IMG*.jpeg」と指定すれば、IMG0001.jpeg, IMG0002.jpeg, …といった、パターンに当てはまる複数ファイルを一度に指定できるというものですね。

最後のプレビュー機能というのが一番意味不明だと思うのですが、これはほとんどおまけみたいなもので、わざわざ画像ビューアを立ち上げなくてもコマンドラインで画像をプレビューできるというものです。

つまりこういうことです。

![SnapCrab_Windows PowerShell_2023-7-13_2-14-30_No-00.png](..\..\..\assets\img\post\2023-07-13\SnapCrab_Windows%20PowerShell_2023-7-13_2-14-30_No-00.png)

こんな画質の粗い無駄機能いつ使うんだ、という感じですが、実はトリミング機能やグレースケール化機能を使ったとき、操作結果の確認などに使えたりします。

![SnapCrab_Windows PowerShell_2023-7-13_2-22-31_No-00.png](..\..\..\assets\img\post\2023-07-13\SnapCrab_Windows%20PowerShell_2023-7-13_2-22-31_No-00.png)

これは viuer というクレートを利用しており、短いコードで簡単にコマンドラインに画像出力できるという代物になっております。

[viuer - Rust](https://docs.rs/viuer/latest/viuer/)

```rust
fn view(&self) -> Result<(), RusimgError> {
    let conf_width = self.width as f64 / std::cmp::max(self.width, self.height) as f64 * 100 as f64;
    let conf_height = self.height as f64 / std::cmp::max(self.width, self.height) as f64 as f64 * 50 as f64;
    let conf = viuer::Config {
        absolute_offset: false,
        width: Some(conf_width as u32),
        height: Some(conf_height as u32),    
        ..Default::default()
    };

    viuer::print(&self.image, &conf).map_err(|e| RusimgError::FailedToViewImage(e.to_string()))?;

    Ok(())
}
```

# もうちょい中身の話

## 画像構造体

先程、rusimg.rs の下に各画像形式のモジュールがあるという旨の話をしましたが、これは画像形式ごとに使用クレートが異なるためです。

Rust には image というクレートがあり、これは ``ImageBuffer`` という型を持ちます。``ImageBuffer`` は画像の RGB 値をそのままメモリ上に保持する行列になっていて、各ピクセルの値に直接アクセスできるものになっています。

image はまた、``DynamicImage`` という列挙体を持ちます。これを経て ``ImageBuffer`` を 各種フォーマット（RGB8、RGBA8、RGB16、RGBA16、…）に変換することが可能です。``DynamicImage`` は ``ImageBuffer`` 自体を保持しておりますので、画像バッファを保持するための構造体としても利用できます。実際、``image::load_from_memory()`` はファイルのバイナリデータから ``DynamicImage`` に変換する機能を持ちます。

実は image クレートは結構万能で、jpeg, png, bmp の画像入力 / 画像出力に対応しています。ですので、画像ファイルを ``DynamicImage`` に変換しておけば jpeg、png、bmp の各形式に変換・出力できるわけです。

```rust
fn open(path: PathBuf) -> Result<Self, RusimgError> {
    let mut raw_data = std::fs::File::open(&path).map_err(|e| RusimgError::FailedToOpenFile(e.to_string()))?;
    let mut buf = Vec::new();
    raw_data.read_to_end(&mut buf).map_err(|e| RusimgError::FailedToReadFile(e.to_string()))?;
    let metadata_input = raw_data.metadata().map_err(|e| RusimgError::FailedToGetMetadata(e.to_string()))?;

    let image = image::load_from_memory(&buf).map_err(|e| RusimgError::FailedToOpenImage(e.to_string()))?;
    let (width, height) = (image.width() as usize, image.height() as usize);

    let extension_str = path.extension().and_then(|s| s.to_str()).unwrap_or("").to_string();

    Ok(Self {
        image,
        image_bytes: None,
        width,
        height,
        operations_count: 0,
        extension_str,
        metadata_input,
        metadata_output: None,
        filepath_input: path,
        filepath_output: None,
    })
}
```

では、mozjpeg、oxipng はどこで使っているのでしょうか？これらは画像の圧縮、すなわち品質調整に使っています。画像圧縮機能は image クレートには備わっていないので、圧縮時だけそれぞれの画像形式のクレートに合わせてデータを変換した上で処理しています。

```rust
fn compress(&mut self, quality: Option<f32>) -> Result<(), RusimgError> {
    let quality = quality.unwrap_or(75.0);  // default quality: 75.0

    let image_bytes = self.image.clone().into_bytes();

    let mut compress = Compress::new(ColorSpace::JCS_RGB);
    compress.set_scan_optimization_mode(ScanMode::AllComponentsTogether);
    compress.set_size(self.width, self.height);
    compress.set_mem_dest();
    compress.set_quality(quality);
    compress.start_compress();
    compress.write_scanlines(&image_bytes);
    compress.finish_compress();

    self.image_bytes = Some(compress.data_to_vec().map_err(|_| RusimgError::FailedToCompressImage(None))?);

    println!("Compress: Done.");
    self.operations_count += 1;

    Ok(())
}
```

ところで、一つだけ image クレートが対応していない画像形式があります。そう、webp です。webp だけは画像入力 / 画像出力 / 画像圧縮含め、すべて webp クレートで行う必要があります。

幸い、webp クレートは ``DynamicImage`` からデコード / エンコードする機能を持ちます。よって、これを用いて画像への操作は DynamicImage で行い、画像の読み込み・保存時だけ webp クレートでデコードとエンコードを行います。

```rust
fn open(path: PathBuf) -> Result<Self, RusimgError> {
    let mut raw_data = std::fs::File::open(&path).map_err(|e| RusimgError::FailedToOpenFile(e.to_string()))?;
    let mut buf = Vec::new();
    raw_data.read_to_end(&mut buf).map_err(|e| RusimgError::FailedToReadFile(e.to_string()))?;
    let metadata_input = raw_data.metadata().map_err(|e| RusimgError::FailedToGetMetadata(e.to_string()))?;

    let webp_decoder = webp::Decoder::new(&buf).decode();
    if let Some(webp_decoder) = webp_decoder {
        let image = webp_decoder.to_image();
        let (width, height) = (image.width() as usize, image.height() as usize);

        Ok(Self {
            image,
            image_bytes: Some(buf),
            width,
            height,
            operations_count: 0,
            required_quality: None,
            metadata_input,
            metadata_output: None,
            filepath_input: path,
            filepath_output: None,
        })
    }
    else {
        return Err(RusimgError::FailedToDecodeWebp);
    }
}
```

## コマンドライン引数の受け取り

コマンドライン引数の受け取りには clap という有名なパーサを利用しています。

[clap - Rust](https://docs.rs/clap/latest/clap/)

これはコマンドライン引数の受け取りはもちろん、オプションの生成やヘルプ、バージョン情報の表示も自動で実装してくれる優れものです。

例えばこんな感じで実装すれば、

```rust
pub struct ArgStruct {
    pub souce_path: Option<PathBuf>,
    pub destination_path: Option<PathBuf>,
    pub destination_extension: Option<String>,
    pub quality: Option<f32>,
    pub delete: bool,
    pub resize: Option<u8>,
    pub trim: Option<((u32, u32), (u32, u32))>,
    pub grayscale: bool,
    pub view: bool,
}

#[derive(clap::Parser, Debug)]
#[command(author, version, about, long_about = None)]
struct Args {
    /// Source file path (file name or directory path)
    source: Option<PathBuf>,

    /// Destination file path (file name or directory path)
    #[arg(short, long)]
    output: Option<PathBuf>,

    /// Destination file extension (e.g. jpeg, png, webp, bmp)
    #[arg(short, long)]
    convert: Option<String>,

    /// Resize images in parcent (must be 0 < resize <= 100)
    #[arg(short, long)]
    resize: Option<u8>,

    /// Trim image
    #[arg(short, long)]
    trim: Option<String>,

    /// Grayscale image
    #[arg(short, long)]
    grayscale: bool,

    /// Image quality (for compress, must be 0.0 <= q <= 100.0)
    #[arg(short, long)]
    quality: Option<f32>,

    /// Delete source file
    #[arg(short, long)]
    delete: bool,

    /// View result in the comand line
    #[arg(short, long)]
    view: bool,
}
```

``Option<>`` で囲った変数はオプション引数に該当しますので、これらがコマンドライン引数の受け取り時にオプション引数として扱われます。

また、``#[arg(short, long)]`` を指定することで、変数名に合わせたショート形式・ロング形式のオプション名も生成されます。例えば ``source: Option<PathBuf>`` であれば ``-s`` と ``--source``、``output: Option<PathBuf>`` であれば``-o``と``--output`` といった具合です。

前述の通り、clap は自動でヘルプ画面を生成してくれます。このヘルプ画面に表示する各オプションの説明文ですが、これは ``///`` から始まるドキュメントコメントが自動で反映されます。

```
$ rusimg -h
Usage: rusimg [OPTIONS] [SOURCE]

Arguments:
  [SOURCE]  Source file path (file name or directory path)

Options:
  -o, --output <OUTPUT>    Destination file path (file name or directory path)
  -c, --convert <CONVERT>  Destination file extension (e.g. jpeg, png, webp, bmp)
  -r, --resize <RESIZE>    Resize images in parcent (must be 0 < resize <= 100)
  -t, --trim <TRIM>        Trim image
  -g, --grayscale          Grayscale image
  -q, --quality <QUALITY>  Image quality (for compress, must be 0.0 <= q <= 100.0)
  -d, --delete             Delete source file
  -v, --view               View result in the comand line
  -h, --help               Print help
  -V, --version            Print version
```

## ワイルドカード文字への対応

ファイルパスにおけるワイルドカード文字（``?`` と ``*``）への対応には、glob というクレートを利用しています。

[glob - Rust](https://docs.rs/glob/latest/glob/)

下記のコードは、ワイルドカードを含むファイルパスを与えたときに、該当するすべての画像ファイルパスが格納された配列を返す関数です。

```rust
fn get_files_by_wildcard(source_path: &PathBuf) -> Result<Vec<PathBuf>, String> {
    let mut ret = Vec::new();
    for entry in glob(source_path.to_str().unwrap()).expect("Failed to read glob pattern") {
        match entry {
            Ok(path) => {
                // 画像形式であればファイルリストに追加
                if rusimg::get_extension(&path).is_ok() {
                    ret.push(path);
                }
            },
            Err(e) => println!("{:?}", e),
        }
    }
    Ok(ret)
}
```

ワイルドカードで指定した場合、複数のファイルパスが該当し得ます。よって glob はパターンマッチしたファイルパスエントリを配列で返しますので、その中から対応画像フォーマット形式のものを抽出し、順次配列 ``ret`` に追加して返します。

# 今後の予定

- gif、RAW 形式への対応
  
  - [gif - Rust](https://docs.rs/gif/latest/gif/)
  
  - [GitHub - pedrocr/rawloader: rust library to extract the raw data and some metadata from digital camera images](https://github.com/pedrocr/rawloader)

- Inpainting 機能
  
  - 消しゴムマジックで消してやるのさっ！
  
  - [GitHub - EmbarkStudios/texture-synthesis: 🎨 Example-based texture synthesis written in Rust 🦀](https://github.com/EmbarkStudios/texture-synthesis)

- Style Transfer 機能
  
  - 画像の質感を他の画像に適用する機能
  
  - [GitHub - EmbarkStudios/texture-synthesis: 🎨 Example-based texture synthesis written in Rust 🦀](https://github.com/EmbarkStudios/texture-synthesis)

- 顔検出・モザイク機能
  
  - OpenCV の利用を検討
  
  - yapps 向けに以前作ったものを流用？
    
    - [opencv.jsを使ってブラウザ上で顔を検出＆モザイクをかける](../../../2022/12/08/opencv.js%E3%81%A7%E9%A1%94%E8%AA%8D%E8%AD%98-%E3%83%A2%E3%82%B6%E3%82%A4%E3%82%AF%E3%82%92%E3%81%8B%E3%81%91%E3%82%8B.html)

- 二値化機能

- エッジ検出機能

- 更新日時でのファイル指定機能

- バグ修正

- とにかく完成させる

とりあえずは現状の機能でバグ修正を進め、一旦ベータ版として近日中に公開しようかなと思っております。

# 参考文献

- [image - Rust](https://creative-coding-the-hard-way.github.io/Agents/image/index.html)

- [GitHub - mozilla/mozjpeg: Improved JPEG encoder.](https://github.com/mozilla/mozjpeg)

- [oxipng - Rust](https://docs.rs/oxipng/latest/oxipng/)

- [webp - Rust](https://docs.rs/webp/latest/webp/)

- [viuer - Rust](https://docs.rs/viuer/latest/viuer/)

# リポジトリ

コードを公開しておりますので、Rust の開発環境があれば Windows / macOS / Linux でコンパイル可能です。ただしまだ動作の保証はできませんのでご承知おきください。

[GitHub - yotiosoft/rusimg: A image processing CLI tool for bmp, jpeg, png and webp written by Rust.](https://github.com/yotiosoft/rusimg)
