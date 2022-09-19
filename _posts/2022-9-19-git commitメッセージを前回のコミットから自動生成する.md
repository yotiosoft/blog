---
layout: post
title: "git commitメッセージを前回のコミットから自動生成する"
tags: [git, GitHub, Mac, Linux]
excerpt_separator: <!--more-->
---

gitで前回のコミットから数行程度の修正しかない小規模なコミットするときにいちいちコミットメッセージを書くのが面倒なので、直前のコミットメッセージを取得し、それに「小規模な変更」を示す「small change: 」という定型文を加えてコミットメッセージに代えることに。そこで、``git commit``に

```zsh
git log -p -1 | grep "    " | cut -c 5- | sed "s/^/small change: /"
```

の結果をパイプ入力で渡して、コミットメッセージを自動生成するコマンドを作成しました。

<!--more-->  

# 結論

```zsh
git log -p -1 | grep "    " | cut -c 5- | sed "s/^/small change: /"
```

でいけるます。grepに指定している文字列の空白文字は半角4文字です。  


以下、解説という名の蛇足。

## git log -1

1個前のコミット情報を抜き出す。

```zsh
% git log -1
Author: YotioSoft <ytani0323@gmail.com>
Date:   Mon Aug 29 22:07:44 2022 +0900

    update README
```

## grep "    "

前者で取得した1個前のコミット情報からコミットメッセージの文字列を抜き出す。``git log -1``ではコミットメッセージの前に空白文字4文字が挿入されるので、それがある行を切り出し。

```zsh
% git log -p -1 | grep "    "
    update README
```

## cut -c 5-

5文字目以降を切り出す。これで文字列先頭部分の空白文字が消去される。

```zsh
% git log -p -1 | grep "    " | cut -c 5-
update README
```

## sed "s/^/small change: /"

「small change: 」という文字列を行頭に挿入する。文字列に関してはお好みに。

```zsh
% git log -p -1 | grep "    " | cut -c 5- | sed "s/^/small change: /"
small commit: update README
```

## git commit --file=-

最後に生成した文字列をコミットメッセージにしてコミット。``--file=-``で標準入力から文字列を取得します。（あ、もちろんその前に``git add``しとくことを忘れずに）

```zsh
% git log -p -1 | grep "    " | cut -c 5- | sed "s/^/small change: /" | git commit --file=-
[develop c8daaf3] small change: update README
 1 file changed, 2 insertions(+), 2 deletions(-)
```

# おわりに

コマンドが長ったらしいので、これをシェルスクリプトなりに保存してパスを通しておくと便利かと思います。

