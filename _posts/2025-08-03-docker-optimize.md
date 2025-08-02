---
layout: post
title: "無駄に肥大化したDockerイメージを縮小する"
tags: [docker, WSL2, ollama]
excerpt_separator: <!--more-->
---

Docker で一度確保された領域は、イメージ内で空き領域を増やしても、どうやらホストマシン上では自動的には開放されないらしい。ということで、今回は無駄に拡大されて開放されなくなってしまった Docker イメージ（正確には仮想マシンの VHDX）を縮小していきます。

<!--more-->	

# 背景

普段から Windows マシンで Docker 上に ollama をインストールしてローカル LLM を動かしているんですが、LLM モデルファイルは1つ1つが数 GB～数十 GB  と非常に大きいためストレージ消費が激しいです。

```powershell
> ollama ls
NAME                                           ID              SIZE      MODIFIED
phi4-mini:latest                               78fad5d182a7    2.5 GB    2 weeks ago
gemma3:12b                                     f4031aab637d    8.1 GB    2 weeks ago
deepseek-r1:14b                                c333b7232bdb    9.0 GB    2 weeks ago
phi4:latest                                    ac896e5b8b34    9.1 GB    4 weeks ago
hf.co/elyza/Llama-3-ELYZA-JP-8B-GGUF:latest    df34de06196d    4.9 GB    4 weeks ago
llama3.1:latest                                46e0c10c039e    4.9 GB    4 weeks ago
deepseek-r1-for-cline:latest                   91e9b8eb46fe    5.2 GB    7 weeks ago
deepseek-r1:latest                             6995872bfe4c    5.2 GB    7 weeks ago
mxbai-embed-large:latest                       468836162de7    669 MB    2 months ago
operator:latest                                3b55a5dcc283    9.1 GB    4 months ago
```

色々試していく中で使わなくなったモデルも出てくるわけで、そういったモデルは都度 ``ollama rm`` で削除しているんですが、どうも削除しても Windows の空き領域が増えない。

ドライブを [Diskinfo3](https://forest.watch.impress.co.jp/library/software/diskinfo/){:target="_blank"} で解析してみると、Docker の WSL2 の仮想ハードディスクファイル（``docker_data.vhdx``）が 100GB も消費しています。

![image-20250803024650001](../../../assets/img/post/2025-08-03-docker-optimize/image-20250803024650001.webp)

image サイズは 5.2GB 程度、展開後の仮想マシンの消費サイズでさえ 48GB ほどですので明らかに過剰に確保されています。

```powershell
> docker images
REPOSITORY      TAG       IMAGE ID       CREATED        SIZE
ollama/ollama   latest    2ea3b768a8f2   2 months ago   5.22GB
```

```bash
root@4b9355386e76:/# df -h
Filesystem      Size  Used Avail Use% Mounted on
overlay        1007G   48G  909G   5% /
```

# vhdx を縮小する

docker が WSL2 で動いている場合、vhdx は自動で最適化されないようです。というわけで vhdx の最適化を実行します。

vhdx はコンテナ起動中の場合ロックされていますので、まずは WSL2 を一旦シャットダウンします。

```powershell
> wsl --shutdown
```

次に、``Optimize-VHD`` で圧縮を開始します。

```powershell
Optimize-VHD -Path C:\Users\[ユーザ名]\AppData\Local\Docker\wsl\disk\docker_data.vhdx -Mode full
```

開始すると、しばらく「仮想ディスクの圧縮」画面が表示されます。

![実行中](../../../assets/img/post/2025-08-03-docker-optimize/fewew.webp)

完了すると表示が消えます。

# 実行結果

縮小した結果、docker の VHDX ファイルは 100GB → 47GB にまで縮小されました。半分以上無駄に確保されていたんですね。

![image-20250803025920660](../../../assets/img/post/2025-08-03-docker-optimize/image-20250803025920660.webp)

この後は普通に docker イメージを起動すれば OK です。先程 WSL2 をシャットダウンしましたが、イメージ起動と同時に WSL2 も再開します。
