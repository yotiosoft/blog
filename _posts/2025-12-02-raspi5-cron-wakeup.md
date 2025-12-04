---
layout: post
title: "Raspberry Pi5でも決まった時間に自動起動できるようにする"
tags: [Linux, Ubuntu, Raspberry Pi]
excerpt_separator: <!--more-->
---

以前、cron と rtcwake を使って通常の Linux PC で決まった時間に自動起動させる方法についてまとめました。

- [cronとrtcwakeでLinux PCが決まった時間帯だけ起動するようにする \| 為せばnull](https://blog.yotio.jp/2025/10/19/linux-cron-rtcwake.html)

自宅の Raspberry Pi 5 サーバでもこれができないかなと思ったんですが、ラズパイは設定先が異なっていたのでメモしておきます。

<!--more-->	

# 実行環境

- サーバ
  - Raspberry Pi 5
  - Ubuntu 24.04.3 LTS (Linux 6.8.0)

# Raspberry Pi 5 には RTC がある

まず、意外にも（？）Raspberry Pi 5 はきちんと RTC を備え付けています。

ただし RTC の電池パックは付属していません。ラズパイの公式の RTC 用電池パックは別売り1000円。ちょっとお高い…と悩んでいたのですが、今回の利用用途では**電池パックは必要ありません。**今回は電源に繋いだ状態でシャットダウン状態から復帰させたいだけですので、電池パックは買わずとも USB-C 端子から RTC に電力供給されます。

# 設定先

```
/sys/class/rtc/rtc0/wakealarm
```

ここに UNIX TIME で自動起動させたい時刻を書き込めばおkです。

ちなみに UNIX TIME の代わりに、例えば  `+600` と書くと 600秒後（10分後）に起動時間がセットされるらしい。

# 決まった時間にシャットダウン＆決まった時間に起動

cron と組み合わせれば簡単に実現できます。例えば深夜2時にシャットダウンして、翌朝（当日）6時に起動させたい場合。

まずは crontab を開いて…

```bash
$ sudo crontab -e
```

下記の行を追加します。（明日の時刻を指定したい場合は、today を tomorrow に変更してください）

```
# m h  dom mon dow   command
00 02 * * * /usr/bin/date -d 'today 06:00' +\%s | tee /sys/class/rtc/rtc0/wakealarm && /usr/sbin/shutdown
```

これを保存し、cron を再起動することで反映されます。

```bash
$ sudo systemctl restart cron
```

最後に、一応 `crontab -l` で設定内容を確認しておくとよいでしょう。

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
30 03 * * * /usr/bin/date -d 'today 06:45' +\%s | tee /sys/class/rtc/rtc0/wakealarm && /usr/sbin/shutdown
```



以上です。
