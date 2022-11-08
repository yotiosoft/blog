---
layout: post
title: "UbuntuにHomebrewをインストールしたらRustがビルドできなくなった"
tags: [Homebrew, Ubuntu, Rust]
excerpt_separator: <!--more-->
---

Mac 用のパッケージマネージャである Homebrew、実は Linux にも対応していて、この度 Ubuntu 20.04 に導入したのですが…  
導入完了後、なぜか自作の Rust 製のツールが rustc でコンパイルできない状況に。

<!--more-->  

# 概要

```bash
$ cargo run
   Compiling proc-macro2 v1.0.43
   Compiling autocfg v1.1.0
   Compiling unicode-ident v1.0.4
   Compiling quote v1.0.21
   　　　　　　︙
   Compiling itoa v1.0.4
   Compiling regex v1.6.0
   Compiling async-std v1.12.0
   Compiling toml v0.5.9
   Compiling confy v0.5.0
   Compiling dptran v0.1.0 (/home/ytani0323/git/dptran)
error: linking with `cc` failed: exit status: 1
  |
  = note: "cc" "-m64" "/tmp/rustc66PyJg/symbols.o"
  　　　　　　　　　　　　　　︙
```

コンパイル時に``cc``とのリンクに失敗したようで。少なくともプログラムの問題ではなさそうです。``cc``なので、Rust ではなく C++ コンパイラのエラーか。  
で、もう少し下を見てみると。  

```bash
  = note: /usr/bin/ld: /home/linuxbrew/.linuxbrew/opt/openssl@1.1/lib/libcrypto.so.1.1: undefined reference to `pthread_setspecific@GLIBC_2.34'
          /usr/bin/ld: /home/linuxbrew/.linuxbrew/opt/openssl@1.1/lib/libcrypto.so.1.1: undefined reference to `dlopen@GLIBC_2.34'
          /usr/bin/ld: /home/linuxbrew/.linuxbrew/opt/openssl@1.1/lib/libcrypto.so.1.1: undefined reference to `pthread_rwlock_init@GLIBC_2.34'
          /usr/bin/ld: /home/linuxbrew/.linuxbrew/opt/openssl@1.1/lib/libcrypto.so.1.1: undefined reference to `pthread_rwlock_wrlock@GLIBC_2.34'
          /usr/bin/ld: /home/linuxbrew/.linuxbrew/opt/openssl@1.1/lib/libcrypto.so.1.1: undefined reference to `dlerror@GLIBC_2.34'
          /usr/bin/ld: /home/linuxbrew/.linuxbrew/opt/openssl@1.1/lib/libcrypto.so.1.1: undefined reference to `pthread_getspecific@GLIBC_2.34'
          /usr/bin/ld: /home/linuxbrew/.linuxbrew/Cellar/curl/7.86.0/lib/libcurl.so: undefined reference to `pthread_create@GLIBC_2.34'
          /usr/bin/ld: /home/linuxbrew/.linuxbrew/Cellar/curl/7.86.0/lib/libcurl.so: undefined reference to `pthread_join@GLIBC_2.34'
          /usr/bin/ld: /home/linuxbrew/.linuxbrew/opt/openssl@1.1/lib/libcrypto.so.1.1: undefined reference to `dlclose@GLIBC_2.34'
          /usr/bin/ld: /home/linuxbrew/.linuxbrew/Cellar/curl/7.86.0/lib/libcurl.so: undefined reference to `pthread_detach@GLIBC_2.34'
          /usr/bin/ld: /home/linuxbrew/.linuxbrew/opt/openssl@1.1/lib/libcrypto.so.1.1: undefined reference to `pthread_rwlock_rdlock@GLIBC_2.34'
          /usr/bin/ld: /home/linuxbrew/.linuxbrew/opt/openssl@1.1/lib/libcrypto.so.1.1: undefined reference to `pthread_key_delete@GLIBC_2.34'
          /usr/bin/ld: /home/linuxbrew/.linuxbrew/Cellar/curl/7.86.0/lib/libcurl.so: undefined reference to `fstat@GLIBC_2.33'
          /usr/bin/ld: /home/linuxbrew/.linuxbrew/Cellar/curl/7.86.0/lib/libcurl.so: undefined reference to `stat@GLIBC_2.33'
          /usr/bin/ld: /home/linuxbrew/.linuxbrew/opt/openssl@1.1/lib/libcrypto.so.1.1: undefined reference to `pthread_once@GLIBC_2.34'
          /usr/bin/ld: /home/linuxbrew/.linuxbrew/opt/openssl@1.1/lib/libcrypto.so.1.1: undefined reference to `dladdr@GLIBC_2.34'
          /usr/bin/ld: /home/linuxbrew/.linuxbrew/opt/openssl@1.1/lib/libcrypto.so.1.1: undefined reference to `pthread_rwlock_destroy@GLIBC_2.34'
          /usr/bin/ld: /home/linuxbrew/.linuxbrew/opt/openssl@1.1/lib/libcrypto.so.1.1: undefined reference to `pthread_key_create@GLIBC_2.34'
          /usr/bin/ld: /home/linuxbrew/.linuxbrew/opt/openssl@1.1/lib/libcrypto.so.1.1: undefined reference to `pthread_rwlock_unlock@GLIBC_2.34'
          /usr/bin/ld: /home/linuxbrew/.linuxbrew/opt/openssl@1.1/lib/libcrypto.so.1.1: undefined reference to `dlsym@GLIBC_2.34'
          collect2: error: ld returned 1 exit status
          
  = help: some `extern` functions couldn't be found; some native libraries may need to be installed or have their path specified
  = note: use the `-l` flag to specify native libraries to link
  = note: use the `cargo:rustc-link-lib` directive to specify the native libraries to link with Cargo (see https://doc.rust-lang.org/cargo/reference/build-scripts.html#cargorustc-link-libkindname)
```

なんか大量の glibc の関数が undefined reference になってる…なぜ？  
# 直前にやったこと

Linux 版 Homebrew の導入。  
Homebrew を導入する時点ではまだ Linux に Rust は導入しておらず、Homebrew でとあるパッケージをビルド＆インストールする際に rustc や cargo なども一緒にインストールされました。（結局こっちも同様の症状でビルドに失敗したけど…）  

その後、以下のようにして Rust の再インストールを試み、Rust のインストール自体は成功したもののビルドエラーは解決せず。  
```bash
$ curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
$ source "$HOME/.cargo/env"
```

# 解決策

エラーを見る限り、各ライブラリのパスが``linuxbrew``配下になっているので、まあ Homebrew が原因かなということで、とりあえず Homebrew をアンインストール。  
```bash
$ /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/uninstall.sh)"
```

このとき、``.linuxbrew/Celler/``もきちんと削除している様子が確認できました。  
```bash
Warning: This script will remove:
/home/linuxbrew/.linuxbrew/Caskroom/
/home/linuxbrew/.linuxbrew/Cellar/
/home/linuxbrew/.linuxbrew/Homebrew/
/home/linuxbrew/.linuxbrew/Homebrew/.devcontainer/
/home/linuxbrew/.linuxbrew/Homebrew/.dockerignore
/home/linuxbrew/.linuxbrew/Homebrew/.editorconfig
/home/linuxbrew/.linuxbrew/Homebrew/.github/
/home/linuxbrew/.linuxbrew/Homebrew/.gitignore
/home/linuxbrew/.linuxbrew/Homebrew/.shellcheckrc
/home/linuxbrew/.linuxbrew/Homebrew/.sublime/
/home/linuxbrew/.linuxbrew/Homebrew/.vale.ini
/home/linuxbrew/.linuxbrew/Homebrew/.vscode/
/home/linuxbrew/.linuxbrew/Homebrew/CHANGELOG.md
/home/linuxbrew/.linuxbrew/Homebrew/CONTRIBUTING.md
/home/linuxbrew/.linuxbrew/Homebrew/Dockerfile
/home/linuxbrew/.linuxbrew/Homebrew/LICENSE.txt
/home/linuxbrew/.linuxbrew/Homebrew/Library//
/home/linuxbrew/.linuxbrew/Homebrew/README.md
/home/linuxbrew/.linuxbrew/Homebrew/bin/brew
/home/linuxbrew/.linuxbrew/Homebrew/completions/
/home/linuxbrew/.linuxbrew/Homebrew/docs/
/home/linuxbrew/.linuxbrew/Homebrew/manpages/
　　　　　　　　　　　︙
```

ただし以下のディレクトリは自動では削除されず。  
```bash
The following possible Homebrew files were not deleted:
/home/linuxbrew/.linuxbrew/bin/
/home/linuxbrew/.linuxbrew/etc/
/home/linuxbrew/.linuxbrew/include/
/home/linuxbrew/.linuxbrew/lib/
/home/linuxbrew/.linuxbrew/opt/
/home/linuxbrew/.linuxbrew/sbin/
/home/linuxbrew/.linuxbrew/share/
You may wish to remove them yourself.
```


そして、もう一度ビルドしてみると成功。  
```bash
$ cargo run
   Compiling dptran v0.1.0 (/home/ytani0323/git/dptran)
    Finished dev [unoptimized + debuginfo] target(s) in 1.86s
     Running `target/debug/dptran`
```

うーん、Homebrew をインストールした時に path が linuxbrew 配下の方のライブラリに上書きされていて、そっちのライブラリに何らかの問題があったっぽい。path さえ元々のライブラリのファイルパスに書き換えたら、わざわざ Homebrew をアンインストールしなくても解決できるかも。
