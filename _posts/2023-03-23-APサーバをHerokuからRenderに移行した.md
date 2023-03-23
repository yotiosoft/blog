---
layout: post
title: "APサーバをHerokuからRenderに移行した"
tags: [GitHub,Heroku,Python,Yapps]
excerpt_separator: <!--more-->
---

去年の11月28日の Heroku 有料化に伴い一時的に非公開にしていた Yapps の「[乱数生成](https://yapps.yotiosoft.com/random/)」ですが、4ヶ月間もの放置を通して、本日バックエンドサーバを移し再度公開しました。

サーバの移転先はタイトルにもある通り [Render](https://render.com/){:target="_blank"} というサービスです。

<!--more-->

# Render の概要

Heroku と同じくWeb サービスを利用する上での様々なプラットフォームを備えたサービス、と捉えておけばよいかと思います。まだちょっとしか触ってないので物知り顔で語らないようにしておきます。ありがたいことに無料で利用可能です。

Render では GitHub リポジトリからデプロイして Web API を作成することができます。試した限りでは、Python のサーバアプリのデプロイが難なくできました。

# 参考にした記事

[Herokuが無料で使えなくなるのでRenderへ移行する（Renderのデプロイ） - Qiita](https://qiita.com/matsutogen/items/f29ad5c244fdca24e4cf)

基本的にはこちらの記事の通りに手順を進めれば OK です。こちらの記事では Hello World を返す簡単な Web API を作成していますが、今回は元々 Heroku でのデプロイ用に作ったリポジトリがあるので、そちらを Render でデプロイしました。

[GitHub - yotiosoft/yapps_random_api](https://github.com/yotiosoft/yapps_random_api)

↑のリポジトリを選び、Web サービスの名前やサーバの場所 (Region)、ランタイム、プランなどの設定画面へ。

![SnapCrab_NoName_2023-3-21_11-48-37_No-00.webp](../../../assets/img/post/2023-03-21/SnapCrab_NoName_2023-3-21_11-48-37_No-00.webp)

サーバの場所はオレゴン、フランクフルト、オハイオ、シンガポールから選択でき、今回は一番近いシンガポールにしておきました。

# 引っかかったこと

Heroku でデプロイしていたリポジトリをほぼそのまま Render でもデプロイさせることができましたが、一点だけ引っかかった点がありました。

![SnapCrab_NoName_2023-3-21_11-50-37_No-00.webp](../../../assets/img/post/2023-03-21/SnapCrab_NoName_2023-3-21_11-50-37_No-00.webp)

Render のデプロイ環境のバージョンが古く、numpy が version 1.21.6 までしか使えません。これは requirements.txt で numpy の要求バージョンを 1.21.6 に下げることで対応しました。ちなみに Python のバージョンは 3.7.10 でした。

![SnapCrab_NoName_2023-3-21_11-53-35_No-00.webp](../../../assets/img/post/2023-03-21/SnapCrab_NoName_2023-3-21_11-53-35_No-00.webp)

requirements.txt を書き換え、コミットして GitHub に push した後に「Deploy latest commit」を選択。

![SnapCrab_NoName_2023-3-21_11-55-19_No-00.webp](../../../assets/img/post/2023-03-21/SnapCrab_NoName_2023-3-21_11-55-19_No-00.webp)

再度デプロイが始まり、今度は無事完了しました。

# おわりに

再公開した乱数生成アプリがこちらになります。  
[乱数生成 - Yapps](https://yapps.yotiosoft.com/random/)

応答速度は申し分なく快適です。ただ、起動（スリープ状態からの復帰）に15秒ほど要しており、これに関しては Heroku のほうが早かったかと思います。
