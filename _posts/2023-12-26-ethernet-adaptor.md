---
layout: post
title: "LANアダプタのドライバが自動インストールできなかったときので対処した"
tags: [その他, Windows]
excerpt_separator: <!--more-->
---

クリスマスイブの日、ソロ充を楽しもうと秋葉原の電気街を散策していたときに、こちらのイーサネットアダプタを購入しました。



「電脳大作戦 USB 接続 LAN アダプタ DEN-004」

![PXL_20231225_151804766.MP.jpg](..\..\..\assets\img\post\2023-12-26\PXL_20231225_151804766.MP.jpg)

価格は 600 円台と、数千円する他のアダプタと比べてお安いです。実は以前 MacBook 用に（おそらく）同型のものを購入したのですが紛失してしまい、今回は Windows  PC 用に買い直しました。

で、以前 Mac に繋いだときはカーネルに組み込まれていたためか特に何もせずとも勝手にイーサネットに接続できたのですが、Windows に繋いだときはひと悶着あったので、今回はその記録です。

<!--more-->

# PC 環境

- メーカー：Diginnos

- 型番：DG-STK1B

- OS：Windows 10 Home 32bit

- CPU：Intel Atom Z3735F

- RAM：2GB

こちらは2017年の元日に2017円で購入したスティック型 PC です。購入してから数年間使わず放置してました。

ファイルサーバ 兼 VPN サーバにでもしようかと思っていたのですが、LAN ポートを搭載しておらずイーサネットに接続できないためアダプタを購入しました。32bit 版 Windows ということでサポート終了まで秒読みの状態ですが、今回は実験的に組むだけなので本格運用する際に Linux に入れ直そうかと思っています。

# 状況

- DEN-004 を Windows 機に挿し込んだ

- ドライバのインストールが始まったが、失敗した

- ドライバを検索しても「このデバイス用のドライバーが見つかりませんでした」と表示される



まず、LAN アダプタを挿入した後に Windows のネットワーク設定を見てもイーサネット接続は確認できず。

![SnapCrab_設定_2023-12-26_0-3-59_No-00.png](..\..\..\assets\img\post\2023-12-26\SnapCrab_設定_2023-12-26_0-3-59_No-00.png)



次にデバイスドライバを見てみると、「ネットワークデバイス」には LAN アダプタのドライバは見当たりません。代わりに「その他のデバイス」に「USB 2.0 10/100M Ethernet adaptor」なるもの警告付きで表示されています。

![SnapCrab_デバイス マネージャー_2023-12-26_0-2-52_No-00.png](..\..\..\assets\img\post\2023-12-26\SnapCrab_デバイス%20マネージャー_2023-12-26_0-2-52_No-00.png)



どうもこの「USB 2.0 10/100M Ethernet adaptor」が今回購入した LAN アダプタのドライバのようなんですが、エラーコード 28 でインストールが中断してしまっています。どうやらドライバを検索したが見つからなかった模様。

![SnapCrab_USB 20 10100M Ethernet Adaptorのプロパティ_2023-12-26_0-5-10_No-00.png](..\..\..\assets\img\post\2023-12-26\SnapCrab_USB%2020%2010100M%20Ethernet%20Adaptorのプロパティ_2023-12-26_0-5-10_No-00.png)



「ドライバーの更新」を選択しても、Windows 側でドライバを見つけられなかったという旨の下記のエラーメッセージが表示されるのみです。

（撮影ミスって変な領域でキャプチャされてしまいました）

![SnapCrab_USB 20 10100M Ethernet Adaptorのプロパティ_2023-12-25_23-55-32_No-00.png](..\..\..\assets\img\post\2023-12-26\SnapCrab_USB%2020%2010100M%20Ethernet%20Adaptorのプロパティ_2023-12-25_23-55-32_No-00.png)



# 解決策

ドライバ名で検索したものの、この LAN アダプタのドライバを公開している公式サイトらしきものは見当たらず…。

似たような名前のドライバを公開しているページがヒットしますが、商品写真を見る限りこちらは名前が似ているだけの別物です。

[USB 2.0 - 10/100 Mbps Ethernet 有線LANアダプタ - USB &amp; USB-C ネットワークアダプタ | StarTech.com 日本](https://www.startech.com/ja-jp/networking-io/usb2100){:target="_blank"}



そこでヒットしたのがこちらの動画。

[Download RD9700 USB2.0 to Fast Ethernet Adapter driver for Windows 7 / 8 - YouTube](https://youtu.be/DzkiH-Tu4KA)

どうやらこちらの商品、OEM のようで、「RD9700 USB2.0 to Fast Ethernet Adapter」なるものがこの LAN アダプタのドライバに当たるようです。こちらの動画にはドライバのダウンロードリンクが概要欄に貼られていますが、今見たらドライバファイルが非公開になっていました。

というわけで、ネットに転がってる同名の LAN アダプタのドライバーをインストールすることで解決しました。怪しいダウンローダサイトが多数ヒットしますので、一番マシそうな archive.org からダウンロードしました。ただしこちらも公式のものではないので自己責任でお願いします。

[RD9700 USB Ethernet adapter driver disc : Free Download, Borrow, and Streaming : Internet Archive](https://archive.org/details/rd9700){:target="_blank"}



![SnapCrab_NoName_2023-12-25_23-59-42_No-00.png](..\..\..\assets\img\post\2023-12-26\SnapCrab_NoName_2023-12-25_23-59-42_No-00.png)

上記のリンクから「WINDOWS EXECUTABLE」を選択して Setup.exe ダウンロードし、

![SnapCrab_NoName_2023-12-26_0-2-7_No-00.png](..\..\..\assets\img\post\2023-12-26\SnapCrab_NoName_2023-12-26_0-2-7_No-00.png)

Setup.exe を実行します。

![SnapCrab_RD9700 USB Ethernet Adapter - InstallShield Wizard_2023-12-26_0-2-35_No-00.png](..\..\..\assets\img\post\2023-12-26\SnapCrab_RD9700%20USB%20Ethernet%20Adapter%20-%20InstallShield%20Wizard_2023-12-26_0-2-35_No-00.png)

あとは指示に従ってインストールするだけ。

インストール完了後、再起動するとドライバが適用されました。

![SnapCrab_デバイス マネージャー_2023-12-26_0-6-9_No-00.png](..\..\..\assets\img\post\2023-12-26\SnapCrab_デバイス%20マネージャー_2023-12-26_0-6-9_No-00.png)

イーサネットにも繋がりましたとさ。めでたしめでたし。

![SnapCrab_設定_2023-12-26_0-6-16_No-00.png](..\..\..\assets\img\post\2023-12-26\SnapCrab_設定_2023-12-26_0-6-16_No-00.png)
