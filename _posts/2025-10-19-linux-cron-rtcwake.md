---
layout: post
title: "cronとrtcwakeでLinux PCが決まった時間帯だけ起動するようにする"
tags: [Linux, Ubuntu]
excerpt_separator: <!--more-->
---

新しい勤め先ではパソコンカタカタのお仕事でして、一日中職場にある Ubuntu のデスクトップ PC を使います。また、コロナ禍以降は職場はハイブリット制になっていまして、週何回出社という目標さえ守っていれば、週何日かはリモートワークできます。

それ自体はいいんですが、そうなると毎日出社するわけではないので、家から SSH 接続で仕事する日もあります。そのたびに誰かに毎日電源スイッチを押すよう頼むのも面倒ですし、毎日つけっぱなしにしておくのも電気がもったいない…

というわけで、平日の始業時間になったら PC を自動で起動し、終業時間になったら自動でシャットダウンするよう Ubuntu デスクトップにて設定しました。

<!--more-->	

# 実行環境

- Ubuntu 24.04.3 LTS
- Linux 6.14
- BIOS/UEFI は RTC に対応

# やったこと

決まった時間に決まった処理を実行するよう定義する cron を使えば簡単に実現できます。

今回の方針をまとめると下記の通り。

- どうやって決まった時間に処理を実行させる？
  - ⇛ cron を使います
- どうやってシャットダウンさせる？
  - ⇛ shutdown コマンドを使います
- どうやって起動させる？
  - ⇛ rtcwake を使います

## 1. crontab の設定を開く

今回は root 権限が必要なので su に入るか sudo で crontab を編集します。

```bash
$ sudo crontab -e
```

すると crontab の中身が表示されます。ここに設定しておく時刻と実行するコマンドを定義していきます。

最初は何も定義されていないので、下記のような表示になっているはずです。

```bash
# Edit this file to introduce tasks to be run by cron.
#
# Each task to run has to be defined through a single line
# indicating with different fields when the task will be run
# and what command to run for the task
#
# To define the time you can provide concrete values for
# minute (m), hour (h), day of month (dom), month (mon),
# and day of week (dow) or use '*' in these fields (for 'any').
#
# Notice that tasks will be started based on the cron's system
# daemon's notion of time and timezones.
#
# Output of the crontab jobs (including errors) is sent through
# email to the user the crontab file belongs to (unless redirected).
#
# For example, you can run a backup of all your user accounts
# at 5 a.m every week with:
# 0 5 * * 1 tar -zcf /var/backups/home.tgz /home/
#
# For more information see the manual pages of crontab(5) and cron(8)
#
# m h  dom mon dow   command
```

``# m h  dom mon dow   command`` と説明書きがあるように、下記のように対応付けて実行する時刻を設定します。

| 名前    | 意味                | 設定値                  |
| ------- | ------------------- | ----------------------- |
| m       | minute（分）        | 0-59                    |
| h       | hour（時）          | 0-23                    |
| dom     | day of month（日）  | 1-31                    |
| mon     | month（月）         | 1^12                    |
| dow     | day of week（曜日） | 0-7（0は日曜、7は土曜） |
| command | コマンド            | 実行するコマンド文字列  |

## 2. rtcwake の設定を書く

rtcwake は指定した時刻に電源を投入するよう RTC (Real Time Clock) に設定するコマンドです。

これを使うと、指定した時刻に PC を起動させることができます。

```bash
rtcwake [options] [-d device] [-m standby_mode] {-s seconds|-t time_t}
```

- [Ubuntu Manpage: rtcwake - enter a system sleep state until specified wakeup time](https://manpages.ubuntu.com/manpages/jammy/en/man8/rtcwake.8.html){:target="_blank"}

また、rtcwake は ``-m`` オプションでシステムをスリープ、サスペンド、ハイバネート、電源オフ、その他の動作をさせることが可能です。オプションの内容は下記の通り。

| オプション | ACPI | 結果                                                     |
| ---------- | ---- | -------------------------------------------------------- |
| standby    | S1   | 省電力状態                                               |
| freeze     |      | サスペンド。電源オフにはならない                         |
| mem        | S3   | 通常のスリープ                                           |
| disk       | S4   | ハイバネート。メモリ内容をディスクに書き出して休止状態に |
| off        | S5   | シャットダウン                                           |
| no         |      | RTC の設定のみ                                           |
| on         |      | RTC を監視してアラート発生まで待機                       |
| disable    |      | 設定済みのアラームを無効化                               |
| show       |      | 現在のアラーム設定を表示                                 |

今回はシャットダウンさせたいので ``-m off`` を使います。



また、``-t`` で起動させたい時刻を指定します。こちらは UNIX TIME で指定しなければならないため要注意です。

今回は ``date`` コマンドを使います。例えば、「明日の8:00」で UNIX TIME を取得したうえで ``rtcwake -t`` に渡すようにすると下記のようになります。

```bash
/usr/sbin/rtcwake -m off -t $(/usr/bin/date -d 'tomorrow 08:00' +\%s)
```

## 3. 日時を設定する

「平日の決まった時間にシャットダウンし、次の日の決まった時間に起動」なので、dow (day of week) に対して ``1-5`` を指定し、月～金曜日の終業時間に、rtcwake で「次の日の朝」に起動するよう設定すればよいでしょう。

終業時間は仮に 21:00 とします（職場の名誉のために補足しますが、21:00 まで残業することはほぼないです）。

```bash
# m h  dom mon dow   command
00 21 * * 1-5 /usr/sbin/rtcwake -m off -t $(/usr/bin/date -d 'tomorrow 08:00' +\%s)
```

…ちょっと待ったー！これでは土日が起動しっぱなしになってしまいます。

正しくはこうです。

```bash
# m h  dom mon dow   command
00 21 * * 1-4 /usr/sbin/rtcwake -m off -t $(/usr/bin/date -d 'tomorrow 08:00' +\%s)
00 21 * * 5 /usr/sbin/rtcwake -m off -t $(/usr/bin/date -d 'next mon 08:00' +\%s)
```

金曜日（dow=5）だけ「次の月曜日の朝 8:00」に起動する rtcwake を実行するよう設定しています。

もちろん、時刻は自分の環境に合わせて適宜変更してください。

## 4. crontab を保存＆確認する

エディタを閉じて、書き込まれたことを確認しましょう。

```bash
$ sudo crontab -l
# Edit this file to introduce tasks to be run by cron.
#
# Each task to run has to be defined through a single line
# indicating with different fields when the task will be run
# and what command to run for the task
#
# To define the time you can provide concrete values for
# minute (m), hour (h), day of month (dom), month (mon),
# and day of week (dow) or use '*' in these fields (for 'any').
#
# Notice that tasks will be started based on the cron's system
# daemon's notion of time and timezones.
#
# Output of the crontab jobs (including errors) is sent through
# email to the user the crontab file belongs to (unless redirected).
#
# For example, you can run a backup of all your user accounts
# at 5 a.m every week with:
# 0 5 * * 1 tar -zcf /var/backups/home.tgz /home/
#
# For more information see the manual pages of crontab(5) and cron(8)
#
# m h  dom mon dow   command
00 21 * * 1-4 /usr/sbin/rtcwake -m off -t $(/usr/bin/date -d 'tomorrow 08:00' +\%s)
00 21 * * 5 /usr/sbin/rtcwake -m off -t $(/usr/bin/date -d 'next mon 08:00' +\%s)
```

こうなっていれば OK です。

最後に、cron を再起動しておきます。ディストリビューションやバージョンによっては不要かもしれないですが念の為。

```bash
$ sudo systemctl restart cron
```

# おわりに

これにて月曜～金曜、8:00～21:00 だけ起動するシステムが完成しました。あなたも cron でレッツ節電！
