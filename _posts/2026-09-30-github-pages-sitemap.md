---
layout: post
title: "Jekyllのsitemap作成もGitHub Actionsにやらせよう"
tags: [Jekyll, GitHub]
excerpt_separator: <!--more-->
---

ブログのアクセス数を向上させたいなら、sitemap の用意は欠かせません。しかし、更新する度に sitemap を用意するのは面倒です。面倒だから自動化したいと思います。

このブログは GitHub Pages で運営していて、フレームワークとして Jekyll を利用しています。GitHub Pages は GitHub Actions でデプロイ作業を通してページやファイルの生成を自動実行させることが可能です。というわけで、sitemap.xml の作成を GitHub Actions で自動化したのでそのメモ。

<!--more-->

# 前提条件

- GitHub Pages で運営している
- Jekyll を利用している

GitHub Pages でなくても、Jekyll を利用しているのであれば同様の手段でデプロイと同時に sitemap 生成が可能かと思います。

# TL;DR

`jekyll-sitemap` という Jekyll プラグインを使います。

{% github jekyll/jekyll-sitemap % }



適用方法はいたって簡単。Jekyll 公式のプラグインなので GitHub Pages にぬるっと適用できます。

まずは `_config.yml` にてプラグインを追加します。

```yaml
 plugins:
+  - jekyll-sitemap
```



また、GitHub Actions から `bundle exec jekyll build` して sitemap を生成できるように、 `Gemfile` にも追加しておきます。

```
+gem 'jekyll-sitemap'
```



最後に GitHub Actions 用の job を定義します。こちらは一例です。

```yaml
name: Update sitemap

on:
  push:
    branches: [master]
    paths-ignore:
      - sitemap.xml
  workflow_dispatch:

permissions:
  contents: write

concurrency:
  group: update-sitemap-${{ github.ref }}
  cancel-in-progress: false

jobs:
  update-sitemap:
    runs-on: ubuntu-latest

    steps:
      - name: Check out repository
        uses: actions/checkout@v4

      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.3'
          bundler-cache: true

      - name: Build the Jekyll site
        run: bundle exec jekyll build

      - name: Copy generated sitemap
        run: cp _site/sitemap.xml sitemap.xml

      - name: Commit sitemap when it changed
        run: |
          if git diff --quiet -- sitemap.xml; then
            echo "sitemap.xml is already up to date."
            exit 0
          fi

          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add sitemap.xml
          git commit -m "chore: update sitemap"
          git push
```

`bundle exec jekyll build` で sitemap を作成します。デフォルトでは `_site/sitemap.xml` に生成されるのですが、Google Search Console に設定しているパスの関係上、トップディレクトリにコピーしています。

`update-sitemap` の `Commit sitemap when it changed` でコミットしているような形です。

# 成果物

生成された sitemap.xml を見てみます。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.sitemaps.org/schemas/sitemap/0.9 http://www.sitemaps.org/schemas/sitemap/0.9/sitemap.xsd" xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
<url>
<loc>https://blog.yotio.jp/2021/08/11/Hello-World!.html</loc>
<lastmod>2021-08-11T00:00:00+09:00</lastmod>
</url>
<url>
<loc>https://blog.yotio.jp/2021/08/11/%E6%96%B0%E3%81%97%E3%81%84%E3%83%9B%E3%83%BC%E3%83%A0%E3%83%9A%E3%83%BC%E3%82%B8%E3%81%AB%E3%81%A4%E3%81%84%E3%81%A6.html</loc>
<lastmod>2021-08-11T00:00:00+09:00</lastmod>
</url>
<url>
<loc>https://blog.yotio.jp/2021/08/12/%E3%81%93%E3%82%8C%E3%81%BE%E3%81%A7%E3%81%AE%E3%83%9B%E3%83%BC%E3%83%A0%E3%83%9A%E3%83%BC%E3%82%B8%E3%81%AE%E3%82%A2%E3%83%BC%E3%82%AB%E3%82%A4%E3%83%96%E3%81%AB%E3%81%A4%E3%81%84%E3%81%A6.html</loc>
<lastmod>2021-08-12T00:00:00+09:00</lastmod>
</url>
<url>
<loc>https://blog.yotio.jp/2021/08/17/%E6%96%B0HP%E3%81%AE%E3%82%B9%E3%83%9E%E3%83%9B%E7%89%88%E3%82%92%E5%85%AC%E9%96%8B.html</loc>
<lastmod>2021-08-17T00:00:00+09:00</lastmod>
</url>
<url>
<loc>https://blog.yotio.jp/2021/08/20/Rust%E3%82%92%E5%A7%8B%E3%82%81%E3%81%A6%E3%81%BF%E3%82%8B-Mac%E7%B7%A8.html</loc>
<lastmod>2021-08-20T00:00:00+09:00</lastmod>
</url>
...
```

デフォルトでファイルパスから lastmod を抽出する仕組みになっています。うん、いい感じですね。

