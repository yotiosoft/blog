---
layout: post
title: "RTL-SDRで予約録音できるようにする"
tags: [Linux, RTL-SDR, Raspberry Pi]
excerpt_separator: <!--more-->
---

RTL-SDR を購入して10ヶ月、すっかりラジオ代わりとして普段使いしております。広い周波数帯を聞けることが RTL-SDR のメリットですが、結局は FM ラジオの受信がほとんどです。

普段はラズパイ（Linux）をサーバとして RTL-SDR のドングルを USB 接続しており、rtl\_tcp を使って LAN 内の Windows マシンや Mac マシンなどのクライアントソフトで受信して聞いています。

毎週水曜に深夜ラジオ番組を聞くんですが、就職に伴いリアタイできなくなってしまいました。そこで予約録音したいのですが、普段使っている SDR# で予約録音するにはクライアント側、すなわち Windows マシンを録音する深夜に立ち上げておかなければなりません。

そこで、Linux 向けの RTL-SDR コマンドツールとシェルスクリプトを組み合わせ、指定した日時から指定した時間分の録音をできるようにしました。

<!--more-->	

# 環境

- Raspberry Pi 5
  - Ubuntu 24.04.2 LTS
  - RAM 8GB

- RTL-SDR Blog V4

ドングルは RTL-SDR Blog V4 です。購入時の概要やセットアップについては下記の記事をご覧ください。

- [RTL-SDR Blog V4を買った \| 為せばnull](https://blog.yotio.jp/2024/08/25/rtl-sdr.html)

# 使うコマンドツール

今回は「**softfm**」という RTL-SDR クライアントソフトを録音用に採用しました。

RTL-SDR での FM 受信によく使われるツールとしては rtl\_fm がありますが、こちらのクライアントソフトではモノラル音声でしか受信できません。対して softfm ではステレオ音声での受信に対応していたため、今回はこちらを採用することとしました。

さらに、softfm には録音したファイルを出力する機能もあります。



ただ、残念ながら softfm、優秀かつそれなりに利用されてきたツールにも関わらず、開発者様本人によるリポジトリが消滅しています（なぜ…）。

- [https://github.com/jorisvr/SoftFM](https://github.com/jorisvr/SoftFM){:target="_blank"}

色々と事情はあるのでしょうが、どうしても使いたかったので自己責任で fork 版のリポジトリから拝借しました。

- [https://github.com/zf-lab/SoftFM](https://github.com/zf-lab/SoftFM){:target="_blank"}

commit history を見る限りだと開発者様のコミットのみですので、history が改竄されていない限りは変な改造はされていない…はずです。

# 今回作ったスクリプト

開始日時、録音時間、保存先、周波数を指定すると、その時間に指定された周波数で softfm を呼び出すスクリプトです。

```shell
start_date=$1
rec_minutes=$2
rec_dir=$3
freq=$4

rec_file=$(date -d "$start_date" +%Y%m%d%H%M%S).wav
rec_file_path=$rec_dir/$rec_file
rec_seconds=$(($rec_minutes*60))

# If the recording directory does not exist, create it
if [ ! -d $rec_dir ]; then
    mkdir -p $rec_dir
fi

# If the recording file exists, rename
if [ -f $rec_file_path ]; then
    mv $rec_file_path $rec_file_path.bak
fi

# wait until the start time
echo $start_date
while [ $(date +%s) -lt $(date -d "$start_date" +%s) ]; do
    sleep 1
done

# Record the audio
timeout $rec_seconds softfm -f $freq -W $rec_file_path
```

ファイル名は録音日時から自動生成します。もし被っているファイルがあったらバックアップファイルを作ります（が、普通の使い方をしていれば被ることはないはずです）。

softfm は強制終了するとその時点で録音を停止します。よって、``timeout`` コマンドで必要な時間分 softfm を立ち上げておき、録音終了時間になったら softfm を終了させます。

## 使用方法例

```bash
./rec.sh "2025/05/01 02:00:00" 120 /mnt/usbhdd1/rtl_sdr_rec/ 79.95M
```

午前2時から2時間、80 MHz で録音させる場合です。79.95M となっていますが、これはちょっとした softfm 側のクセで、少し周波数を前にズラさないと雑音が入ってしまうためです。少し受信周波数がズレているのかもしれません。

# おわりに

いまのところ一度も失敗することなく利用できています。RTL-SDR、今のところ FM 受信と短波受信くらいにしか使えていませんが、PC や Linux で扱えるため自分でスクリプトを組んであれこれ工夫できるのが良いところですね。
