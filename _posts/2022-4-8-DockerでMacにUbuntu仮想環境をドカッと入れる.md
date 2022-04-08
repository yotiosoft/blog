---
layout: post
title: "DockerでMacにUbuntu仮想環境をドカッと入れる"
tags: [Mac, Ubuntu]
excerpt_separator: <!--more-->
---

お久しぶりです。最近ブログに書けるネタが少なくて困っていたところ、研究室での活動でMacBook上でUbuntuを動かす必要が出てきました。そこで今回は、Dockerを使ってMacにUbuntuの仮想環境を導入する様子を備忘録として書いておきたいと思います。

<!--more-->  

# 使用環境

- MacBook Pro 2016
- OS: macOS Monterey 12.2.1
- CPU: intel i5 6360U
- RAM: 8GB

# 手順

## 1. Dockerをインストール

[Install Docker Desktop on Mac \| Docker Documentation](https://docs.docker.com/desktop/mac/install/){:target="_blank"}  

上記のダウンロードページに行き、使用機種のCPUに応じて「Mac with Intel chip」または「Mac with Apple chip」のいずれかを選んでダウンロードします。今回はIntel機に導入するのでIntel chipを選びました。ダウンロードに要する時間は5分ほど。  

ダウンロードが完了したらDocker.dmgを開き、Docker.appをアプリケーションフォルダに突っ込みます。ファイルサイズは1.62GBとやや大きめです。  


![スクリーンショット 2022-04-08 16.36.36](../../../assets/img/post/スクリーンショット 2022-04-08 16.36.36.png)  
初回起動時はヘルパーツールのインストールのためアクセス権が要求されます。Macのパスワードを入力すればOKです。  

![スクリーンショット 2022-04-08 16.44.19](../../../assets/img/post/スクリーンショット 2022-04-08 16.44.19.png)

インストールが完了したら利用規約が表示され、Agreeボタンを押すとDockerが利用可能になります。  

以降、Docker起動中はメニューバーにDockerのアイコンが表示され、端末でdockerコマンドが利用可能になります。

## Ubuntuのインストール

[Docker Hub](https://hub.docker.com/_/ubuntu/){:target="_blank"}に利用可能なバージョンが掲載されているので、ここから一つ選んでインストールします。  
今回は最新のUbuntu 22.04を選びました。  

端末を起動して以下のコマンドを実行。  

```bash
$ docker pull ubuntu:22.04
```

1分とかからずにインストールは完了。  

``$docker images``で正常にインストールできたかどうか確認できます。

```bash
$ docker images
REPOSITORY               TAG       IMAGE ID       CREATED       SIZE
ubuntu                   22.04     f0b07b45d05b   2 days ago    77.9MB
```

## Ubuntuの起動

### 起動

``--name``で任意のコンテナ名を設定します。今回は無難にubuntuとしておきました。

```bash
$ docker run -it -d --name ubuntu ubuntu:22.04
```

### 起動の確認

```bash
$ docker ps
```

実行結果にubuntuが載っていればOK。

### コンテナに入る

Ubuntuの環境に入ります。コンテナに入ることで、Ubuntuコマンドが利用可能になります。

```bash
$ docker exec -it ubuntu /bin/bash
```

以降、bashがUbuntuに入り、入力がUbuntuコンテナ上で実行されます。  

```bash
root@97f851a88bc0:/# ls
bin  boot  dev  etc  home  lib  lib32  lib64  libx32  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
root@97f851a88bc0:/# pwd
/
```

### 終了する

まずはコンテナから出て、

```bash
root@97f851a88bc0:/# exit
```

コンテナを停止させます。  

```bash
$ docker stop ubuntu
```

停止後、コンテナを再開する場合は``$ docker start``を実行し、もう一度``$ docker exec``でコンテナに入ります。

```bash
$ docker start ubuntu
```



# インストール後にやったこと

最低限のUbuntu環境しか提供されていないので、普段よく使うツールは導入されていません。  
とりあえず、以下の環境をインストールしておきました。  

- エディタ（Vim、Emacsなど）
- git
- gcc, g++

とりあえず今やりたいこと（C++プログラムをコンパイルして実行してgit commitしてGitHubにpush）はできました。起動も早く、特に不具合なく快適に動きました。
