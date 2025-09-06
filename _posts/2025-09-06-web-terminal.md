---
layout: post
title: "Webブラウザ上でDockerコンテナにアクセスできるターミナルを作る"
thumbnail: "/assets/img/thumbnails/feature-img/250906.webp"
tags: [JavaScript, Docker, Linux]
excerpt_separator: <!--more-->
---

Webブラウザからコンテナ内のターミナルにアクセスできるようにしたい！という欲が出てきたので作りました。

普通は SSH でシェルに繋げば済む話ですが、最終的には自分の Web サイト上で自作 OS を試用できるようにしたいという願望があります。v86 のように WASM で動く仮想マシンを使うアプローチもありますが、まずは実装が簡単そうなサーバサイドでコンテナを動かすアプローチで実装してみたいと思います。

<!--more-->	

# 使うもの

## クライアントサイド

- [xterm.js](https://xtermjs.org){:target="_blank"}
  - Web ブラウザ上で簡単にターミナル画面を生成できます。本当に簡単です

## サーバサイド

- Docker
  - コンテナ起動に使います。あらかじめ起動させたいイメージを用意しておきましょう。
- node.js
  - dockerode
    - Docker API のクライアント
    - Docker デーモンを操作し、node.js から Docker コンテナの生成、起動、終了、削除といった操作ができます
  - ws
    - WebSocket サーバの作成に使います。今回はコンテナ内シェルの内容は WebSocket で転送します。
  - stream
    - stdout、stderr に分けてストリーム送信するために使います。

# 実行の流れ

## クライアントサイド

- xterm.js でターミナル画面を作成する。
- WebSocket でサーバに接続し、転送されてきたデータを xterm.js のターミナル画面に書き込む。
- 入力されたデータを WebSocket サーバに転送する。

## サーバサイド

- WebSocket サーバを立てる。
- Docker コンテナを作成する。今回は仮にイメージを ``ubuntu:latest`` とする。
- Docker コンテナで ``/bin/bash`` を起動させる。
- コンテナ出力があれば WebSocket に送信する。
- WebSocket からの入力があればコンテナに渡す。
- 接続終了時にコンテナを停止して削除する。

# 実装

## クライアントサイド

```html
<!DOCTYPE html>

<html>
    <head>
      <link rel="stylesheet" href="node_modules/xterm/css/xterm.css" />
      <script src="node_modules/xterm/lib/xterm.js"></script>
    </head>
    <body>
        <div id="terminal" style="width: 100%; height: 100vh;"></div>
        <script>
            const term = new Terminal();
            term.open(document.getElementById('terminal'));
            const socket = new WebSocket("ws://localhost:3000");
            socket.onmessage = (event) => {
                term.write(event.data);
            };
            term.onData((data) => {
                socket.send(data);
            });
        </script>
    </body>
</html>
```

WebSocket サーバは localhost で動かすものとして実装していますが、ここはサーバの運用に合わせて変更してください。

## サーバサイド

```javascript
const WebSocket = require("ws");
const Docker = require("dockerode");
const { PassThrough } = require("stream");

const wss = new WebSocket.Server({ port: 3000 });
const docker = new Docker();

wss.on("connection", async (ws) => {
    try {
        const container = await docker.createContainer({
            Image: "ubuntu:latest",
            Cmd: ["/bin/bash"],
            Tty: true,
            OpenStdin: true,
            StdinOnce: false,
        });
        await container.start();

        const exec = await container.exec({
            Cmd: ["/bin/bash"],
            AttachStdin: true,
            AttachStdout: true,
            AttachStderr: true,
            Tty: true,
        });

        const stream = await exec.start({ hijack: true, stdin: true });

        const stdout = new PassThrough();
        const stderr = new PassThrough();
        container.modem.demuxStream(stream, stdout, stderr);

        stdout.on("data", (data) => {
            ws.send(data.toString("utf-8"));
        });

        stderr.on("data", (data) => {
            ws.send(data.toString("utf-8"));
        });

        ws.on("message", (msg) => {
            stream.write(msg);
        });

        ws.on("close", async () => {
            await container.stop();
            await container.remove();
        });
    } catch (error) {
        console.error("Error:", error);
        ws.close();
    }
});
```

かなりシンプルな実装です。一つずつ解説していきます。



```javascript
const wss = new WebSocket.Server({ port: 3000 });
const docker = new Docker();
```

ここでは見ての通り、WebSocket サーバの準備と Docker API の用意をしています。



```javascript
wss.on("connection", async (ws) => {
    try {
        ...
    } catch (error) {
        console.error("Error:", error);
        ws.close();
    }
});
```

ここでは WebSocket サーバの connection 待ちと、try catch でエラー処理として WebSocket close を実装しています。



```javascript
const container = await docker.createContainer({
    Image: "ubuntu:latest",
    Cmd: ["/bin/bash"],
    Tty: true,
    OpenStdin: true,
    StdinOnce: false,
});
await container.start();
```

Docker コンテナを準備する部分になります。

今回はイメージに ``ubuntu:latest`` を指定し、``/bin/bash`` が起動するようにしています。このとき、tty を有効にして標準入力を開いたままにします。



```javascript
const exec = await container.exec({
    Cmd: ["/bin/bash"],
    AttachStdin: true,
    AttachStdout: true,
    AttachStderr: true,
    Tty: true,
});
const stream = await exec.start({ hijack: true, stdin: true });
```

こちらではコンテナの起動後、再度 bash を実行しています。

``Attach[Stdin|Stdout|Stderr]`` で入出力を stream に流すようにし、最後にストリームを ``stream`` に繋げて WebSocket で入出力できる状態にしています。



```javascript
const stdout = new PassThrough();
const stderr = new PassThrough();
container.modem.demuxStream(stream, stdout, stderr);
```

``stream`` には stdout と stderr が混在して出力されるので、ここでは ``demuxStream`` を使って stdout と stderr を分けています。



```javascript
stdout.on("data", (data) => {
    ws.send(data.toString("utf-8"));
});
stderr.on("data", (data) => {
    ws.send(data.toString("utf-8"));
});
```

こちらではそれぞれ、stdout と stderr を WebSocket に送信する部分になります。UTF-8 に変換しています。



```javascript
ws.on("message", (msg) => {
    stream.write(msg);
});
```

クライアントサイドから送られてきた WebSocket からの入力を ``stream`` に書き込みます。するとコンテナ内の bash に入力が渡されます。



```javascript
ws.on("close", async () => {
    await container.stop();
    await container.remove();
});
```

最後に、WebSocket が閉じられたらコンテナを停止して削除します。

# 実行

![image-20250906180836480](../../../assets/img/post/2025-08-17-web-terminal/image-20250906180836480.webp)

こんな感じで、ブラウザからコンテナイメージ内の bash にアクセスできました。

![image-20250906180923196](../../../assets/img/post/2025-08-17-web-terminal/image-20250906180923196.webp)

こんな感じでビューワも使えます。操作感は通常のターミナルと遜色ないです。

今回はセッションごとに個別にコンテナを作成しておりますので、ブラウザでリロードしたらコンテナの内容は一切削除されます。

# おわりに

今回はあくまでも実験的な実装ですが、簡単に Web ブラウザからサーバ上の Docker コンテナへアクセスする手段を構築できました。

ゆくゆくは自作 OS を Web ページ上で公開することが最終目標ですので、コンテナ上で QEMU を立ち上げて OS を起動させる、といった流れになると思います。まあ、まだ公開する自作 OS が全然できてないんだけどね。。
