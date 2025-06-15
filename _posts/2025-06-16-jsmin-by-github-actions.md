---
layout: post
title: "JavaScriptのminifyをGitHub Actionsにやらせる"
tags: [Yapps, GitHub]
excerpt_separator: <!--more-->
---

普段より Yapps という Web アプリサイトを運営しており、主に JavaScript でアプリを実装しているのですが、これまでは自分で書いた js ファイルはミニファイをしておらず、改行・コメント等が残されたままでした。

ご存知の通り、JavaScript はブラウザで実行させるためにバイナリにコンパイルして配布するわけではなく、ソースファイルを配布するのが一般的です。ブラウザでファイルを読み込み、ブラウザ実装の js エンジンで構文解析、バイトコード変換、インタプリタ実行、JIT compile、…といった手順が踏まれます。

js ファイルは単なるテキストファイルですので、単なる改行であろうと単なるコメントであろうとブラウザはそれも含めて読み込みます。が、実行の上では無駄なデータでしかなく、それを回避することは有意義と言えます。

この無駄な改行やコメントを削除する工程を「minify（ミニファイ）」「minimize」あるいは「圧縮」などといいます。世の中のイケてるサービスはだいたいミニファイをきちんと実施している印象です。というわけで、世の中で一般的に行われているミニファイを適用しようと意気込んだのですが、毎回自分でやるのは面倒なので GitHub Actions にやらせました。

<!--more-->	

# つかうもの

| ツール         | 概説                                |
| -------------- | ----------------------------------- |
| Python         | jsmin を実行するのに使う            |
| jsmin          | JavaSctipy をミニファイするのに使う |
| GitHub Actions | デプロイ時に jsmin を動かしてくれる |

# 作った workflow

全体像は下記の通り。

```yaml
name: Minify JavaScript Files

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  minify-js:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Safe directory
        run: git config --global --add safe.directory $(pwd)

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.x'

      - name: Install jsmin
        run: pip install jsmin

      - name: Minify all .js files in-place
        run: |
          find . -type f -name "*.js" ! -name "*.min.js" ! -path "./.git/*" | while read -r file; do
            minified_file="${file%.js}.min.js"
            echo "Minifying $file -> $minified_file"
            python -m jsmin "$file" > "$minified_file"
          done

      - name: Commit and push changes
        run: |
          git config --local user.name "github-actions"
          git config --local user.email ""
          git add .
          git commit -am "Minified JavaScript files" || echo "No changes to commit"
          git push origin main || echo "No changes to commit"
```

一つずつ見ていきます。

```yaml
- name: Safe dirctory
        run: git config --global --add safe.directory $(pwd)
```

ここでは実行ユーザが所有権を持たない .git ディレクトリを読み込めるように設定しています。GitHub Actions に git commit させるうえで必須のようです。

```yaml
- name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.x'
```

ここでは Python 3.x を設定。

```yaml
- name: Install jsmin
        run: pip install jsmin
```

pip で jsmin を実行環境にインストール。

```yaml
- name: Minify all .js files in-place
        run: |
          find . -type f -name "*.js" ! -name "*.min.js" ! -path "./.git/*" | while read -r file; do
            minified_file="${file%.js}.min.js"
            echo "Minifying $file -> $minified_file"
            python -m jsmin "$file" > "$minified_file"
          done
```

ここがキモです。

まず、一番上のディレクトリを起点に ``*.js`` を探します。重要なのは、``*.min.js`` をここに含めないことです。後ほど ``git commit`` してコミットに minify された js ファイルが含まれますので、次に GitHub Actions が ``*.min.js`` を作成するときに、既に minify されたファイルに対して minify を適用してしまいます（つまり、``*.min.min.js`` が作られてしまいます）。これを回避します。

次にファイル名を ``minified_file`` に設定します。ここでは ``hoge.js`` が ``hoge.min.js`` になるようにしています。

最後に、各ファイルに対して ``python -m jsmin`` によって jsmin を実行し、minify されたファイルを作成して ``minified_file`` として保存します。

```yaml
- name: Commit and push changes
        run: |
          git config --local user.name "github-actions"
          git config --local user.email ""
          git add .
          git commit -am "Minified JavaScript files" || echo "No changes to commit"
          git push origin main || echo "No changes to commit"
```

最後にコミットして push します。前半ではユーザ名を設定しています。``git commit``、``git push`` では、minify の結果、差分がない場合は commit できずエラーになります。これは正常な動作とみなさないと workflow が失敗扱いになります。これを阻止するために、エラー時は echo で「No changes to commit」と表示するだけで済ませます。

# 結果

![image-20250616015318347](../../../assets/img/post/2025-06-16-jsmin-by-github-actions/image-20250616015318347.webp)

![image-20250616015413646](../../../assets/img/post/2025-06-16-jsmin-by-github-actions/image-20250616015413646.webp)

![image-20250616015533611](../../../assets/img/post/2025-06-16-jsmin-by-github-actions/image-20250616015533611.webp)

無事、コミットの push 時に workflow が走り、各 js ファイルと同じ場所に min.js が作成されている様子を確認できました。

ただ、GitHub Pages によってコミットごとに pages build and deployment が動く仕組みになっているので、自分のコミット→ pages build and deployment → minify による自動コミット → pages build and deployment と、無駄に2回 pages build が動いちゃうのはなんとかしたいですね。

## パフォーマンス

で、PageSpeed Insights でパフォーマンスの変化を計測してみたんですが、そこまで大して変わらず…。

（minify 前）

![image-20250616021759559](../../../assets/img/post/2025-06-16-jsmin-by-github-actions/image-20250616021759559.webp)

（minify 後）

![image-20250616021817723](../../../assets/img/post/2025-06-16-jsmin-by-github-actions/image-20250616021817723.webp)

おそらくヘッダファイルを別 HTML で書いちゃってるのがよくなくて、HTML の読み込みに足を引っ張られている印象です。まあこれは Yapps 固有の改善すべき点ですね。

![image-20250616021842028](../../../assets/img/post/2025-06-16-jsmin-by-github-actions/image-20250616021842028.webp)

肝心の js minify の方ですが、そこまで大きな js ファイルを置いていないのが原因じゃないかと推測しています。せいぜい数十～100 行程度のスクリプトファイルでは効果がほぼ得られないのかもしれません。とはいえやらないよりはマシです。GitHub Actions ならタダで勝手に処理してくれますし、やっておいて損はないでしょう。
