---
layout: post
title: "adbでAndroid端末の写真をLinux端末にバックアップする"
tags: [Linux, Android]
excerpt_separator: <!--more-->
---

ストレージ容量の少ない Android スマホを使っていると頻繁に写真や動画データをバックアップしなければならなくなるのですが、バックアップ中に起こりがちなのがバックアップ中の切断。おま環かもしれないですが、物理的には接続されているのにコピー中にビジー状態となって勝手に切断されたり、何らかの理由でコピーに失敗することが多いです。

<!--more-->	

# 今回の方針

adb (Android Debug Bridge) を使ってコピーさせてみます。自分の環境では adb であれば数十 GB 程度のコピーでも安定して実行できました。

# 実行環境

- スマートフォン
  - Pixel 7 128GB
  - Android 16

- バックアップ先
  - Raspberry Pi 5
  - Ubuntu 24.04.3 LTS
  - Linux 6.8.0
  - adb: Android Debug Bridge version 1.0.41


# 手順

## Android デバイス側：USB デバッグを ON にする

Android デバイスの「設定」→「システム」→「開発者向けオプション」で

- 「開発者向けオプションを使用」
- 「USB デバッグ」

を ON にします。

<img src="../../../assets/img/post/2026-01-04-adb-copy/Screenshot_20260415-215813.webp" alt="Screenshot_20260415-215813" style="zoom:50%;" />

「USB デバッグを許可しますか？」と聞かれたら「許可」をタップします。

<img src="../../../assets/img/post/2026-01-04-adb-copy/Screenshot_20260415-223107.webp" alt="Screenshot_20260415-223107" style="zoom:50%;" />

## バックアップ先：準備

まずは `adb` をインストール。

```bash
sudo apt install adb
```

インストールできたら Android デバイスをバックアップ先端末（Linux）に接続します。

## バックアップ先：デバイス探索

`adb devices -l` でデバイスの ID を取得します。（念のため一部隠してます）

```bash
$ adb devices -l
* daemon not running; starting now at tcp:5037
* daemon started successfully
List of devices attached
29051■■■■■■766         unauthorized usb:4-2 transport_id:1
```

## バックアップ先：コピー

ファイルコピーは次のようなコマンドで実行できます。

`$DEVICE_ID` は上記で得た Android デバイスのデバイス ID、`$SRC` は Android デバイス上のコピー元ディレクトリ、`$DST` はバックアップ先のコピー先ディレクトリです。

```bash
adb -s "$DEVICE_ID" pull "$SRC" "$DST"
```

コピー元ディレクトリは端末によりますが、自分の端末（Pixel7）の場合は写真の保存先は `/sdcard/DCIM/Camera` でした。

するとディレクトリ内の全ファイルのコピーが始まります。

```bash
$ adb -s 29051■■■■■■766 pull /sdcard/DCIM/Camera /home/ytani/backup/pictures/pixel7
[  0%] /sdcard/DCIM/Camera/PXL_20260321_234739537.jpg: 7%
```

コピーが終わったら忘れずに USB デバッグモードを OFF にしましょう。

# おまけ：コピー自動化スクリプト

デバイス ID 取得からコピーまでを自動化したスクリプトを作成しました。

```bash
# !/bin/bash
#
# はじめに USB デバッグを ON にすること。
# ただし、ON にすると銀行系のアプリが利用不可能になるため、
# 完了したらもとに戻すこと。
#

set -euo pipefail

SRC=$1
DST=$2

if [[ -z "${SRC}" ]]; then
  echo "[ERROR] Please set SRC dir"
  exit 1
fi

if [[ -z "${DST}" ]]; then
  echo "[ERROR] Please set DST dir"
  exit 1
fi

echo "[*] Detecting adb device..."

DEVICE_ID=$(adb devices -l | awk '
  NR>1 && $2=="device" && $1 !~ /^emulator/ { print $1; exit }
')

if [[ -z "${DEVICE_ID}" ]]; then
  echo "[ERROR] No usable adb device found."
  adb devices -l
  exit 1
fi

echo "[OK] Using device: $DEVICE_ID"

echo "[*] Creating destination directory:"
echo "    $DST"
mkdir -p "$DST"

echo "[*] Copying files..."
adb -s "$DEVICE_ID" pull "$SRC" "$DST"

echo "[OK] Copy completed successfully."
```

使い方：第1引数にコピー元、第2引数にコピー先を指定してください。

```bash
$ ./copy_with_dst.sh /sdcard/DCIM/Camera /home/ytani/backup/pictures/pixel7
[*] Detecting adb device...
[OK] Using device: 29051FDH200766
[*] Creating destination directory:
    .
[*] Copying files...
[  0%] /sdcard/DCIM/Camera/PXL_20260320_102006823.jpg: 2%
```

