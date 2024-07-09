---
layout: post
title: "GDBで関数の実行時間を計測する"
tags: [Linux, Ubuntu, xv6, GDB, C/C++]
excerpt_separator: <!--more-->
---

C 言語などで利用できる GNU デバッガツールである GDB では、関数呼び出しの履歴を backtrace で確認したり、ブレークポイントを置いて任意の位置で実行を止めたり、その時その時の変数やレジスタの状態などを確認できます。非常に便利なツールなのですが、GDB には関数単位やブレークポイント単位でのプログラムの実行時間の計測機能は調べた限り存在しません。

そこで今回は、GDB でむりやりプログラムの実行時間を計測してみた、という趣旨の話になります。

<!--more-->

# 背景

通常であれば、プログラムの実行時間を計測したい場合は``clock()``なり``time()``などを使えば問題はありません。秒単位だけでなく、マイクロ秒単位までこれらで計測できます。

しかしながら非常に稀有なケースとして、実行環境によってはこれらの解決策が利用できない場合があります。Linux 環境や Windows 環境であれば先程挙げた ``time.h`` の時刻取得系の関数を使えば OK ですが、例えば xv6 の場合は最小で 1 ticks ≒ 0.01 秒精度でしか実行時間を計測できません。

いやいや、誰が xv6 で実行時間の計測なんかするねん、と思われるでしょう。しかしながら今、自身の研究で xv6 でマイクロ秒精度（つまり 0.000001 秒精度）での実行時間の計測データが必要なのです。

xv6 は仮想マシンとして QEMU（あるいは Bochs）での実行を想定しており、コンパイラとして GCC を、デバッガとして GDB をサポートしています。デバッグするときは QEMU で xv6 を実行しながら GDB でデバッグを進めていく、といった形になります。逆に言うとそれら以外は（おそらく）使えません。

# 実現方針

## GDB でブレークポイント到達時にコマンドを実行させる

GDB ではブレークポイントを任意のソースファイルの任意の位置に指定できます。これはもちろん、関数の開始地点と終了地点にも置くことが可能です。では、関数開始・終了地点のそれぞれのブレークポイントに到達したときに現在時刻を取得して、その差分を取れば、関数の実行時間が GDB で計測できるのではないか？と考えました。

GDB には、特定のブレークポイントに達したときに任意のコマンドを実行させる機能があります。

- [GDB マニュアル - Break Commands](https://flex.phys.tohoku.ac.jp/texi/GDB-j/GDB-j_22.html){:target="_blank"}

ここでは主に GDB コマンド（例えば ``print`` で変数値を表示する、``set`` で変数に数値を代入するなど）の利用が想定されていますが、GDB にはシェルから外部コマンドを実行させる ``shell`` コマンドがあります。

ここで言うシェルや外部コマンドというのは xv6 のシェルやコマンドのことではなく、QEMU を実行しているホスト環境（= Linux や Windows など）のシェルやコマンドのことです。Linux には現在時刻を取得するコマンドがありますので、これらを組み合わせれば実現できそうです。

注：最終的には ``shell`` コマンドは使いません（後述）

## Linux で（マイクロ秒単位で）現在時刻を取得する

今回は ``date`` コマンドを選びました。システム上の現在時刻を取得するコマンドです。

- [man date (1): システムの日付や時刻の表示、設定を行う](https://ja.manpages.org/date){:target="_blank"}
- [date コマンドで日時のミリ秒単位まで表示する #ShellScript - Qiita](https://qiita.com/niwasawa/items/9502e97b6c4d28d24042){:target="_blank"}

マイクロ秒単位で時間計測をしたいので、1970年1月1日を起点とした経過時間をマイクロ秒単位で表示させてみます。

```bash
$ date +%s.%6N
1720518841.110454
```

これを GDB の ``shell`` コマンドで実行しようとすると下記のようになります。

```bash
(GDB) shell date +%s.%6N
```

## 問題点：GDB では ``shell`` の実行結果を簡易変数に代入できない

GDB では ``shell`` コマンドで外部コマンド ``date`` を実行すれば、GDB でブレークポイントに達したときに現在時刻を表示させることは上記で示しました。ただ、ここで一つ問題があります。``shell`` コマンドはあくまでもコマンドを実行して表示するのみで、その結果を簡易変数に代入できないのです。

そこで、``shell`` コマンドを使うのは取りやめ、代わりに Python を利用します。GDB は Python の呼び出しをサポートしており、``python`` コマンドで Python スクリプトを1行ずつ実行させることができるのです。Python は ``os`` モジュールで外部コマンドの実行ができるので、``date`` コマンドを Python 側で実行し、その結果を Python 側で実数に変換し、Python 側で差分の計算と表示も行わせてみます。（``GDB.execution()`` を使って GDB の簡易変数に Python の演算結果を代入させることも可能ですが、今回は割愛します）

Python スクリプトを示すとこんな感じになります。

```python
import os

start_time = float(os.popen("date +%s.%6N").read())

# ここで次のブレークポイントまでの処理...

end_time = float(os.popen("date +%s.%6N").read())
print("sys_open run time: " + str(end_time - start_time))
```



# 実践

ここからは実際に動かしていきます。

まず、GDB 起動のたびにブレークポイントや commands を定義するのは面倒です。そこで、コマンドファイルにブレークポイントの定義や実行するコマンドを書いていきます。

- [GDB マニュアル - Command Files](https://flex.phys.tohoku.ac.jp/texi/GDB-j/GDB-j_54.html){:target="_blank"}

今回は例として、xv6 の open() システムコールのエントリポイント関数 ``sys_open()`` の実行時間を計測してみます。

![スクリーンショット 2024-07-09 22.11.58](../../../assets/img/post/2024-07-09-GDB-timespan/スクリーンショット 2024-07-09 22.11.58.webp)

今回置くブレークポイントは ``sys_open()`` の先頭と最後です。ブレークポイントの位置（``break なんとか`` の箇所）は、適宜計測したいプログラムや関数に合わせて変更しておいてください。

作成したコマンドファイルを下記に示します。

```
# Python 側で os モジュールをインポート
python import os

# sys_open() の開始地点
# ブレークポイントを設定し、タイムスタンプを取得する
break sys_open
commands
    silent
    python start_time = float(os.popen("date +%s.%6N").read())
    continue
end

# sys_open() の終了地点：sysfile.cの318行目
# ブレークポイントを設定し、タイムスタンプを取得する
break sysfile.c:318
commands
    silent
    python end_time = float(os.popen("date +%s.%6N").read())
    python print("sys_open run time: " + str(end_time - start_time))
    continue
end
```

内容はコメントに書いたとおりです。補足すると、``silent`` はブレークポイント到達時の情報（何行目のどのコードか、など）を非表示にするコマンド、``continue`` は一連のコマンド処理が終わった時にすぐに実行を再開するためのコマンドです。



これを適当な場所に置き（ファイル名は ``gdb_timestamp.txt`` としました）、GDB の ``source`` コマンドでコマンドファイルを読み込ませます。

```bash
$ gdb out/kernel.elf
(gdb) target remote ...    # QEMU に接続
(gdb) source gdb_timestamp.txt     # ← これ
```

## いざ実行！

実行してみた結果がこちらです。

![スクリーンショット 2024-07-09 22.18.08](../../../assets/img/post/2024-07-09-gdb-timespan/スクリーンショット 2024-07-09 22.18.08.webp)

マイクロ秒精度よりも細かい数値なので有効桁数の調整は必要ですが、ひとまず ``sys_open()`` 関数実行中の実行時間を GDB で計測できていることが確認できました。

以上です。お疲れ様でした。

# 参考文献

- [GDB マニュアル - Break Commands](https://flex.phys.tohoku.ac.jp/texi/GDB-j/GDB-j_22.html){:target="_blank"}
- [man date (1): システムの日付や時刻の表示、設定を行う](https://ja.manpages.org/date){:target="_blank"}
- [date コマンドで日時のミリ秒単位まで表示する #ShellScript - Qiita](https://qiita.com/niwasawa/items/9502e97b6c4d28d24042){:target="_blank"}
- [Redirecting/storing output of shell into GDB variable? - Stack Overflow](https://stackoverflow.com/questions/6885923/redirecting-storing-output-of-shell-into-gdb-variable){:target="_blank"}
