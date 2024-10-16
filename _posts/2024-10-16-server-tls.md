---
layout: post
title: "自宅サーバ用にSSL/TLS証明書を発行した"
tags: [開発日記, Linux, Ubuntu, LINE, ドメイン]
excerpt_separator: <!--more-->
---

前回の記事では LINE Messaging API を利用して、コマンドラインからプッシュメッセージを送信するツールを作成しました。

- [LINE Messaging API #1 Rustで簡単なLINEメッセージ送信ツールを作る \| 為せばnull](../../../2024/09/23/mes.html){:target="_blank"}

その発展形として Webhook を利用して bot サーバを作ろうとしたのですが、Webhook を利用するにはサーバの SSL/TLS 証明書が必要です。

- [Webhook送信元のSSL/TLS仕様 \| LINE Developers](https://developers.line.biz/ja/docs/messaging-api/ssl-tls-spec-of-the-webhook-source/){:target="_blank"}

というわけで、今回は自宅サーバに向けて SSL/TLS を無料で発行する手順についてメモっておきます。



※ 今回は LINE Messaging API シリーズに関連しているといえばしているのですが、独立した話題になりますので別記事として書きました。

<!--more-->

# サーバの概要

- Raspberry Pi 5
  - OS : Ubuntu 23.10 (Linux kernel ver.6.5.0, 64bit)
  - CPU : BCM2835
  - RAM : 8GB

# Let's Encrypt を使う

SSL/TLS 証明書を手に入れるには認証局での認証処理が必要です。認証には手数料が有料の場合が多いですが、Let's Encrypt であれば無料で発行できます。

- [Let's Encrypt - フリーな SSL/TLS 証明書](https://letsencrypt.org/ja/){:target="_blank"}

## ドメインの入手

認証したいドメインを予め取得しておきます（サブドメインも可）。以降、``[YOUR DOMAIN HERE]`` と示します。

## 手順

### 前準備：ポートを開放しておく

サーバ側で 80番ポートを開放し、ルータ側でポートマッピングを設定しておきます。

```bash
$ sudo ufw allow 80
```

### 前準備：サーバアプリケーション（apache2 など）を停止しておく

後ほど使う ``certbot`` は TCP 80 番ポートを認証時に利用します。

もしサーバ側で既に apache2（または nginx）などのサーバアプリケーションが実行されている場合は、予め停止しておいてください。

```bash
$ sudo systemctl stop apache2
```

### certbot のインストール＆認証

まずは証明書生成ツールである ``certbot`` をインストール。

```bash
$ sudo apt install certbot
```

インストールが完了したらツールを起動します。

```bash
$ sudo certbot certonly --standalone -d [YOUR DOMAIN HERE]
```

初回起動時はメールアドレスの入力、同意書への同意を求められます。

```bash
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Enter email address (used for urgent renewal and security notices)
 (Enter 'c' to cancel): [YOUR E-MAIL ADDRESS HERE]
```

```bash
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please read the Terms of Service at
https://letsencrypt.org/documents/LE-SA-v1.4-April-3-2024.pdf. You must agree in
order to register with the ACME server. Do you agree?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o:
```

次に証明書の発行完了後にメールアドレスの共有をしても良いか？と聞かれます。

```bash
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Would you be willing, once your first certificate is successfully issued, to
share your email address with the Electronic Frontier Foundation, a founding
partner of the Let's Encrypt project and the non-profit organization that
develops Certbot? We'd like to send you email about our work encrypting the web,
EFF news, campaigns, and ways to support digital freedom.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o:
```



これにて初期設定は完了です。続いてドメインの認証処理が開始されます。

```bash
Account registered.
Requesting a certificate for [YOUR DOMAIN]
```

ここで DNS のチェック等が入ります。もしドメインからサーバまで 80 番ポートでアクセスできない場合、ここで失敗します。

```bash
Certbot failed to authenticate some domains (authenticator: standalone). The Certificate Authority reported these problems:
  Domain: [YOUR DOMAIN]
  Type:   connection
  Detail: xxx.xxx.xxx.xxx: Fetching http://[YOUR DOMAIN]/.well-known/acme-challenge/...: Timeout during connect (likely firewall problem)
```

成功したら証明書のファイルパスが表示されます。

```bash
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/[YOUR DOMAIN]/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/[YOUR DOMAIN]/privkey.pem
This certificate expires on 2025-01-14.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

以降、この証明書を使って SSL/TLS 通信が可能です。

例えば、Rust の Actix web の場合：

```rust
#[post("/")]
async fn index() -> impl Responder {
    ...
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    rustls::crypto::aws_lc_rs::default_provider()
        .install_default()
        .unwrap();

    let mut certs_file = BufReader::new(File::open("fullchain.pem").unwrap());		// ← これと
    let mut key_file = BufReader::new(File::open("privkey.pem").unwrap());			// ← これ
    
    // load TLS certs and key
    // to create a self-signed temporary cert for testing:
    // `openssl req -x509 -newkey rsa:4096 -nodes -keyout key.pem -out cert.pem -days 365 -subj '/CN=localhost'`
    let tls_certs = rustls_pemfile::certs(&mut certs_file)
        .collect::<Result<Vec<_>, _>>()
        .unwrap();
    let tls_key = rustls_pemfile::pkcs8_private_keys(&mut key_file)
        .next()
        .unwrap()
        .unwrap();
    
    // set up TLS config options
    let tls_config = rustls::ServerConfig::builder()
        .with_no_client_auth()
        .with_single_cert(tls_certs, rustls::pki_types::PrivateKeyDer::Pkcs8(tls_key))
        .unwrap();

    HttpServer::new(|| {
            App::new()
                .service(index)
        })
        .bind_rustls_0_23(("0.0.0.0", 443), tls_config)?
        .run()
        .await
}
```



証明書の有効期限は3ヶ月間です。自動更新をさせたい場合はこちら ↓ の記事が参考になるかと思います。

- [Let's Encrypt の証明書の更新を自動化する手順 (cron) ](https://weblabo.oscasierra.net/letsencrypt-renew-cron/){:target="_blank"}

# おわりに

次回はこれを利用して、Rust で LINE Messaging API の Webhook サーバを作っていきます。
