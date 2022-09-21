---
layout: post
title: "warning: the following packages contain code that will be rejected by a future version of Rust"
tags: [Rust, Windows]
excerpt_separator: <!--more-->
---

最近、Rustを個人的に勉強し始めていて、簡単なツールを作っているのですが、Windowsでビルドしたときだけこんな警告が現れるようになりました。  

```powershell
> cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.13s
warning: the following packages contain code that will be rejected by a future version of Rust: winapi v0.2.8
note: to see what the problems were, use the option `--future-incompat-report`, or run `cargo report future-incompatibilities --id 59`
```

「次のパッケージは、将来rejectされるコードを含んでいます」という内容の警告。``winapi v0.2.8``がこの警告の原因のようで、これはWindows APIを呼び出すためのパッケージです。

<!--more-->  

# レポートを見てみる

詳しくは``cargo report future-incompatibilities --id 59``を実行しろと書いてあるので、それを実行してみる。

```powershell
> cargo report future-incompatibilities --id 59
The following warnings were discovered during the build. These warnings are an
indication that the packages contain code that will become an error in a
future release of Rust. These warnings typically cover changes to close
soundness problems, unintended or undocumented behavior, or critical problems
that cannot be fixed in a backwards-compatible fashion, and are not expected
to be in wide use.

Each warning should contain a link for more information on what the warning
means and how to resolve it.


To solve this problem, you can try the following approaches:


- Some affected dependencies have newer versions available.
You may want to consider updating them to a newer version to see if the issue has been fixed.

winapi v0.2.8 has the following newer versions available: 0.3.0, 0.3.1, 0.3.2, 0.3.3, 0.3.4, 0.3.5, 0.3.6, 0.3.7, 0.3.8, 0.3.9


- If the issue is not solved by updating the dependencies, a fix has to be
implemented by those dependencies. You can help with that by notifying the
maintainers of this problem (e.g. by creating a bug report) or by proposing a
fix to the maintainers (e.g. by creating a pull request):

  - winapi@0.2.8
  - Repository: https://github.com/retep998/winapi-rs
  - Detailed warning command: `cargo report future-incompatibilities --id 59 --package winapi@0.2.8`

- If waiting for an upstream fix is not an option, you can use the `[patch]`
section in `Cargo.toml` to use your own version of the dependency. For more
information, see:
https://doc.rust-lang.org/cargo/reference/overriding-dependencies.html#the-patch-section

The package `winapi v0.2.8` currently triggers the following future incompatibility lints:
> warning: `#[derive]` can't be used on a `#[repr(packed)]` struct that does not derive Copy (error E0133)
>    --> C:\Users\ytani\.cargo\registry\src\github.com-1ecc6299db9ec823\winapi-0.2.8\src\macros.rs:263:29
>     |
> 263 |           #[repr(C)] #[derive(Debug)] $(#[$attrs])*
>     |                               ^^^^^
>     |
>    ::: C:\Users\ytani\.cargo\registry\src\github.com-1ecc6299db9ec823\winapi-0.2.8\src\mmreg.rs:290:1
>     |
> 290 | / STRUCT!{#[repr(packed)] struct WAVEFORMATEX {
> 291 | |     wFormatTag: ::WORD,
> 292 | |     nChannels: ::WORD,
> 293 | |     nSamplesPerSec: ::DWORD,
> ...   |
> 297 | |     cbSize: ::WORD,
> 298 | | }}
>     | |__- in this macro invocation
>     |
>     = note: `#[allow(unaligned_references)]` on by default
>     = warning: this was previously accepted by the compiler but is being phased out; it will become a hard error in a future release!
>     = note: for more information, see issue #82523 <https://github.com/rust-lang/rust/issues/82523>
>     = note: this warning originates in the derive macro `Debug` (in Nightly builds, run with -Z macro-backtrace for more info)
(後略)
```

…まあいろいろ出てくるわけだけど、なんとなくwinapiのコード``#[derive(Debug)]``周りで警告が出てくることはわかる。  

もっと新しいバージョンがあるよと示されているので、そっちを使えばおそらく解決するのだと思われます。  

# 原因を探る

今回の場合はwinapiは自分でCargo.tomlのdependenciesに追加したわけではないので、dependenciesに追加した何らかのパッケージがこれに依存しているものと考えられます。今、Cargo.tomlのdependenciesがこんな感じ。

```toml
[dependencies]
curl = "0.4.44"
serde = { version = "^1.0.144", features = ["derive"] }
serde_json = "1.0.85"
regex = "0.0.1"
```

``cargo update``を使うなり``cargo search``で検索するなりして各パッケージを最新バージョンを見てみると、regexがやたら古いバージョンを指定していたことが判明。多分、自分が以前参考にしたネットのサンプルなり本なりに載っていたregaxのバージョン番号が相当古かったのでしょう。自分でちゃんと最新バージョンを調べないとだめですね。  

# 解決

というわけでregexを最新版の1.6.0に変更。  

```toml
[dependencies]
curl = "0.4.44"
serde = { version = "^1.0.144", features = ["derive"] }
serde_json = "1.0.85"
regex = "1.6.0"
```

再びビルドすると警告は消えました。自作プログラムの方は特に変更する必要はありませんでした。
