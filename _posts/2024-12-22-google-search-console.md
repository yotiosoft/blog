---
layout: post
title: "Google Search Consoleのアドレス変更ツールでサイト移転を通知する"
tags: [ドメイン, Google Search Console]
excerpt_separator: <!--more-->
---

前回、前々回に引き続き、当ブログの移転時に行ったことを書き残していきます。今回は Google Search Console でアドレス変更ツールを使う編です。

<!--more-->

前回：[Cloudflare Redirect Rulesで移転先サイトにリダイレクトさせる \| 為せばnull](https://blog.yotio.jp/2024/12/17/cloudflare-redirect.html)

前々回：[URL移転しました \| 為せばnull](https://blog.yotio.jp/2024/12/14/url-moved.html)

# 概要

独自ドメイン間のアドレス移転を行ったため、Google の検索結果や Search Console の分析対象を、移転元の旧 URL から移転先の新 URL に変更してもらわなければなりません。そこで使うのがアドレス変更ツールです。

[アドレス変更ツール - Search Console ヘルプ](https://support.google.com/webmasters/answer/9370220?hl=ja){:target="_blank"}

アドレス変更ツールは Google の検索エンジンに対して同一サイトの URL 移転を通知するために提供されているものです。

実際にどの程度検索結果に反映されるのか、どれくらいの期間で反映されるのか、という点については Google のランキングアルゴリズム等が非公開のため不明ですが、まあやらないよりはマシでしょう。

## アドレス変更ツールを使うべきケース

- ドメインから別のドメインへの移転（``example.com`` から ``example2.com`` など）
- サブドメインから別のサブドメインへの移転（``test.example.com`` から ``test2.example.com`` など）
- その他、ドメイン / サブドメインの変更を伴う移転（``test.example.com`` から ``test.example2.com`` など）

## アドレス変更ツールが使えないケース

- http から https への移転
- 同じドメイン内でのディレクトリなどの移転（``example.com/abc/index.html`` から ``example.com/def/index.html`` など）
- 同じドメイン内の ``www`` とそれ以外のサブドメイン間の移転（``www.example.com`` から ``example.com`` など）
- URL 変更を伴わない移転（アドレスが変更されないホストサーバの変更など）

# （再掲）リダイレクト元・リダイレクト先の概要

このブログはもともと blog.yotiosoft.com という URL でした。

下記のように、ドメインの配下に年月日の順でディレクトリが、続いて記事の html ファイルが続きます。

```
https://blog.yotiosoft.com/2024/12/14/url-moved.html
```

移転後はドメインだけ変わり、ドメイン配下の構造は変わりません。

```
https://blog.yotio.jp/2024/12/14/url-moved.html
```

## リダイレクト方法

Cloudflare Redirect Rules を使って 301 リダイレクトを実現しています。この点については[前回](https://blog.yotio.jp/2024/12/17/cloudflare-redirect.html)の記事をご参照ください。

（結果）

```bash
$ curl -I https://blog.yotiosoft.com/2024/12/14/url-moved.html
HTTP/2 301
date: Tue, 17 Dec 2024 09:17:42 GMT
content-type: text/html
content-length: 167
location: https://blog.yotio.jp/2024/12/14/url-moved.html
cache-control: max-age=3600
expires: Tue, 17 Dec 2024 10:17:42 GMT
report-to: {"endpoints":[{"url":"https:\/\/a.nel.cloudflare.com\/report\/v4?s=wHtB43z1ag0mjjl2DesLLUqLHYkbSuCCRmg21IQZAxJEuvilvnH9xzuUG5VZ0QK0dztEsKBhhpmc81lKtDN0GxPkOukve7EDjPA0YwVRIDhjRgtUGgAVftdUqarJtoYJl7BzOk%2FQ5w6glV%2FaZC9qOL4%3D"}],"group":"cf-nel","max_age":604800}
nel: {"success_fraction":0,"report_to":"cf-nel","max_age":604800}
server: cloudflare
cf-ray: 8f35d2520ad880e7-NRT
```

# アドレス変更ツールに通知する

ここからが本題です。Google Search Console を使ってアドレス変更を Google Search Console に通知します。

## 1. 移転先 URL のプロパティを新規作成する

現在使っている移転前のプロパティを URL 変更することはできないようで、アドレス変更ツールを使う際に移転先 URL のプロパティの選択が求められます。

ですので、予め移転先 URL 用の Google Search Console プロパティを作成しておきます。

## 2. 移転元 URL のプロパティでアドレス変更を通知する

次に移転元 URL のプロパティで設定画面を開きます。そして「アドレス変更」を選択します。

![image-20241222224309132](../../../assets/img/post/2024-12-21-google-search-console/image-20241222224309132.webp)

次に「新しいサイトを選択」を選択します。

![image-20241222224958838](../../../assets/img/post/2024-12-21-google-search-console/image-20241222224958838.webp)

ここで移転先 URL のプロパティを選択します。

![image-20241222225705845](../../../assets/img/post/2024-12-21-google-search-console/image-20241222225705845.webp)

「検証して更新」をクリックします。

![image-20241222231258195](../../../assets/img/post/2024-12-21-google-search-console/image-20241222231258195.webp)

ここで移転元から移転先に 301 リダイレクトされているかのチェックが行われます。チェックに成功したら「検証に合格しました」と表示されます。

![スクリーンショット 2024-12-14 043913](../../../assets/img/post/2024-12-21-google-search-console/スクリーンショット 2024-12-14 043913.webp)

以降、「アドレス変更」画面で「このサイトは現在移行中です」と表示されます。

![スクリーンショット 2024-12-14 043937](../../../assets/img/post/2024-12-21-google-search-console/スクリーンショット 2024-12-14 043937.webp)

# 以降開始後は何したら良い？

通知が完了をしたら直ちに検索結果に現れるわけではありません。あとは Google の検索アルゴリズムのみぞ知る世界ですので、早期移行完了を願って待ちます。

Search Console ヘルプ曰く、Search Console の移転元・移転先の各プロパティを日々確認し、移転元への検索数が徐々に減少＆移転先の検索数が徐々に上昇しているかを確認するようにとのことです。

あとは移転先 URL のサイトマップも登録しておくと、より早く移転先ページがインデックス登録されるようになるかと思います。

# 移転してからどうなった？

すでに12月14日の移転開始から1週間経ちました。移転元・移転先 URL の検索数はそれぞれこんな感じです。

（移転元）

![image-20241222232712230](../../../assets/img/post/2024-12-22-google-search-console/image-20241222232712230.webp)

（移転先）

![image-20241222232742462](../../../assets/img/post/2024-12-22-google-search-console/image-20241222232742462.webp)

移転元が圧倒的にアクセス数が上回っているわけですが…うーん…1週間ならまだこんなもんなのかな…

その反面、インデックス登録は早期に完了しているページがほとんどのようで、あとは移転先ページの検索結果への早期反映を願うばかりですね。

![image-20241222232908935](../../../assets/img/post/2024-12-22-google-search-console/image-20241222232908935.webp)

# 参考文献

- [アドレス変更ツール - Search Console ヘルプ](https://support.google.com/webmasters/answer/9370220?hl=ja){:target="_blank"}
