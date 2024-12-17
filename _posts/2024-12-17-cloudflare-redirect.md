---
layout: post
title: "Cloudflare Redirect Rulesで移転先サイトにリダイレクトさせる"
tags: [ドメイン, Cloudflare, Jekyll]
excerpt_separator: <!--more-->
---

当ブログの blog.yotiosoft.com から blog.yotio.jp への移転に伴い、旧 URL から新 URL への移転が必要になりました。

いくつかリダイレクトさせる方法があるのですが、Cloudflare Redirect Rules を使うのが一番良さそうだったので、そのときのメモです。

<!--more-->

前回：[URL移転しました \| 為せばnull](https://blog.yotio.jp/2024/12/14/url-moved.html)

# リダイレクト元・リダイレクト先の概要

このブログはもともと blog.yotiosoft.com という URL でした。

下記のように、ドメインの配下に年月日の順でディレクトリが、続いて記事の html ファイルが続きます。

```
https://blog.yotiosoft.com/2024/12/14/url-moved.html
```

移転後はドメインだけ変わり、ドメイン配下の構造は変わりません。

```
https://blog.yotio.jp/2024/12/14/url-moved.html
```

この場合、ドメインだけ移転先に置き換えてディレクトリ等はそのままリダイレクトさせるページごとのリダイレクトが重要になります。

こうしないと外部サイトから当サイトの記事へのリンクが全てリンク切れを起こしてしまいます。IPA のアドレス変更でリダイレクトが行われずにリンク切れを起こしまくって一騒動があったのは記憶に新しいですね。

## 制約条件

このブログは GitHub Pages で運営していて、ブログは Jekyll で生成しています。ここでこんな制約条件が出てきます。

- .htaccess が使えない。
  - GitHub Pages ではセキュリティの観点から .htaccess をサポートしていないし、ゆえに HTTP 3xx（301 含む）でリダイレクトができない。ただし、JavaScript でリダイレクトさせる方法なら実現可能（この場合は HTTP 200 OK でリダイレクトされる）。

- HTTP [301 Moved Permanently](https://developer.mozilla.org/ja/docs/Web/HTTP/Status/301){:target="_blank"} でリダイレクトしなければならない。
  - 上記と食い違ってしまうのだが、Google Search Console ではアドレス変更を Google に通知する際に、301（あるいは 308 etc.）でリダイレクトされていなければならない。（参考：[https://support.google.com/webmasters/answer/9370220?hl=ja](https://support.google.com/webmasters/answer/9370220?hl=ja){:target="_blank"}


# リダイレクト方法の候補

自分なりに考えたリダイレクトの実現方法はこんな感じです。

## (1) 1ページずつ手作業でリダイレクト処理を実装してく（静的なリダイレクト）

JavaScript の ``window.location.href()`` などを利用して、各記事の URL に対応するリダイレクト用のページを用意していく方法です。

- 一番シンプルなやり方だが、150記事近くあるので面倒
- HTTP 301 Moved Permanently ではなく HTTP 200 OK でリダイレクトされることになる。ゆえに Google Search Console のアドレス変更ツールが使えない。

## (2) Jekyll のリダイレクト機能を使う

Jekyll 公式のプラグインである ``jekyll-redirect-from`` を使う方法です。

- こちらも結局1ページずつリダイレクト先を指定しなければならないっぽい
- (1) と同様、 HTTP 200 OK でリダイレクトされることになる

## (3) 404 Not Found を利用する

index ページ以外のページ（つまり各記事のページ）でわざと 404 Not Found が起きる状態にしておき、404 ページからリダイレクトさせる方法です。

404 ページでは ``location.href`` で現在の URL を取得し、その後 ``replace()`` でドメインを移転先に置き換え、``window.location = 移転先URL`` でリダイレクトさせます。

- 1ページずつリダイレクト先を指定しなくて良いので楽
- しかし、HTTP 404 Not Found でリダイレクトされることになる

## (4) 自宅サーバで動的にリダイレクトさせる

自宅サーバで ``.htaccess`` を使い、301 Moved Permanently リダイレクトさせます。

- 確実な方法だが、自宅サーバが常に元気な状態とは限らない
- サーバを公開しなければならないし、全て自分で管理しなければならないのでセキュリティ面が心配

## (5) Cloudflare Redirect Rules（旧 Page Rules）で動的にリダイレクトさせる

ここまで来てようやくたどり着いたのがこちらの方法です。

Cloudflare Rules を利用すると、自宅サーバを用意する必要なく、Cloudflare のサーバ側で動的にリダイレクトしてくれます。

もともとリダイレクトできるサービスとして提供されていた Cloudflare Page Rules の廃止がアナウンスされていたようですが、新しく Redirect Rules として更に柔軟に設定できるようになって提供されています。

- 参考：[Page Rules のリタイアメントと新しいルールへの自動移行について](https://zenn.dev/kameoncloud/articles/0ea8889483d6c2){:target="_blank"}

# Cloudflare Redirect Rules でリダイレクトさせる

ここからは Cloudflare Redirect Rules を利用してリダイレクト設定するまでにやったことを書いていきます。

## 0. 前提条件

Cloudflare 側のサーバでリダイレクトされますので、リダイレクト元のドメインは Cloudflare を経由するようにしなければなりません。具体的には、**リダイレクト元ドメインを Cloudflare のネームサーバに設定しておき、**Cloudflare の DNS プロキシを経由するよう設定しておく必要があります。

[https://www.cloudflare.com/ja-jp/application-services/products/dns/](https://www.cloudflare.com/ja-jp/application-services/products/dns/){:target="_blank"}

筆者は移転元ドメイン（yotiosoft.com）を [Cloudflare Registrar](https://www.cloudflare.com/ja-jp/products/registrar/){:target="_blank"} で管理していますので、ネームサーバも既定で Cloudflare のネームサーバになっています。しかし、移転元ドメインが別のレジストラ（お名◯.com など）であっても、ドメイン移管することなく無料で Cloudflare の DNS サービスが利用できます。

## 1. Cloudflare Rules でリダイレクト設定を登録する

ドメインが Cloudflare に登録できたら、Cloudflare Rules で移転元から移転先へのリダイレクトルールを登録しておきます。

まずは Cloudflare にログインし、アカウント ホーム画面から移転元の独自ドメインを選択→「ルール」を選択

![image-20241217175127272](../../../assets/img/post/2024-12-16-google-search-console/image-20241217175127272.webp)

![image-20241217175227930](../../../assets/img/post/2024-12-16-google-search-console/image-20241217175227930.webp)

Cloudflare Rules の画面が現れたら、「別のドメインにリダイレクトする」の「ルールを作成する」を選択

![image-20241217175409903](../../../assets/img/post/2024-12-16-google-search-console/image-20241217175409903.webp)

ここでリダイレクトルールを登録します。

- ルール名は任意で OK です。

- リダイレクトする際の条件などを細かく設定できますが、今回の ``https://blog.yotiosoft.com/2024/12/14/url-moved.html`` → ``https://blog.yotio.jp/2024/12/14/url-moved.html ``のような URL の一部分の置き換えを要するようなリダイレクトを行う場合は、「ワイルドカード パターン」を使います。
- 「リクエスト URL」には移転元の URL を登録します。置き換えるドメイン部分だけを記述し、残りの部分は ``*`` でワイルドカードとして表記しておきます。
- 「ターゲット URL」には移転先の URL を登録します。1つ目のワイルドカード ``*`` に対応する部分を ``${1}``、2つ目のワイルドカード ``*`` に対応する部分を ``${2}``、⋯として記述します（今回の場合はワイルドカードは1つのみ）。
- ステータスコードは 301、302、303、307、308 から選べます。Google Search Console でアドレス変更ツールを利用する場合は 301 が推奨されていますので、今回は 301 に設定しておきます。
- URL がクエリを伴う場合は「クエリ文字列を保持する」にチェックを入れます。

![image-20241217175831545](../../../assets/img/post/2024-12-16-google-search-console/image-20241217175831545.webp)

![image-20241217180310141](../../../assets/img/post/2024-12-16-google-search-console/image-20241217180310141.webp)

一番下の「展開」を選択すると、「新しいルールを確認する」という注意書きが出てきます。日本語が少しおかしいですが、リクエスト URL のドメインを DNS レコードにきちんと登録しておけよ、という注意書きです。これについては 2. で設定します。

![スクリーンショット 2024-12-14 041400](../../../assets/img/post/2024-12-16-google-search-console/スクリーンショット 2024-12-14 041400.webp)

最後に、「有効」にチェックが入っていることを確認します。

![image-20241217180602240](../../../assets/img/post/2024-12-16-google-search-console/image-20241217180602240.webp)

## 2. DNS レコードを更新する

移転元ドメインを「A レコード」に変更し、DNS プロキシを有効にしておきます（ここ重要）。プロキシを通すことでドメインへのアクセス時に Cloudflare の DNS サーバを経由するようになりますので、その際に移転先 URL に転送されるようになります。



まずは Cloudflare アカウントページからドメインを選択し、DNS レコード設定を開きます。

![image-20241217181120466](../../../assets/img/post/2024-12-16-google-search-console/image-20241217181120466.webp)

DNS 管理画面で移転したいドメインのレコード設定を開きます（なければ設定を追加）。

ここで重要なのが、

- レコードタイプは「A」にする
- IPv4 アドレスは使われていない適当なアドレスにしておく（例：192.0.0.1）
- プロキシ ステータスは「プロキシ済み」にしておく

という点です。「使われていない適当なアドレスにしておく」点がなんとなく気持ち悪いですが、入力必須なので仕方ありません。プロキシサーバによって移転先 URL にリダイレクトされますので、実際にはこの IP アドレスに飛ばされることはありません。

![image-20241217181237155](../../../assets/img/post/2024-12-16-google-search-console/image-20241217181237155.webp)

最後に保存して完了。少し待つと 301 リダイレクトされるようになります。

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

![image-20241217182731512](../../../assets/img/post/2024-12-16-cloudflare-rules-redirect/image-20241217182731512.webp)

# おわりに

サイト移転時に意外と面倒くさそうなリダイレクト。Cloudflare Rules のおかげで簡単に実現できました。
