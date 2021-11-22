---
layout: post
title: Firefoxにおけるbreak-inside: column-avoidの挙動について
tags: [HTML, CSS]
feature-img: "/assets/img/feature-img/211115.png"
thumbnail: "/assets/img/thumbnails/feature-img/211115.png"
excerpt_separator: <!--more-->
---

とあるサイトを作っていてハマった時の話です。  
HTMLで複数のカラムに分割すると、要素の途中で折り返されてしまいます。これを避けるためのプロパティが``break-inside: column-avoid``ですが、このプロパティ、Firefoxでは思い通りに動いてくれませんでした。

<!--more-->

# Firefoxにおける対応状況

[break-inside - CSS: カスケーディングスタイルシート | MDN](https://developer.mozilla.org/ja/docs/Web/CSS/break-inside){:target="_blank"}  

一応、Firefoxでも対応してる…っぽいけど、Chromeと異なる挙動をする。詳細は下記。

# コードと各ブラウザの動作

元のコード：  

```html
<html>
    <head>
        <style>
            #columns_wrap {
                column-count: 3;
            }
            #column1 {
                background-color: brown;
                break-inside: avoid-column;
                page-break-inside: avoid;
            }
            #column2 {
                background-color: darkorange;
                break-inside: avoid-column;
                page-break-inside: avoid;
            }
            #column3 {
                background-color: greenyellow;
                break-inside: avoid-column;
                page-break-inside: avoid;
            }
        </style>
    </head>

    <body>
        <div id="columns_wrap">
            <div id="column1">
                Lorem ipsum dolor sit amet, consectetur adipisci elit, sed eiusmod tempor incidunt ut labore et dolore magna aliqua.
            </div>

            <div id="column2">
                Lorem ipsum dolor sit amet, consectetur adipisci elit, sed eiusmod tempor incidunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrum exercitationem ullam corporis suscipit laboriosam, nisi ut aliquid ex ea commodi consequatur. Quis aute iure reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint obcaecat cupiditat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum.
            </div>

            <div id="column3">
                Lorem ipsum dolor sit amet, consectetur adipisci elit, sed eiusmod tempor incidunt ut labore et dolore magna aliqua.
            </div>
        </div>
    </body>
</html>
```

このコードでは、ChromeやEdgeでは思い通りの動作をしてくれます。  
![SnapCrab_1270015500testhtml - Google Chrome_2021-11-22_17-56-28_No-00](../../../assets/img/post/2021-11-22-Firefoxでpage-break-inside avoidが使えない/SnapCrab_1270015500testhtml - Google Chrome_2021-11-22_17-56-28_No-00.png)  
3つのカラムがあって、それが横一列に途中で折り返されることなく並んでくれるのが今回の理想形です。  

しかし、これをFirefoxで開くと…  
![SnapCrab_Mozilla Firefox_2021-11-22_17-56-37_No-00](../../../assets/img/post/2021-11-22-Firefoxでpage-break-inside avoidが使えない/SnapCrab_Mozilla Firefox_2021-11-22_17-56-37_No-00.png)  
高さを均一にするため、真ん中のカラムが途中で折り返され、末尾が右の列に食い込んでしまっています。これを防ぐために``break-inside: column-avoid``しているのに、どうもうまく機能していないようです…

# とりあえずの対処法

Firefoxからアクセスされたときだけ、各カラムを``inline-block``にしておくことで解決しました。  

```html
<html>
    <head>
        <style>
            #columns_wrap {
                column-count: 3;
            }
            #column1 {
                background-color: brown;
                break-inside: avoid-column;
                page-break-inside: avoid;
            }
            #column2 {
                background-color: darkorange;
                break-inside: avoid-column;
                page-break-inside: avoid;
            }
            #column3 {
                background-color: greenyellow;
                break-inside: avoid-column;
                page-break-inside: avoid;
            }
            /* 追記 */
            _:lang(x)::-moz-placeholder, #column1 {
                display:inline-block;
            }
            _:lang(x)::-moz-placeholder, #column2 {
                display:inline-block;
            }
            _:lang(x)::-moz-placeholder, #column3 {
                display:inline-block;
            }
        </style>
    </head>

    <body>
        <div id="columns_wrap">
            <div id="column1">
                Lorem ipsum dolor sit amet, consectetur adipisci elit, sed eiusmod tempor incidunt ut labore et dolore magna aliqua.
            </div>

            <div id="column2">
                Lorem ipsum dolor sit amet, consectetur adipisci elit, sed eiusmod tempor incidunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrum exercitationem ullam corporis suscipit laboriosam, nisi ut aliquid ex ea commodi consequatur. Quis aute iure reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint obcaecat cupiditat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum.
            </div>

            <div id="column3">
                Lorem ipsum dolor sit amet, consectetur adipisci elit, sed eiusmod tempor incidunt ut labore et dolore magna aliqua.
            </div>
        </div>
    </body>
</html>
```

![SnapCrab_Mozilla Firefox_2021-11-22_17-58-22_No-00](../../../assets/img/post/2021-11-22-Firefoxでpage-break-inside avoidが使えない/SnapCrab_Mozilla Firefox_2021-11-22_17-58-22_No-00.png)  
Firefox以外からのアクセスにも``inline-block``を適用してしまうと、表示がズレてしまうので要注意。
