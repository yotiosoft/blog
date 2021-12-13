---
layout: post
title: "Win32APIのRestart Managerでファイルをロックしているプロセスを特定する"
tags: [C/C++, Win32API, Windows]
excerpt_separator: <!--more-->
---

ふと、「このファイルは他のプログラムによって使用されています」とWindowsで表示された時に、どのプロセスがファイルをロックしているのかささっと特定できるツールがあれば便利だなと思い、その実現方法を調査しました。  

わざわざ作らなくてもWindows標準付属のリソースモニターを使えば確認できますが、いちいちリソースモニターを開いてファイル名を入力するのが面倒だったのでツールも自作しました。

<!--more-->  

# 実現方法

Win32APIの``Restart Manager``というAPIで実現できます。  
[[Restart Manager について - Win32 apps | Microsoft Docs](https://docs.microsoft.com/ja-jp/windows/win32/rstmgr/about-restart-manager)]{:target="_blank"}  
手順としては、こう。  

1. Restart Managerセッションを開始する（RmStartSession）
2. 調査対象のファイルをセッションに登録する（RmRegisterResources）
3. そのファイルをロックしているプロセスの一覧を取得する（RmGetList）
4. 用が済んだらセッションを終了する（RmEndSession）

``Restart Manager``は、更新プログラムを実行する際に更新しなければならないファイルを開いているプロセスを特定し、そのプロセスを再起動させるためのAPIですが、再起動はさせなくともファイルをロックしているプロセスを特定するだけの目的でも利用できます。ちなみに同様の機能を提供する``IFileIsInUse``というAPIも存在しますが、こちらはまだ試していません。使い勝手はどちらもほぼ同じだと思います。

# 注意点

Restart Managerでは比較的簡単にファイルをロックしているプロセスを特定できますが、注意すべきはファイルをロックしているプロセス"しか"特定できないということです。ファイルを何らかのプロセスが開いているとき、以下の2通りの状態が考えられます。  

1. ファイルがロックされている
2. ファイルはロックされていないが、何らかのプロセスがファイルを開いている

前者はRestart Managerで特定可能です。WordファイルをMS Wordで開くと、ファイルがロックされRestart ManagerでもWordがロックしていることを特定できます。ファイルがロックされているとき、ファイルを削除したり移動したり名称変更したりすることはできません。実は、ファイルがロックされている場合はエクスプローラーでも「何がファイルをロックしているか」が表示されます。  
![SnapCrab_使用中のファイル_2021-12-13_22-14-26_No-00](../../../assets/img/post/2021-12-07-Win32ApiのRestart Managerでファイルをロックしているプロセスを特定する/SnapCrab_使用中のファイル_2021-12-13_22-14-26_No-00.png)  

一方で、後者はRestart Managerで特定できません。ファイルがロックされていないのでファイルを削除、移動、名称変更することは可能ですが、そのファイルが存在するフォルダを移動したり名称変更しようとすると「別のプログラムがこのフォルダーまたはファイルを開いているので、操作を完了できません」と表示されます。  
![SnapCrab_使用中のフォルダー_2021-12-13_22-11-4_No-00](../../../assets/img/post/2021-12-07-Win32ApiのRestart Managerでファイルをロックしているプロセスを特定する/SnapCrab_使用中のフォルダー_2021-12-13_22-11-4_No-00.png)  
これではどのプロセスが原因なのかわかりません。本当はこっちを特定できるようにしたかったのですが、Win32APIのドキュメント化されていない関数を使わなければならず難しいというのが正直なところです。これに関しては色々試しているところで、実現できたらまた記事に書きます。  

というわけで、今回実現できるのは「ファイルがロックされている」場合のみです。

# CUIで動かす

作成中

# GUIで動かす

ドラッグアンドドロップで動くようにしたかったので、GUIで実装しました。ライブラリにOpenSiv3Dを利用しています。

```c++
#include <Siv3D.hpp> // OpenSiv3D v0.6.3
#include <Siv3D/Windows/Windows.hpp>
#include <string>

#include <winternl.h>
#include <ntstatus.h>

#include <RestartManager.h>
#include <winerror.h>

#pragma comment(lib, "Rstrtmgr.lib")

void Main()
{
	// 背景の色を設定 | Set background color
	Scene::SetBackground(ColorF{ 0.3, 0.3, 0.3 });

	Print << U"Drop File Here!";

	while (System::Update())
	{
		// ファイルがドラッグアンドドロップされたら
		if (DragDrop::HasNewFilePaths()) {
			// 文字をクリア
			ClearPrint();

			// ファイルの総数（1つ）
			DroppedFilePath file = DragDrop::GetDroppedFilePaths()[0];
			int files_n = 1;

			// RestartManagerセッションの開始
			DWORD dw_session, dw_error;
			WCHAR sz_session_key[CCH_RM_SESSION_KEY + 1]{};

			dw_error = RmStartSession(&dw_session, 0, sz_session_key);
			if (dw_error != ERROR_SUCCESS) {
				throw std::runtime_error("fail to start restart manager.");
			}

			// ファイルをセッションに登録
			//wchar_t* buf;
			//file_path.toWstr().assign(buf);

			std::wstring wstr = file.path.toWstr();
			PCWSTR pcwstr = wstr.c_str();
			Print << Unicode::FromWstring(pcwstr);

			dw_error = RmRegisterResources(dw_session, files_n, &pcwstr, 0, NULL, 0, NULL);
			if (dw_error != ERROR_SUCCESS) {
				Console << U"Err";
				throw std::runtime_error("fail to register target files.");
			}

			// そのファイルを使用しているプロセスの一覧を取得
			UINT n_proc_info_needed = 0;
			UINT n_proc_info = 1;
			RM_PROCESS_INFO* rgpi = new RM_PROCESS_INFO[n_proc_info];
			DWORD dw_reason;

			dw_error = RmGetList(dw_session, &n_proc_info_needed, &n_proc_info, rgpi, &dw_reason);

			if (dw_error == ERROR_MORE_DATA) {		// rgpiの足りない分を追加
				delete[] rgpi;
				n_proc_info = n_proc_info_needed;
				rgpi = new RM_PROCESS_INFO[n_proc_info];
				dw_error = RmGetList(dw_session, &n_proc_info_needed, &n_proc_info, rgpi, &dw_reason);	// もう一度取得
				Print << U"this file is opened by " << (int)n_proc_info_needed << U" processes.";
			}
			else if (dw_error == ERROR_SUCCESS) {
				Print << U"this file is opened by " << (int)n_proc_info_needed << U" processes.";
			}
			else {
				Print << U"Error";
				break;
			}
			if (dw_error != ERROR_SUCCESS) {
				throw std::runtime_error("fail to get process list.");
			}

			for (int i = 0; i < n_proc_info_needed; i++) {
				Print << U"---------------------------------------------------------------";
				Print << U"プロセスID: " << rgpi[i].Process.dwProcessId;
				Print << U"アプリ名: " << Unicode::FromWstring((std::wstring)rgpi[i].strAppName);
				Print << U"アプリのタイプ: " << rgpi[i].ApplicationType;
				Print << U"---------------------------------------------------------------";
			}

			// セッション終了
			RmEndSession(dw_session);
		}
	}
}
```


動作確認として、

1. まだ開いていないWordファイルをドラッグ&ドロップ
2. そのファイルをWordで開く
3. もう一度同じWordファイルをドラッグ&ドロップ

の順序で試してみました。  

<video src="../../../assets/img/post/rmwhichgi.mp4" controls></video>

とりあえずWordが特定できてるっぽいので、Restart Managerの利用法としては成功。  
ただし、前述の通りTerapadのようにファイルをロックしないプロセスは特定できません。  
![SnapCrab_NoName_2021-12-13_23-56-5_No-00](../../../assets/img/post/2021-12-07-Win32ApiのRestart Managerでファイルをロックしているプロセスを特定する/SnapCrab_NoName_2021-12-13_23-56-5_No-00.png)  
次はファイルをロックしないプロセスも特定できるようにしたいなあ。
