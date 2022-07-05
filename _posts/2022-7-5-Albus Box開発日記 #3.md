---
layout: post
title: "Albus Box開発日記 #3"
tags: [Albus Box, 開発日記, C/C++, OpenSiv3D]
thumbnail: "/assets/img/thumbnails/feature-img/220705.png"
excerpt_separator: <!--more-->
---

[前回](../../05/14/Albus-Box%E9%96%8B%E7%99%BA%E6%97%A5%E8%A8%98-2.html)に引き続き、歌詞関連の機能を作成中です。ほぼ完成の状態まで来ており、残すは動作テストとバグ修正のみです。完了次第、次期バージョンをリリースしたいと思っております。

<!--more-->  

# 歌詞設定画面

既存のプレイリスト画面と同様、リスト形式で歌詞が書かれたカードが並ぶスタイルで実装しました。  
![SnapCrab_NoName_2022-7-4_23-43-55_No-00](../../../assets/img/post/2022-7-5/SnapCrab_NoName_2022-7-4_23-43-55_No-00.png)

マウスでスクロール可能で、各カードは歌詞の表示順に並ばせています。  
![SnapCrab_NoName_2022-7-4_23-44-22_No-00](../../../assets/img/post/2022-7-5/SnapCrab_NoName_2022-7-4_23-44-22_No-00.png)

カードをクリックすると歌詞の表示開始時間、表示終了時間、歌詞の内容、歌詞の削除といった設定ができます。  
![SnapCrab_NoName_2022-7-4_23-44-3_No-00](../../../assets/img/post/2022-7-5/SnapCrab_NoName_2022-7-4_23-44-3_No-00.png)  

設定が完了したら、カード外の部分をクリックすると歌詞の変更が反映されます。このとき、歌詞の表示時間を変更すると、歌詞の表示順にカードが並び替えられます。  

右下の＋ボタンを押すと新たな歌詞カードが末尾に挿入されます。デフォルトで表示開始時間に現在の再生位置が挿入され、表示終了時間はその3秒後としています。もちろん歌詞の表示時間に合わせて変更可能です。  
![SnapCrab_NoName_2022-7-4_23-52-16_No-00](../../../assets/img/post/2022-7-5/SnapCrab_NoName_2022-7-4_23-52-16_No-00.png)  

実際に歌詞を設定しているときの様子を録画しました。よかったらご覧ください。  

<video src="../../../assets/img/post/2022-7-5/albus220705.mp4" controls></video>

使用楽曲は魔王魂さんの「シャイニングスター」です。言わずとしれたフリー素材の名曲ですね。  
[https://maou.audio/14_shining_star/](https://maou.audio/14_shining_star/){:target="_blank"}  

スクロールの実装については、OpenSiv3Dの[RenderTexture](https://zenn.dev/reputeless/books/siv3d-documentation/viewer/tutorial-rendertexture){:target="_blank"}を利用しています。RenderTexture上にボタンなどを描画していき、マウススクロールに応じてそのテクスチャの設置位置をずらすことで実現しています。

## 問題点

入力ボックスから文字が溢れると、歌詞の文字列が表示しきれない問題があります。これは恐らくライブラリ側の仕様で、対処方法を検討中です。

# バグ修正

音声ファイルの読み込み中にボタンやウィンドウが操作可能のままになっている問題を修正しました。  
この問題により、音声ファイルの読み込み完了時にウィンドウがマウスに勝手に追従してしまったり、ボタンの上にカーソルがあると押していないにも関わらずそのボタンが反応してしまうなどのバグがありました。

# ToDo

- 動作テスト：主に歌詞設定画面と歌詞表示機能を動作確認
- バグ修正

# 次期バージョンについて

次期ベータ版ver.0.2.0として今月中を目処に公開予定です。ver.0.2.0に搭載予定の機能自体は完成していますので、動作確認とバグ修正が完了し次第リリースします。もうしばしお待ち下さい。

# develop版のリポジトリ

[https://github.com/YotioSoft/Albus-Box/tree/develop](https://github.com/YotioSoft/Albus-Box/tree/develop){:target="_blank"}
