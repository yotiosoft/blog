---
layout: post
title: "Rust+winapiで現在動作中のプロセス一覧を取得する"
tags: [Windows, Rust, Windows API, winfuser, winprocinfo]
excerpt_separator: <!--more-->
---

Rust で WindowsAPI を用いて、現在 Windows 上で実行されているプロセスの一覧を取得するプログラムを作ったので、その手法について書いていこうと思います。プロセス ID とプロセス名を取得し、それをコンソール上に表示するという内容です。

<!--more-->

# 背景

1年半前、C++ で WindowsAPI の RestartManager API を使ってファイルをロックしているプロセスを取得する記事を書きました。

[Win32APIのRestart Managerでファイルをロックしているプロセスを特定する \| 為せばnull](https://blog.yotiosoft.com/2021/12/13/Win32API%E3%81%AERestart-Manager%E3%81%A7%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB%E3%82%92%E3%83%AD%E3%83%83%E3%82%AF%E3%81%97%E3%81%A6%E3%81%84%E3%82%8B%E3%83%97%E3%83%AD%E3%82%BB%E3%82%B9%E3%82%92%E7%89%B9%E5%AE%9A%E3%81%99%E3%82%8B.html)

こちらの記事では文字通り「ファイルをロックしているプロセスを取得する」ことはできたものの、ファイルをロックせずにファイルを開いているプロセスについての取得までは辿り着けませんでした。理由は「RestartManeger がファイルをロックしているプロセスしか取得できない仕様」だからです。

ファイルをロックせずに開いているプロセスの一覧が取得できると何が嬉しいかというと、Windows 標準のエクスプローラでは表示されない「どのプロセスが開いているか」という情報を取得できるようになります。

例えばフォルダの名前を変更したいとき、当該フォルダ内の何らかのファイルを開いているプロセスがあると、「別のプログラムがこのフォルダーまたはファイルを開いているので、操作を完了できません」というエラーが表示されます。この「別のプログラム」が何なのかを取得したいのです。

一応、先行研究はありまして、C# や C++ で実現している[先人方](http://hudson.doorblog.jp/archives/41656640.html){:target="_blank"}がいらっしゃいます。でも、どうせなら WinAPI の勉強がてら自分で実装したい、あわよくば GUI も付けて使いやすくしたい、というのが自分なりの欲望です。こういった操作には Microsoft によるドキュメント化がなされていない API 関数を用いる必要があり、慎重に調査する必要があります。

それで、今回は何をするかというと、とりあえず前段階として、WinAPI で「現在 Windows 上で動いているプロセスの一覧を取得する」というところまでを実現したいと思います。ここで取得したプロセス一覧取得は、後々ファイルを開いているプロセスの一覧取得プログラムに組み込んでいきます。

# 使用言語など

今回は Rust で書いています。まあぶっちゃけ、大部分を unsafe コードが占めることになるのですが、Rust 向けの WinAPI の crate が存在するのと、Rust の方が依存関係の管理・導入が楽なので Rust で書きました。

- OS : Windows 10 Home (64bit)

- Rust: version 1.67.0

- winapi crate: version 0.3.9 （[ドキュメント](https://docs.rs/winapi/latest/winapi/index.html){:target="_blank"}）

- ntapi crate: version 0.4.1（[ドキュメント](https://docs.rs/ntapi/latest/ntapi/){:target="_blank"}）

# 実装

- Cargo.toml

```toml
[package]
name = "WinProcInfo"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html
[dependencies]
ntapi = "0.4.1"
winapi = { version = "0.3.9", features = ["memoryapi", "processthreadsapi", "errhandlingapi"] }
```

- main.rs

```rust
use winapi::ctypes::*;
use winapi::um::memoryapi::*;
use winapi::um::processthreadsapi::*;
use winapi::um::winnt::{ MEM_COMMIT, MEM_RELEASE, PAGE_EXECUTE_READWRITE };
use winapi::shared::ntdef::*;
use ntapi::ntexapi::*;

// 現在動作中のすべてのプロセス情報を取得
// SystemProcessInformation を buffer に取得
fn get_system_processes_info(mut buffer_size: u32) -> Option<*mut c_void> {
    let base_address = unsafe {
        VirtualAlloc(std::ptr::null_mut(), buffer_size as usize, MEM_COMMIT, PAGE_EXECUTE_READWRITE)
    };

    // プロセス情報を取得
    // SystemProcessInformation : 各プロセスの情報（オプション定数）
    // base_address             : 格納先
    // buffer_size              : 格納先のサイズ
    // &mut buffer_size         : 実際に取得したサイズ
    let res = unsafe {
        NtQuerySystemInformation(SystemProcessInformation, base_address, buffer_size, &mut buffer_size)
    };

    // 取得失敗 → 解放
    if NT_ERROR(res) {
        unsafe {
            VirtualFree(base_address, 0, MEM_RELEASE);
            return None;
        }
    }

    Some(base_address)
}

// プロセス一つ分の情報を取得
fn get_proc_info(next_address: isize) -> SYSTEM_PROCESS_INFORMATION {
    unsafe {
        let mut system_process_info: SYSTEM_PROCESS_INFORMATION = std::mem::zeroed();

        // base_address の該当オフセット値から SYSTEM_PROCESS_INFORMATION 構造体の情報をプロセス1つ分取得
        ReadProcessMemory(
            GetCurrentProcess(), next_address as *const c_void, &mut system_process_info as *mut _ as *mut c_void, 
            std::mem::size_of::<SYSTEM_PROCESS_INFORMATION>() as usize, std::ptr::null_mut()
        );
        system_process_info
    }
}

// プロセス名を取得し、String 型で返す
fn get_proc_name(proc_info: SYSTEM_PROCESS_INFORMATION) -> String {
    // プロセス名を取得
    let mut image_name_vec: Vec<u16> = vec![0; proc_info.ImageName.Length as usize];
    unsafe {
        ReadProcessMemory(
            GetCurrentProcess(), proc_info.ImageName.Buffer as *const c_void, image_name_vec.as_mut_ptr() as *mut c_void, 
            proc_info.ImageName.Length as usize, std::ptr::null_mut()
        );
    }
    // \0 を除去
    let proc_name = String::from_utf16_lossy(&image_name_vec).trim_matches(char::from(0)).to_string();

    proc_name
}

// プロセス ID を取得
fn get_proc_id(proc_info: SYSTEM_PROCESS_INFORMATION) -> u32 {
    proc_info.UniqueProcessId as u32
}

fn main() {
    // メモリサイズ
    let buffer_size = 0x500000;

    // プロセス情報を取得
    let base_address_o = get_system_processes_info(buffer_size);
    if base_address_o.is_none() {
        println!("Failed to get system process information.");
        return;
    }
    let base_address = base_address_o.unwrap();

    // base_address に取得したプロセス情報を SYSTEM_PROCESS_INFORMATION 構造体 system_process_info に格納
    let mut system_process_info = get_proc_info(base_address as isize);

    let mut next_address = base_address as isize;
    // すべてのプロセス情報を取得
    loop {
        // 次のプロセス情報の格納先アドレス
        next_address += system_process_info.NextEntryOffset as isize;

        // base_address に取得したプロセス情報を SYSTEM_PROCESS_INFORMATION 構造体 system_process_info に格納
        system_process_info = get_proc_info(next_address);

        // プロセス名を取得
        let proc_name = get_proc_name(system_process_info);

        // プロセスIDを取得
        let proc_id = get_proc_id(system_process_info);

        // プロセス名とプロセスIDを表示
        println!("pid {} - {}", proc_id, proc_name);

        // すべてのプロセス情報を取得したら終了
        if system_process_info.NextEntryOffset == 0 {
            break;
        }
    }   
    unsafe {
        // メモリ解放
        VirtualFree(base_address, buffer_size as usize, MEM_RELEASE);
    }
}
```

# 実装の詳細

## プロセス情報の取得：get_system_procs_info(), get_proc_info()

大まかな手順は以下のとおりです。

1. ``NtQuerySystemInformation()`` で ``SystemProcessInformation`` を取得する

2. 1 で取得した全プロセス情報より、各プロセスの情報（プロセス ID とプロセス名）を取得する

3. 取得内容を表示

ここではポインタの操作や型破りな（つまり、型の構造体にメモリから値をぶち込む）操作が必要ですので、すべて unsafe モードでの動作となります。よって、メモリの扱いに注意が必要です。

`SystemProcessInformation` は「システムのプロセス情報を格納せよ」と ``NtQuerySystemInformation()`` に伝える定数であり、`NtQuerySystemInformation()` はこれを基にバッファ用アドレス ``base_address`` に対して全プロセスの情報を取得します。

その後、``ReadProcessMemory()`` で、取得したプロセス情報の内容を SYSTEM_PROCESS_INFORMATION 構造体型の変数 ``system_process_info`` に格納していきます。SYSTEM_PROCESS_INFORMATION 構造体はプロセスの情報を格納するための各メンバ変数を保持しており、これによってデータ単位で取り出し可能です。以下のメンバ変数を含みます。

```c
typedef struct _SYSTEM_PROCESS_INFORMATION {
    ULONG NextEntryOffset;
    ULONG NumberOfThreads;
    BYTE Reserved1[48];
    UNICODE_STRING ImageName;
    KPRIORITY BasePriority;
    HANDLE UniqueProcessId;
    PVOID Reserved2;
    ULONG HandleCount;
    ULONG SessionId;
    PVOID Reserved3;
    SIZE_T PeakVirtualSize;
    SIZE_T VirtualSize;
    ULONG Reserved4;
    SIZE_T PeakWorkingSetSize;
    SIZE_T WorkingSetSize;
    PVOID Reserved5;
    SIZE_T QuotaPagedPoolUsage;
    PVOID Reserved6;
    SIZE_T QuotaNonPagedPoolUsage;
    SIZE_T PagefileUsage;
    SIZE_T PeakPagefileUsage;
    SIZE_T PrivatePageCount;
    LARGE_INTEGER Reserved7[6];
} SYSTEM_PROCESS_INFORMATION;
```

（出典：[NtQuerySystemInformation 関数 (winternl.h) - Win32 apps \| Microsoft Learn](https://learn.microsoft.com/ja-jp/windows/win32/api/winternl/nf-winternl-ntquerysysteminformation){:target="_blank"}）

`base_adress` は、現在動作中の全プロセス分のデータを持つ可変長なデータですが、各プロセスのオフセット値は SYSTEM_PROCESS_INFORMATION 構造体のメンバ変数 ``NextEntryOffset`` から取得できます。よって、オフセット分を足した次のアドレスを算出していき、それぞれのデータについて loop でぶん回して見ていきます。

## プロセス名の取得：get_proc_name()

ここではプロセス名を取得するために、以下の手順で処理を進めます。

1. ``ImageName.Buffer`` から ``image_name_vec`` にバッファ内容をコピー

2. ``\0`` を除去

ここでは、SYSTEM_PROCESS_INFORMATION 構造体 の ``ImageName.Buffer`` から、[u16] 型の文字列 `image_name_vec` にプロセス名を格納していきます。このとき、ここで格納した文字列にはヌル文字 ``\0`` が含まれますので、これを除去しておきます。

## プロセス ID の取得：get_proc_id()

プロセス ID は SYSTEM_PROCESS_INFORMATION 構造体 の ``UniqueProcessId`` から取得可能です。

# 実行

![SnapCrab_Windows PowerShell_2023-6-25_19-1-19_No-00.png](\..\..\..\assets\img\post\2023-06-25\SnapCrab_Windows%20PowerShell_2023-6-25_19-1-19_No-00.png)

![SnapCrab_Windows PowerShell_2023-6-25_19-1-2_No-00.png](\..\..\..\assets\img\post\2023-06-25\SnapCrab_Windows%20PowerShell_2023-6-25_19-1-2_No-00.png)

自分自身のプロセス（win_proc_info.exe）も含め、すべてのプロセスを取得できました。同じ名前のプロセスがいくつもあるのが気になりますが、おそらく同一プロセス内のスレッドかと思われます。

# おわりに

今回は、Rust で winapi と ntapi の関数を用いてプロセス情報を取得するところまでできました。今回表示したのはプロセス ID とプロセス名のみですが、プロセス内のスレッド数や仮想メモリのサイズ、メモリページ数などが取得できるようです。詳しくは後述の参考文献 1 をご覧ください。

プロセス一覧の取得までできたので、次のステップとして、それぞれのプロセスが調査対象のファイルを開いているかどうか、それぞれが持つハンドラを取得することで判定するプログラムを今後は作っていきたいと思います。

# 参考文献

1. [NtQuerySystemInformation 関数 (winternl.h) - Win32 apps \| Microsoft Learn](https://learn.microsoft.com/ja-jp/windows/win32/api/winternl/nf-winternl-ntquerysysteminformation){:target="_blank"}

2. [winapi - Rust](https://docs.rs/winapi/latest/winapi/index.html){:target="_blank"}

3. [ntapi - Rust](https://docs.rs/ntapi/latest/ntapi/){:target="_blank"}

# リポジトリ

[GitHub - yotiosoft/WinProcInfoByRust](https://github.com/yotiosoft/WinProcInfoByRust){:target="_blank"}
