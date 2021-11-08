---
layout: post
title: ChromebookでもTyporaが使いたい！
tags: [Chrome OS, Typora]
excerpt_separator: <!--more-->
---

さて、MacBook Proの代わりにしばらくChromebookを使うことになったわけですが、やはりChromebookでもブログ更新を行いたい。このブログはJekyllで運用しているので、普段はTyporaを使ってMarkdownで記事を書いているわけですが、ここで疑問。  
「ChromebookでTyporaって使えるの？」  

結論から言うと使えます。ただし導入が結構面倒です。  
特に日本語入力環境の導入が面倒。でも、この手順さえ一度やっておけば、今後ChromebookでいろいろなLinuxアプリで日本語入力ができるようになります。

<!--more-->

# 気をつけるべき点

Chromebookで使えることは使えるんですが、Linux仮想環境上で使うので多少制約があります。  

- ファイルをドラッグ&ドロップでMarkdownに追加できない
- Linuxと共有されたディレクトリからしかファイルを追加・読み込み・保存できない
- **Chrome OSのIMEは使えない。日本語入力したい場合は別途IMEのインストールが必要（ここが面倒）**
- IMEの切り替え方法が若干異なる（「かな↔英数」キーのみ切り替え可、「英数」「かな」キーでは切り替えできない）

# 手順

## Linux仮想環境の導入

Chrome OSでLinuxターミナルが使えるようにします。既に導入している場合は飛ばしてください。  

まずは設定を開き、「詳細設定」→「デベロッパー」から「Linux開発環境」をオンにします。  
![](../../../assets/img/post/2021-11-10-ChromebookでもTyporaが使いたい！/Screenshot 2021-11-07 22.07.26.png)  
設定を進めます。  
![](../../../assets/img/post/2021-11-10-ChromebookでもTyporaが使いたい！/Screenshot 2021-11-07 22.07.48.png)  
ユーザー名と容量の設定を済ませると、あとは自動でインストールしてくれます。1分程度で終わります。  
![](../../../assets/img/post/2021-11-10-ChromebookでもTyporaが使いたい！/Screenshot 2021-11-07 22.07.54.png)  
インストールが終わるとターミナルが開けます。

## Typoraの導入

続いてTyporaを導入します。まずはTyporaの[公式サイト](https://typora.io/#linux){:target="_blank"}に書かれたコマンドを実行してみます。  

```bash
# or run:
# sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys BA300B7755AFCFAE
wget -qO - https://typora.io/linux/public-key.asc | sudo apt-key add -
# add Typora's repository
sudo add-apt-repository 'deb https://typora.io/linux ./'
sudo apt-get update
# install typora
sudo apt-get install typora
```


まずは1つ目。  

```bash
$ wget -qO - https://typora.io/linux/public-key.asc | sudo apt-key add -
```

```
OK
```

完了すると「OK」と表示されます。ここは特に問題なし。  

次が問題。aptにTyporaのリポジトリを導入しますが、そのまま実行すると``add-apt-repository``なんて知らねーよと言われます。  

```bash
$ sudo add-apt-repository 'deb https://typora.io/linux ./'
```

```
sudo: add-apt-repository: command not found
```

なので先に``add-apt-repository``を導入します。  

```bash
$ sudo apt-get install software-properties-common
```

そしたらもう一度``add-apt-repository``を実行。  

```bash
$ sudo add-apt-repository 'deb https://typora.io/linux ./'
```

これでOK。  

次にapt-getをupdateしてTyporaをインストールします。ここは特に手を加える必要はありませんので一気に示します。  

```bash
$ sudo apt-get update
$ sudo apt-get install typora
```


これでTyporaが起動するはず！とTyporaを起動してみると…  

```bash
$ typora
```

```
typora: error while loading shared libraries: libnss3.so: cannot open shared object file: No such file or directory
```

libnss3が存在しないと言われました。通常、Ubuntu等では最初から導入されているのですが、どうやらCrostini環境には存在しない模様。というわけで別途導入します。  

```bash
$ sudo apt install libnss3
```



さあ、今度こそTyporaが起動するはずだぞ！と早速Typoraを開いてみます。  
![](../../../assets/img/post/2021-11-10-ChromebookでもTyporaが使いたい！/Screenshot 2021-11-07 22.17.23.png)  
起動した！やった！ちなみに初期状態では表示言語は英語ですが、設定から日本語に変更しておきました。

## 日本語入力に対応させる

しかし、喜んでいるのもつかの間。なんと日本語が入力できないのです。  
![](../../../assets/img/post/2021-11-10-ChromebookでもTyporaが使いたい！/Screenshot 2021-11-07 22.17.34.png)   
「かな↔英数」キーを押すと、「Linuxアプリは日本語入力に対応していない」云々のメッセージが表示されます。そう、Chrome OS、現時点ではTyporaのようなLinuxアプリでは日本語入力に対応していないのです。こりゃー困った。  

しかし、Linux環境上に日本語IMEをインストールすれば解決することが判明。以下にその手順を示します。  


### インストールするもの

- Mozc

Ubuntuではおなじみの日本語IME。Google日本語入力のオープンソース版です。Ubuntuなら初期からインストールされていますが、Crostiniの仮想環境には導入されていないので今回はコマンドからインストールしてみます。

### localeの設定

日本語を有効化します。まずはエディタをインストール（お好みのエディタでOK）  

```bash
$ sudo apt install emacs
```

つぎに``/etc/locale.gen``を開き編集します。  

```bash
$ sudo emacs /etc/locale.gen
```

ここでは``# ja_JP.UTF-8 UTF-8``と記述されている部分を探し、コメントアウト（#）を外します。  
![](../../../assets/img/post/2021-11-10-ChromebookでもTyporaが使いたい！/Screenshot 2021-11-07 22.24.15.png)  
↓  
![](../../../assets/img/post/2021-11-10-ChromebookでもTyporaが使いたい！/Screenshot 2021-11-07 22.24.41.png)  
``ja_JP.UTF-8 UTF-8``と書き換えたら保存してエディタを終了します。  

保存が完了したら次にlocaleをgenerateします。  

```bash
$ sudo locale-gen
```

```
Generating locales (this might take a while)...
  ja_JP.UTF-8... done
  en_US.UTF-8... done
Generation complete.
```

日本語が有効化された様子が見られます。さらにロケール設定に反映。  

```bash
$ sudo localectl set-locale LANG=ja_JP.UTF-8 LANGUAGE="ja_JP:ja"
```

完了したら一旦Linuxをシャットダウンします。ターミナルのアイコンを右クリックし「Linuxをシャットダウン」を選択。  
![](../../../assets/img/post/2021-11-10-ChromebookでもTyporaが使いたい！/Screenshot 2021-11-07 22.28.00.png)  

そしてもう一度ターミナルを開き、localeの設定が反映されているか確認します。  

```bash
$ echo $LANG
```

```
ja_JP.UTF-8
```

反映されていることが確認できました。

### mozcのインストール

いよいよ日本語IMEをインストールしていきます。  

```bash
$ sudo apt install fcitx-mozc
$ im-config -n fcitx
```

これでOK。必要な環境はすべてインストールされます。インストールに数分間要します。

### fcitxの設定

次にfcitxの設定を行います。``cros-garcon-override.conf``を開き、  

```bash
$ sudo emacs /etc/systemd/user/cros-garcon.service.d/cros-garcon-override.conf 
```

以下を追記します。  

```
Environment="GTK_IM_MODULE=fcitx"
Environment="QT_IM_MODULE=fcitx"
Environment="XMODIFIERS=@im=fcitx"
```

![](../../../assets/img/post/2021-11-10-ChromebookでもTyporaが使いたい！/Screenshot 2021-11-07 22.31.08.png)  
追記したら保存してエディタを終了。  

さらにfcitxの自動起動を設定します。 

```bash
$ sudo emacs ~/.sommelierrc
```

``.sommelierrc``に以下を追記して保存。  

```
/usr/bin/fcitx-autostart
```

![](../../../assets/img/post/2021-11-10-ChromebookでもTyporaが使いたい！/Screenshot 2021-11-07 22.32.49-16363770710771.png)  

次にfcitxを起動します。  

```bash
$ fcitx-autostart
```

完了したら先程の手順と同じように一旦Linuxをシャットダウンし、またターミナルを起動します。



## IMEの優先順位の変更

初期状態では英語IMEが優先順位が上位にあるため、日本語IMEを上位に設定します。  

```bash
$ fcitx-configtool
```

上記のコマンドを実行すると、以下のようなウィンドウが出てきます。  
![](../../../assets/img/post/2021-11-10-ChromebookでもTyporaが使いたい！/Screenshot 2021-11-07 22.36.17.png)  
ここで、Mozcを選択して、ウィンドウ下部の^ボタンを選択。これでMozcが一番上に表示されたらウィンドウを閉じます。



## Typoraで動作確認

これでできたはずです。早速Typoraを起動してみます。  

```bash
$ typora
```

すると、日本語が…入力できた！  
![](../../../assets/img/post/2021-11-10-ChromebookでもTyporaが使いたい！/Screenshot 2021-11-07 22.37.41.png)  
これでChromebookでもTyporaで日本語が入力できるようになりました。予測変換も利用できます。「かな↔英数」キーで英語IMEと日本語IME(Mozc)との切り替えができます。ちなみにこの記事もChromebookでTyporaを使って書いていますが、動作はとても軽快です。



# 注意点と対処法

冒頭でも示したとおり、

- ファイルをドラッグ&ドロップでMarkdownに追加できない
- Linuxと共有されたディレクトリからしかファイルを追加・読み込み・保存できない
- IMEの切り替え方法が若干異なる（「かな↔英数」キーのみ切り替え可、「英数」「かな」キーでは切り替えできない）

といった制約があります。以下に、それぞれの対処法を示します。

## ファイルをドラッグ&ドロップでMarkdownに追加できない

ファイルのドラッグアンドドロップが使えないので、ファイルをTypora上で選択して追加する必要があります。  
挿入したい場所で右クリック→「挿入」→「画像」(あるいは``Ctrl+Shift+I``)で画像タグを挿入し、フォルダアイコンを押すと画像が選択できます。ちなみにChromebookはShiftキーにShiftと記載されていない機種が多いですが、（少なくとも日本語配列では）ShiftキーはCtrlキーの上にあります。

## Linuxと共有されたディレクトリからしかファイルを追加・読み込み・保存できない

Markdownファイルや画像など、Typoraで利用するあらゆるファイルはLinuxと共有されたディレクトリにあるものしか読み書きできません。一番無難なのは、Markdownファイルも画像ファイルも「Linuxファイル」ディレクトリの配下に保存しておくことだと思います。LinuxファイルディレクトリはCrostini上ではホームディレクトリとして扱われます。  
![image-20211108223241900](../../../assets/img/post/2021-11-10-ChromebookでもTyporaが使いたい！/image-20211108223241900.png)    

![image-20211108223526018](../../../assets/img/post/2021-11-10-ChromebookでもTyporaが使いたい！/image-20211108223526018.png)  
Linuxディレクトリ以外（例えば「ダウンロード」ディレクトリ)でも、「ファイル」アプリでディレクトリを右クリック→「Linuxと共有」で利用可能になります。

## IMEの切り替え方法が若干異なる

スペースキーの両隣にある「かな」「英数」キーではIMEの切り替えができません。日本語配列ではEscキーの下にある「かな↔英数」キーで切り替えが可能です。



# おわりに

今回はChrome OSへのTyporaの導入と注意点について書きました。Crostiniは初期状態で導入されている環境が少なかったりと導入が面倒な点はありますが、導入さえ一度できてしまえば多種多様なLinuxアプリがChrome OSでも利用可能になります。  
今回は日本語入力をCrostini環境で行うためにMozcを導入し様々な設定を行いましたが、これは他のLinuxアプリでも利用可能です。公式には日本語入力には非対応なものの、Mozcを導入すればLinuxアプリでも快適に日本語入力が利用できます。LinuxアプリをChrome OSで利用したい方はぜひ試してみてください。
