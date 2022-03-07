---
layout: post
title: "xv6-riscvのビルド＆QEMUでの実行環境をMacに導入する"
tags: [Jekyll]
excerpt_separator: <!--more-->
---

今回はxv6をMacでビルド＆実行する環境を導入したいと思います。具体的には、xv6-riscvをRISC-Vクロスコンパイラでビルドし、QEMU上で動作させます。  
Linuxでも導入可能ですが、クロスコンパイラのビルドが必要になるようです。その反面、Macではビルド済みのクロスコンパイラがHomebrew上に用意されています。

<!--more-->  

# xv6とは

天下のMIT様が開発したUNIXライクな教育用OS。Version 6 UNIXをx86環境で動作するようにしたもので、OSの仕組みの学習用にシンプルに作られています。C言語で記述され、RISC-Vで動作します。

# 実行環境

- OS : macOS Monterey 12.2.1
- モデル : MacBook Pro 2016
- RAM : 8GB
- CPU : Intel Core i5 @ 2GHz
- Homebrew : version 3.4.0
- エミュレータ : QEMU emulator version 6.2.0

## 注：RISC-Vのクロスコンパイラについて

今回は、xv6をビルドするために必要なクロスコンパイラriscv-toolsをインストールします。  
brewでインストール可能ですが、[riscv-software-src/homebrew-riscv](https://github.com/riscv-software-src/homebrew-riscv)のREADMEによると、macOS Monterey（macOS 12）ではビルド済みのriscv-toolsがインストールできるのに対し、それ以外のバージョンではビルドしてからインストールする作業が必要のようです。riscv-toolsには膨大な数のsubmoduleが含まれているので、結構時間を要するかと思われます。  
よって、Monteryではビルド済みのものが公開されていますので、事前にMacをMonteryにアップグレードしておくことをおすすめします。



# 導入手順

## 1. QEMUのインストール

xv6-riscvを動作させるためのエミュレータQEMUを導入しますが、Macでは最初からQEMUがインストールされています。  

```bash
$ qemu-system-riscv64 --version
QEMU emulator version 6.2.0
Copyright (c) 2003-2021 Fabrice Bellard and the QEMU Project developers
```

もしもインストールされていなかったらHomebrewでインストール可。

```bash
$ brew install qemu
```

## 2. RISC-Vのクロスコンパイラの導入

xv6をビルドするためにRISC-Vのクロスコンパイラをインストールします。  
なお、riscvのリポジトリから導入するため、予め``riscv-software-src/riscv``をbrew tapしておきます。

```bash
$ brew tap riscv-software-src/riscv
$ brew install riscv-tools
```

## 3. xv6-riscvのクローン（or ダウンロード）

任意の場所にxv6-riscvのリポジトリをcloneします。

```bash
$ git clone https://github.com/mit-pdos/xv6-riscv.git
```

これでビルド環境の導入は完了。

# ビルド&実行

ビルド環境が整ったら、xv6-riscvのリポジトリをcloneした場所で

```bash
$ make qemu
```

でxv6のビルドと実行が可能です。  

完了すると、ターミナル上でQEMUでxv6-riscvが実行されます。  
![スクリーンショット 2022-03-04 23.20.22](../../../assets/img/post/スクリーンショット 2022-03-04 23.20.22.png)

上記はxv6上でlsコマンドを実行している様子。``xv6 kernel is booting``から下がQEMUでxv6を実行している際のコンソールです。終了時は``C-A x``（``controlキー + A同時押し``→``x``）でQEMUを終了します。

# おわりに

せっかく導入したので、なんか色々実装して遊んでみたいと思います。つづくかも？

# 参考資料

- [GitHub : mit-pdos/xv6-riscv](https://github.com/mit-pdos/xv6-riscv){:target="_blank"} : xv6-riscvのリポジトリ

- [GitHub : riscv-software-src/homebrew-riscv](https://github.com/riscv-software-src/homebrew-riscv){:target="_blank"} : macOS（Homebrew）向けRISC-V Toolchain（READMEに導入方法記載）
