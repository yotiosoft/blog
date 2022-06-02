---
layout: post
title: "Jekyll(on GitHub Pages)でブログ更新しても記事が反映されない"
tags: [Jekyll, GitHub]
excerpt_separator: <!--more-->
---

昨日、いつもどおりブログに記事を投稿しようとしてGitHubにMarkdownファイルをpushしたら、なかなか記事が反映されない。このブログはJekyllを使ってGitHub Pages上で運営しており、mdファイルを置くとGitHub Actionsが自動でビルドを行い、ブログに記事を反映してくれる仕組みになっています。  

で、GitHub Actionsを見てみると、全く動いていない。mdファイルを置いたにもかかわらず、ビルドが進まない。All workflowsに出てくるのは先日完了したビルドだけ。  
![スクリーンショット 2022-06-01 18.43.38](../../../assets/img/post/2022-5-24-Jekyll/スクリーンショット 2022-06-01 18.43.38.png)  

<!--more-->  

先生ー、何もしてないのに壊れましたー！

# 原因はGitHub Actionsのサーバー障害

あれこれ調べた上で、ふと、この前もGitHubで大規模なサーバー障害が起きていたことを思い出したので、もしやと思って[GitHub Status](https://www.githubstatus.com){:target="_blank"}を見てみたら、GitHub Actions限定でサーバー障害が起きてた。思わぬ盲点。  
![スクリーンショット 2022-06-01 18.48.20](../../../assets/img/post/2022-5-24-Jekyll/スクリーンショット 2022-06-01 18.48.20.png)  
なるほど、GitHub Actionsで障害が起きると記事を更新しても全く反応しなくなるのか。暫く待つと復旧し、GitHub Actionsで記事のビルドが始まりました。  

Jekyll + GitHub Pagesで何もいじってないのに唐突に更新できなくなった、という状況になったら、まずはGitHub Statusを見ておくのがいいかもしれませんね。

