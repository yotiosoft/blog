---
layout: post
title: "VS CodeでGitHub Copilotを使うとき間違ったアカウントでログインした"
tags: [GitHub]
excerpt_separator: <!--more-->
---

GitHub Copilotが自分の研究室用のGitHubアカウントで使えるようになったので、これをVS Codeにインストールしようとしたとき、間違えて個人用のアカウント（GitHub Copilotが有効になっていないアカウント）でVS Codeにログインしてしまいました。GitHub CopilotはVS Codeで使う際にアカウントの連携が必要で、GitHub Copilotが有効化されたアカウントでログインする必要があります。  
![SnapCrab_NoName_2022-6-27_10-59-22_No-00](../../../assets/img/post/2022-6-27-GitHub CopilotをVS Codeで使うとき間違ったアカウントでログインした/SnapCrab_NoName_2022-6-27_10-59-22_No-00.png)

<!--more-->  

# 解決策

GitHub Copilotの設定を見てもVS Codeの設定を見てもアカウントの連携を解除する方法が載っておらず、Google先生に聞いてみたところ、同じ状況の人がいました。  
[VSC Copilot: logged wrong account, help! #7231 - github-community/community](https://github.com/github-community/community/discussions/7231){:target="_blank"}

このdiscussionには「VS Codeの設定ファイルを手動で削除して…」云々という回答があるのですが、さらに下の回答にもっとシンプルな方法がありました。  
VS Codeのアカウント設定からログアウトする方法です。  
![SnapCrab_NoName_2022-6-27_11-12-27_No-00](../../../assets/img/post/2022-6-27-GitHub CopilotをVS Codeで使うとき間違ったアカウントでログインした/SnapCrab_NoName_2022-6-27_11-12-27_No-00.png)  
![SnapCrab_Visual Studio Code_2022-6-27_11-12-40_No-00](../../../assets/img/post/2022-6-27-GitHub CopilotをVS Codeで使うとき間違ったアカウントでログインした/SnapCrab_Visual Studio Code_2022-6-27_11-12-40_No-00.png) 

なんだ、こんなところにアカウント設定があったのか。全然気づかなかった。  

VS Codeで間違えて連携してしまったアカウントからサインアウトして、再起動後、再び「GitHubアカウントとの連携が必要です」という表示が。  

![SnapCrab_NoName_2022-6-27_11-13-8_No-00](../../../assets/img/post/2022-6-27-GitHub CopilotをVS Codeで使うとき間違ったアカウントでログインした/SnapCrab_NoName_2022-6-27_11-13-8_No-00.png)    

これで正しいアカウントでログインし直して無事利用可能になりました。よかった。
