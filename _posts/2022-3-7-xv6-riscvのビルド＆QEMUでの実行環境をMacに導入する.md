---
layout: post
title: "xv6-riscvのビルド＆QEMUでの実行環境をMacに導入する"
tags: [Jekyll]
excerpt_separator: <!--more-->
---

今回はxv6をMacでビルド＆実行する環境を導入したいと思います。具体的には、xv6-riscvをRISC-Vクロスコンパイラでコンパイルし、QEMU上で動作させます。  
Linuxでも導入可能ですが、RISC-Vクロスコンパイラのビルドが必要になります。その反面、Macではビルド済みのクロスコンパイラがHomebrew上に用意されています。  
ちなみにWindowsでは導入できません（RISC-Vクロスコンパイラが未対応のため）。Windowsの場合はWSL等で導入できるかと思います。ここらへんはまた後日検証してみます。

<!--more-->  

# xv6とは

天下のMIT様が開発したUNIXライクな教育用OS。Version 6 UNIXをx86環境で動作するようにしたもので、OSの仕組みの学習用にシンプルに作られています。RISC-VというCPU上で動作し、C言語で記述されています。

# 参考資料

- [GitHub : mit-pdos/xv6-riscv](https://github.com/mit-pdos/xv6-riscv) : xv6-riscvのリポジトリ（READMEに導入方法記載）

- [GitHub : riscv-software-src/homebrew-riscv](https://github.com/riscv-software-src/homebrew-riscv) : macOS（Homebrew）向けRISC-V Toolchain（READMEに導入方法記載）

# 実行環境

- OS : macOS Monterey 12.2.1
- モデル : MacBook Pro 2016
- RAM : 8GB
- CPU : Intel Core i5 @ 2GHz
- エミュレータ : QEMU emulator version 6.2.0

## 注：RISC-Vのクロスコンパイラについて

今回は、xv6をビルドするために必要なクロスコンパイラriscv-toolsをインストールします。  
brewでインストール可能ですが、[riscv-software-src/homebrew-riscv](https://github.com/riscv-software-src/homebrew-riscv)のREADMEによると、macOS Monterey（macOS 12）ではビルド済みのものがインストールできるのに対し、それ以外のバージョンではビルドしてからインストールする作業が必要のようです（``brew install``で自動でビルドが実行されます）。  
よって、Monteryではビルド済みのものが公開されていますので、事前にMacをMonteryにアップグレードしておくことをおすすめします。



# 導入手順

## 1. QEMUのインストール

Macでは最初からQEMUがインストールされています。  

```bash
$ qemu-system-riscv64 --version
QEMU emulator version 6.2.0
Copyright (c) 2003-2021 Fabrice Bellard and the QEMU Project developers
```

インストールされてなかった場合はbrewでインストール可。

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

ビルド環境が整ったら、

```bash
$ make qemu
```

でxv6のビルドと実行が可能です。  

ターミナル上でxv6が実行されます。  
![スクリーンショット 2022-03-04 23.20.22](../../../assets/img/post/スクリーンショット 2022-03-04 23.20.22.png)

上記はxv6上でlsコマンドを実行している様子。``xv6 kernel is booting``から下がQEMUでxv6を実行している際のコンソールです。終了時は``C-A x``（``controlキー + A同時押し``→``x``）でQEMUを終了します。

# おわりに

せっかく導入したので、なんか色々実装して遊んでみたいと思います。つづくかも？
