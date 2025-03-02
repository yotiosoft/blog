---
layout: post
title: "RustでWindowsのプロセス情報を取得する「winprocinfo」を公開しました"
thumbnail: "/assets/img/thumbnails/feature-img/250302.png"
tags: [Rust, Windows, Windows API, 開発日記, winfuser, winprocinfo, お知らせ]
excerpt_separator: <!--more-->
---

先日、crates.io にて、Windows からプロセスやスレッドの情報を取得するライブラリクレート「winprocinfo」を公開しました。

[winprocinfo - crates.io: Rust Package Registry](https://crates.io/crates/winprocinfo){:target="_blank"}

せっかく作ったのでその中身のお話をしたいと思います。

<!--more-->	

# 使用方法

## 動作環境

- Windows x86_64
  - Rust（rustc, cargo）がインストールされていること
  - Windows 10, 11 にて動作確認済み

※ Windows 専用なので Linux、macOS は非対応です

## バイナリクレート（サンプルプログラム）をインストールする場合

```bash
cargo install winprocinfo
```

でインストールしたあと、

```bash
winprocinfo
```

で実行できます。

実行すると、現在 Windows 上で実行されているプロセスの名前や PID、ページ数や priority、そしてそのプロセスが持つスレッドの情報が一覧形式で出力されます。

![スクリーンショット 2025-03-01 112024](../../../assets/img/post/2025-03-02-winprocinfo/スクリーンショット 2025-03-01 112024.webp)

## ライブラリクレートとして利用する場合

追加したいクレートのディレクトリに移動したうえで

```bash
cargo add winprocinfo
```

とするか、あるいは ``Cargo.toml`` に

```toml
[dependencies]
winprocinfo = "0.1.1"
```

を追記します。

ドキュメントはこちらをご覧ください。

[https://docs.rs/winprocinfo/latest/winprocinfo/](https://docs.rs/winprocinfo/){:target="_blank"}

# 開発背景

もともとは、Windows 上でファイルを掴んでいるプロセスの情報を取得するツールを作ろうとしていたのが発端です。

- [Win32APIのRestart Managerでファイルをロックしているプロセスを特定する \| 為せばnull](https://blog.yotio.jp/2021/12/13/Win32API%E3%81%AERestart-Manager%E3%81%A7%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB%E3%82%92%E3%83%AD%E3%83%83%E3%82%AF%E3%81%97%E3%81%A6%E3%81%84%E3%82%8B%E3%83%97%E3%83%AD%E3%82%BB%E3%82%B9%E3%82%92%E7%89%B9%E5%AE%9A%E3%81%99%E3%82%8B.html)（投稿：2021年12月13日）

こちらの記事では Restart Manager という API によってファイルをロックしているプロセスを取得するプログラムを（C++ で）実現していました。

しかし、実際のところはファイルをロックせずに開いている場合があり、フォルダの移動や名称変更の際には、ファイルを開いているすべてのプロセスを閉じなければなりません。



そこで、ファイルを開いているすべてのプロセスを特定するツール（仮称：winfuser）の実現方法について検討していました。

Windows では、Linux でいうところのファイルディスクリプタはファイルハンドルと呼ばれ、プロセスやウィンドウ等とともに「ハンドル」という抽象化された概念によってカーネル内で管理されています。

winfuser は当初、下記のような構成で実現しようと考えていました。

1. まずはシステムからハンドル一覧を問い合わせる（``NtQuerySystemInformation``）
2. ハンドル一覧からファイルハンドルのみを取得する
3. ファイルハンドル群を走査し、検索対象のファイルと一致するものを検索する
4. 目的のファイルハンドルが見つかったら、そのハンドルの持ち主の PID を取得する
5. **PID を基に、プロセスハンドラの一覧からプロセス名を取得する**



このうち、 5. の部分のみを先に作ったのが winprocinfo です。以前、こちらの記事で作ったものをライブラリ化した形になります。

- [Rust+winapiで現在動作中のプロセス一覧を取得する \| 為せばnull](https://blog.yotio.jp/2023/06/25/rust_winapi_get_proc_info.html)（投稿：2023年6月25日）

実際のところは、プロセス名は PID からプロセスハンドルを取得（``OpenProcess``）→プロセスハンドルから実行ファイルのファイルパスを取得（``GetModuleBaseNameA``）という手順で実行したほうがより高速に済みます。よって winfuser では 5. の部分は結局いらなくなってしまったのですが、それはまた別の話です。（winfuser については後日別記事で書きます）

# 苦労した点など

Rust はコンパイラによる型安全性やメモリ安全性の保証が前提ですが、Windows API ともなるとそうはいきません。

Windows API はもともと C 言語や C# など、レガシーでメモリ安全を保証しない言語で利用されてきましたから、メモリ安全性の保証はプログラマが行うことが前提の設計となっており、unsafe なメソッドが多数存在します。

winprocinfo では、ntapi を FFI binding した [ntapi](https://crates.io/crates/ntapi){:target="_blank"} クレートを利用しているのですが、例えばプロセスの情報を格納する ``SYSTEM_PROCESS_INFORMATION`` は構造体はこのような定義になっています。

```rust
#[repr(C)]
pub struct SYSTEM_PROCESS_INFORMATION {

    pub NextEntryOffset: ULONG,
    pub NumberOfThreads: ULONG,
    pub WorkingSetPrivateSize: LARGE_INTEGER,
    pub HardFaultCount: ULONG,
    pub NumberOfThreadsHighWatermark: ULONG,
    ...
    pub ReadTransferCount: LARGE_INTEGER,
    pub WriteTransferCount: LARGE_INTEGER,
    pub OtherTransferCount: LARGE_INTEGER,
    pub Threads: [SYSTEM_THREAD_INFORMATION; 1],
}
```

``pub Threads: [SYSTEM_THREAD_INFORMATION; 1]`` のところにご注目。``Threads`` はプロセスが持つスレッド情報を示す ``SYSTEM_THREAD_INFORMATION`` 構造体の配列で、スレッド数分が確保される可変長配列なのですが、定義上は長さ1の配列になっていますね。

これを Rust で利用すると、``Threads[2]`` 以降にアクセスするコードはそもそもコンパイルが通りません。メモリ安全性が保証できないからですね。

そもそも ``Threads`` が可変長配列ということは、``SYSTEM_PROCESS_INFORMATION`` のサイズも可変長です。1つ目のメンバ変数に ``NextEntryOffset`` というものがありますが、これは ``NtQuerySystemInformation`` で全プロセスの ``SYSTEM_PROCESS_INFORMATION`` の配列を取得したときに、次のプロセスの ``SYSTEM_PROCESS_INFORMATION`` までのオフセット値を示します。つまり、この値は、自身の ``SYSTEM_PROCESS_INFORMATION`` 構造体のサイズを示しているわけです。

よって、まずは ``SYSTEM_PROCESS_INFORMATION`` から ``NextEntryOffset`` を取得し、それを構造体のサイズとします。そして ``SYSTEM_PROCESS_INFORMATION`` 構造体の先頭のアドレス（``next_address``）から ``NextEntryOffset`` までの分をバッファとして読み込んでおくわけです。

```rust
fn read_proc_info(next_address: *mut c_void) -> BufferStruct {
    // NextEntryOffset から構造体のサイズを取得
    let next_entry_offset = unsafe { (next_address as *const SYSTEM_PROCESS_INFORMATION).read().NextEntryOffset };
    
    // バッファの先頭アドレスから NextEntryOffset 分を構造体1つ分として設定
    let mut system_process_info_buffer = BufferStruct::with(next_address, next_entry_offset as usize);
    if next_entry_offset == 0 {
        return system_process_info_buffer;
    }

    system_process_info_buffer.base_address = next_address;
    system_process_info_buffer
}
```

スレッド情報を取得する際には、まずはスレッド配列の先頭アドレスを算出しておきます。``SYSTEM_PROCESS_INFORMATION`` 構造体は定義上、1つ分の``SYSTEM_THREAD_INFORMATION`` 構造体を保持していることになっていますから、構造体の先頭アドレスに対して、「``SYSTEM_PROCESS_INFORMATION`` 構造体1つ分の固定長のサイズ - ``SYSTEM_THREAD_INFORMATION`` 構造体1つ分の固定長のサイズ」を足したものがこれに当たります。

``SYSTEM_THREAD_INFORMATION`` 構造体は固定長ですから、あとは ``std::slice::from_raw_parts`` を利用して、スレッド数分の配列として読み込んでおけば OK です。

```rust
/// Retrieves a vector of ThreadInfo from the given process information buffer.
fn get_thread_info_vec(proc_info_buffer: &BufferStruct, number_of_threads: u32) -> Vec<ThreadInfo> {
    let thread_array_base = proc_info_buffer.base_address as usize + std::mem::size_of::<SYSTEM_PROCESS_INFORMATION>() - std::mem::size_of::<SYSTEM_THREAD_INFORMATION>();
    unsafe { 
        std::slice::from_raw_parts(thread_array_base as *const SYSTEM_THREAD_INFORMATION, number_of_threads as usize)
            .iter()
            .map(|x| ThreadInfo::from(x)).collect() 
    }
}
```

というコードを書くのに苦労しました…。Rust はメモリ安全な言語とはいえ、結局低レイヤなコードを書くうえでは unsafe コードからは逃れられません。

# おわりに

かなりニッチな内容になってしまいましたし、多分 winprocinfo もあまり需要がないと思いますが、Rust で unsafe なコードを書くという経験が得られたので個人的には満足です。せっかく作りましたから、タスクマネージャー的な GUI を実装したものを今後作ってみたいですね。
