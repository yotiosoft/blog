---
layout: post
title: ".ssh/configで複数アカウント運用するならIdentitiesOnlyを有効化すべし"
tags: [GitHub]
excerpt_separator: <!--more-->
---

去年の12月に[「Gitで複数アカウントのssh接続ができたー」みたいな記事](https://blog.yotiosoft.com/2021/12/14/%E3%82%88%E3%81%86%E3%82%84%E3%81%8Fgit%E3%81%A7%E3%81%AEssh%E6%8E%A5%E7%B6%9A%E3%81%AE%E8%A4%87%E6%95%B0%E3%82%A2%E3%82%AB%E3%82%A6%E3%83%B3%E3%83%88%E3%81%AE%E6%89%B1%E3%81%84%E6%96%B9%E3%82%92%E8%A6%9A%E3%81%88%E3%81%9F%E3%81%AE%E3%81%A7%E3%83%A1%E3%83%A2.html)を書いてましたが、最近また問題が勃発。各リポジトリのgitのconfigファイルで正しくssh接続名を指定しても、sshでユーザ切り替えをしようとしても、なぜかユーザーが切り替わりません。  

自分の環境ではGitHubアカウントごとにssh接続用の秘密鍵を使い分けているのですが、どうしても片方の秘密鍵でログインされてしまいます。  
おかしいなと思ってググったら、``~/.ssh/config``で複数アカウントを運用する上で重要な設定``IdentitiesOnly yes``が抜けていたことが判明しました。

<!--more-->  

# 問題点

gitのconfigファイルに接続先を正しく設定していても、なぜかpermission deniedになってしまう。  
さらに、sshコマンドでアカウント切り替えをしようとしても、なぜかアカウントが切り替わらない。  

例）  

```bash
$ ssh -T GitHub-YotioSoft
Hi YotioSoft! You’ve successfully authenticated, but GitHub does not provide shell access.
$ ssh -T GitHub-Labo
Hi YotioSoft! You’ve successfully authenticated, but GitHub does not provide shell access.
```

上記の例では、ホスト名を``GitHub-YotioSoft``として設定したGitHubのYotioSoft（個人用）のアカウントにsshで接続した後に、ホスト名を``GitHub-Labo``として設定したGitHubの研究室用のアカウントにssh接続しようとしています。  
見てみると、YotioSoftに接続しようとしているときに「Hi YotioSoft!」とGitHubのサーバから返ってきています。つまり、個人用のアカウントYotioSoftに接続できているということです。  
そこまでは良いのですが、研究室用アカウントに接続しようとしているときも「Hi YotioSoft!」と返ってきていますね。これではアカウントの切り替えができません。  

理想としては  

```bash
$ ssh -T GitHub-YotioSoft
Hi YotioSoft! You’ve successfully authenticated, but GitHub does not provide shell access.
$ ssh -T GitHub-Labo
Hi [研究室用のアカウント名]! You’ve successfully authenticated, but GitHub does not provide shell access.
```

このようになってほしいのですが、なぜかアカウントを切り替えようとしてもYotioSoftにログインしたままになっています。秘密鍵も正しく設定できているのに、なぜだろう？

# 解決策

``.ssh/config``に問題がありました。``IdentitiesOnly``の値を記述しておらず、デフォルト設定でIdentitiesOnly=noになっていたのが原因のようです。  

以前は、``.ssh/config``を  

```
Host GitHub-YotioSoft
    HostName github.com
    User git
    Port 22
    IdentityFile ~/.ssh/id_rsa_yotiosoft

Host GitHub-Labo
    HostName github.com
    User git
    Port 22
    IdentityFile ~/.ssh/id_rsa_labo
```

このようにしていましたが、各ホストの設定に``IdentitiesOnly yes``を追記し  

```
Host GitHub-YotioSoft
    HostName github.com
    User git
    Port 22
    IdentityFile ~/.ssh/id_rsa_yotiosoft
    TCPKeepAlive yes
    IdentitiesOnly yes

Host GitHub-Labo
    HostName github.com
    User git
    Port 22
    IdentityFile ~/.ssh/id_rsa_labo
    TCPKeepAlive yes
    IdentitiesOnly yes
```

このようにすると解決しました。  

なお、IdentitiesOnlyの他に``TCPKeepAlive yes``も追記していますが、これはssh接続の維持を行わせるための設定で、ついでに設定しておいたものなので今回の話題には直接関係はありません。

## IdentitiesOnlyとは

「IdentityFileで指定した秘密鍵のみを使用するか」という設定。yesならIdentityFileで指定した秘密鍵のみを使用。デフォルトではnoになっているようです。  

今回の原因としては、おそらく個人用の秘密鍵と研究室用の秘密鍵を使い分けている環境下で、秘密鍵の使い分けを指示しておらず、ssh内で個人用の秘密鍵が優先されてしまったのが原因だと思われます。

# まとめ

gitでssh接続で複数のアカウントを利用し、なおかつ秘密鍵を使い分けているときは、必ず``IdentitiesOnly yes``を設定しよう。設定しないとアカウントの切り替えがうまくいかない事態が起こりえます。
