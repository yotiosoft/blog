---
layout: post
title: "TELNETクライアント（読み取り専用）を作った"
tags: [Rust, 開発日記, Rustel]
excerpt_separator: <!--more-->
---

TELNET で公開している電子公告が一部の界隈で話題になっていて、これだったら簡単に受信用クライアントを作れそうだなと思ったので作りました。

まだまだ作成途中ですが、電子公告が次から次へと新機能が実装されており、追いつけなくなってしまうためここで一旦記事にさせていただきます。

<!--more-->

# 実験環境など

- OS : Windows 10, macOS 13.3.1

- Rust 1.67.0

- encoding_rs 0.8.23

# 今回のターゲット

一般社団法人サイバー技術・インターネット自由研究会

インターネット公告システム ホームページ

[telnet://koukoku.shadan.open.ad.jp](telnet://koukoku.shadan.open.ad.jp)

## メッセージ送受信の仕組み

TELNET にもプロトコルが定義されていて、標準表現（standard representation）なるものが RFC 854 で定義されています。

[RFC 854 - Telnet Protocol Specification](https://datatracker.ietf.org/doc/html/rfc854){:target="_blank"}

しかしながら、「インターネット公告システム」では一方的に文字列をポート23で送りつけるだけの、ほぼソケット通信と変わらない仕様になっています（チャット機能はまだ実装してないので知らない）。

## おことわり

この記事の内容は昨日の時点での実装になります。

「インターネット公告システム」があまりにも目まぐるしく進化しており、本日新たにチャット機能が実装されたようですが、現状チャット書き込みには対応できておりません。表示のみ対応しております。

# 実装

「インターネット公告システム」以外の TELNET クライアントとしても使えるよう、エンコードは UTF-8 と Shift_JIS の2種に対応しています。また、IPv6 のサーバも接続可能です。

## 受信

一応 TELNET とされていますが、前述の通り今回の場合はほぼ標準表現は関係なく、単なる平文でのソケット通信と変わりありません。よって TELNET プロトコルの実装は後回しにし、とりあえずは簡易なソケット通信とほぼ同一の実装にしました。

ここは他のサーバとの互換性がないという点で矛盾していますが、まだ途中経過なのでお許しください。

## 文字コード

ちょっと厄介なのが文字コードで、Telnet 版では Shift_JIS になっています（本日公開された TELNETS 版は UTF-8 のようですが）。

基本的に 1 パケット 1 文字で送信しているようなので、バッファは 2 バイトあれば十分かと思います。しかしながら、他のサーバとの通信にも互換性をもたせるため、TCP パケットの最大長（65536 bytes）分のバッファを用意することとしました。

Rust では String 型が UTF-8 前提となっているため、String 型で read しようとするとエラーを返してしまいます。そこでバイナリとして read し、その後に encoding_rs crate を用いて Shift_JIS に変換することで文字コード問題は解決しました。

## コード全体像

```rust
use std::net::{ToSocketAddrs, TcpStream};
use std::io::{BufReader, Write, Read};
use encoding_rs;

const TCP_LEN_MAX: usize = 65536;

enum Encode {
    UTF8,
    SHIFT_JIS,
}

enum IPv {
    IPv4,
    IPv6,
}

// utf-8 の場合
fn telnet_read_utf8(stream: &TcpStream) -> Result<Option<String>, std::io::Error> {
    let mut buf_reader = BufReader::new(stream);
    let mut buffer = String::new();
    buf_reader.read_to_string(&mut buffer)?;
    //println!("Received message: {:?} {}", buffer, text);
    if buffer.len() == 0 {
        return Ok(None);
    }
    Ok(Some(buffer))
}

// sjis の場合
fn telnet_read_sjis(stream: &TcpStream) -> Result<Option<String>, std::io::Error> {
    let mut buf_reader = BufReader::new(stream);
    // 65536 bytes のバッファを確保
    let mut buffer: [u8; TCP_LEN_MAX] = [0; TCP_LEN_MAX];
    buf_reader.read(&mut buffer)?;
    if buffer[0] == 0 {
        return Ok(None);
    }
    // バイナリから文字コード変換
    let (cow, _, _) = encoding_rs::SHIFT_JIS.decode(&buffer);
    let text = cow.into_owned();
    //println!("Received message: {:?} {}", buffer, text);
    Ok(Some(text))
}

fn telnet_read(stream: &TcpStream, encode: &Encode) -> Result<Option<String>, std::io::Error> {
    match encode {
        Encode::UTF8 => telnet_read_utf8(stream),
        Encode::SHIFT_JIS => telnet_read_sjis(stream),
    }
}

fn main() {
    let host = "koukoku.shadan.open.ad.jp";
    let port = 23;
    let encode = Encode::SHIFT_JIS;
    let ipv = IPv::IPv4;

    let host_and_port = format!("{}:{}", host, port);
    let mut addresses = host_and_port.to_socket_addrs().unwrap();

    let address = match ipv {
        IPv::IPv4 => addresses.find(|x| x.is_ipv4()),
        IPv::IPv6 => addresses.find(|x| x.is_ipv6()),
    };
    if let Some(address) = address {
        println!("Found an IPv4 address: {}", address);

        match TcpStream::connect(address) {
            Ok(stream) => {
                println!("Connected to the server!");
                loop {
                    let str = telnet_read(&stream, &encode).unwrap();
                    if let Some(str) = str {
                        print!("{}", str);
                        std::io::stdout().flush().unwrap();
                        // null 文字なら break
                        if str == "\0" {
                            break;
                        }
                        // EOF を含むなら break
                        if str.contains("\u{1a}") {
                            break;
                        }
                    }
                    else {
                        break;
                    }
                }
            },
            Err(e) => println!("Failed to connect: {}", e),
        }
    } else {
        println!("No IPv4 address found");
    }
}
```

# 動作確認

今接続するとチャットの内容とともに投稿者のホスト名まで表示されてしまうため、チャット機能が実装される前の動画を載せておきます。

<div>
<video src="../../../assets/img/post/2023-09-06/telnet.mp4" controls></video>
</div>

# 今後の予定

- チャット書き込みに対応する

- Telnet 標準表現に対応する＆他のサーバにも対応する

- 非同期処理で受信するようにする（Tokio など）

- TELNET on SSL に対応する（できたら）

# 補足：現状の現実的なアクセス方法

公告付き

```bash
$ openssl s_client -connect koukoku.shadan.open.ad.jp:992
```

公告無し（チャットのみ）

```bash
$ openssl s_client -connect koukoku.shadan.open.ad.jp:992 | grep '^>>'
```
