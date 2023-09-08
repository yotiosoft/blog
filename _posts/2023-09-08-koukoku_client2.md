---
layout: post
title: "TELNETクライアントにチャット書き込み機能を実装した"
tags: [Rust, 開発日記, Rustel]
excerpt_separator: <!--more-->
---

[前回](../06/koukoku_client.html)の続きです。例の電子公告にチャットが実装されましたので、チャット書き込み機能を実装しました。

<!--more-->

# 実験環境など

- OS : Windows 10, macOS 13.3.1

- Rust 1.72.0

- encoding_rs 0.8.23

- tokio 1.32.0

- clap 4.4.2

# 今回のターゲット

「一般社団法人サイバー技術・インターネット自由研究会

　インターネット公告システム ホームページ」

[telnet://koukoku.shadan.open.ad.jp](telnet://koukoku.shadan.open.ad.jp)

# リポジトリ

今回実装したものは「Rustel」と名付け、GitHub リポジトリにて公開しています。

[GitHub - yotiosoft/rustel at rustel_v2](https://github.com/yotiosoft/rustel/tree/rustel_v2){:target="_blank"}

## 利用方法

主な使用方法は以下のとおりです。

```bash
$ rustel -u [URL] -p [Port Number] -e [Encode (utf8 or sjis)] -i [IP version (4 or 6)]
```

例の電子公告にアクセスする場合はこんな感じになります。

```bash
$ rustel -u koukoku.shadan.open.ad.jp -p 23 -e sjis
```

# 実装

実装の全体像についてはリポジトリをご覧いただくとして、今回実装した内容を以下に示します。

## チャット書き込みの仕組み

公告の受信が単純な平文での TCP パケットの読み取りなのと同じように、書き込みについても単純に TCP パケットを送るだけです。これはチャットだけでなく、``nobody``といったコマンド送信についても同様です。

Rust の ``std::net::TcpStream`` であれば、``write()`` 関数などを使用してメッセージをサーバに対して平文で送れば書き込みが完了します。

TELNET 版の場合は受信時と同様に、チャット書き込み内容についても Shift_JIS で送信する必要があります。手っ取り早いのは UTF-8 で標準入力から書き込み文字列を取得し、 ``encoding_rs::SHIFT_JIS.encode()`` で Shift_JIS のバイナリに変換する方法です。

```rust
async fn telnet_write_sjis(stream: &mut WriteHalf<TcpStream>, str: &str) -> Result<(), std::io::Error> {
    println!("send: {}", str);
    let buf_writer = stream;
    let (cow, _, _) = encoding_rs::SHIFT_JIS.encode(str);
    buf_writer.write(&cow).await?;
    buf_writer.flush().await?;
    Ok(())
}
```

## 非同期処理化

厄介なのが標準入力からの書き込み受け付けと、電子公告およびチャットの受信・標準出力への出力処理を並行して行わなければならない点です。

通常の ``std::net::TcpStream`` に対する入力では、文字列の入力が完了し Enter キーが押されるまで処理をブロッキングしてしまいます。これでは、受信処理と送信処理をループ内で順に実行させようとしたときに、何か文字列を標準入力に入力（あるいは空の状態で Enter キーを押下）するまで、次の TCP パケットの受信・表示処理が行えません。

そこで今回は tokio による非同期処理を実装し、入力処理・出力処理を別々のスレッドとして実行させています。

具体的には、``tokio::net::tcp::stream::TcpStream`` を ``tokio::io::split()`` で ``ReadHalf<TcpStream>``、``WriteHalf<TcpStream>`` に分割し、それぞれを受信と表示を行う ``telnet_read()`` 、入力受付と送信を行う ``telnet_write()`` に渡します。これはもともと一つであった TcpStream を入力・出力に分け、それぞれを別々のスレッドとして同時実行させるための処置です。これらの関数は ``tokio::spawn()`` で非同期タスクとして実行しています。

```rust
match TcpStream::connect(address).await {
    Ok(stream) => {
        println!("Connected to the server!");
        let (reader, writer) = tokio::io::split(stream);

        // read
        let reader = tokio::spawn(telnet_read(reader, encode.clone()));

        // write
        let writer = tokio::spawn(telnet_input(writer, encode.clone()));

        let _ = reader.await?;
        writer.abort();
    },
    Err(e) => println!("Failed to connect: {}", e),
}
```

これに伴い、受信を行う ``telnet_read()`` の方も修正を加えました。前回は ``main()`` 内で受信内容を表示していましたが、今回は受信用スレッド内で表示させるために`telnet_read()`  内で表示も行います。

```rust
async fn telnet_read_utf8(stream: &mut ReadHalf<TcpStream>) -> Result<Option<String>, std::io::Error> {
    let mut buf_reader = BufReader::new(stream);
    let mut buffer = String::new();
    buf_reader.read_to_string(&mut buffer).await?;
    //println!("Received message: {}", buffer);
    if buffer.len() == 0 {
        return Ok(None);
    }
    Ok(Some(buffer))
}

async fn telnet_read_sjis(stream: &mut ReadHalf<TcpStream>) -> Result<Option<String>, std::io::Error> {
    let mut buf_reader = BufReader::new(stream);
    let buffer = buf_reader.fill_buf().await?;
    if buffer[0] == 0 {
        return Ok(None);
    }
    let (cow, _, _) = encoding_rs::SHIFT_JIS.decode(&buffer);
    let text = cow.into_owned();
    //println!("Received message: {:?} {}", buffer, text);
    Ok(Some(text))
}

async fn telnet_read(mut stream: ReadHalf<TcpStream>, encode: Encode) -> Result<(), std::io::Error> {
    loop {
        let str = match encode {
            Encode::UTF8 => telnet_read_utf8(&mut stream).await?,
            Encode::SHIFTJIS => telnet_read_sjis(&mut stream).await?,
        };

        if let Some(str) = str {
            print!("{}", str);
            std::io::stdout().flush()?;
            if str == "\0" {
                break;
            }
            // is buffer contains EOF?
            if str.contains("\u{1a}") {
                break;
            }
        }
        else {
            break;
        }
    };
    Ok(())
}
```

# 動作確認

![chat.png](../../../assets/img/post/2023-09-08/chat.png)

緑色の書き込みが Rustel で書き込んだ内容です。きちんとチャットに書き込めていることが確認できました。

# 今後の予定

- Telnet 標準表現に対応する

- TELNET サーバも立てられるようにする（なるべく）

- TELNET on SSL に対応する（できたら）

# 参考文献

- [AsyncBufReadExt in tokio::io - Rust](https://docs.rs/tokio/latest/tokio/io/trait.AsyncBufReadExt.html){:target="_blank"}

- [spawn in tokio::task - Rust](https://docs.rs/tokio/latest/tokio/task/fn.spawn.html){:target="_blank"}

- [TcpStream in tokio::net - Rust](https://docs.rs/tokio/latest/tokio/net/struct.TcpStream.html){:target="_blank"}

- [I/O｜Tokio チュートリアル (日本語訳)](https://zenn.dev/magurotuna/books/tokio-tutorial-ja/viewer/io){:target="_blank"}
