---
layout: post
title: "C言語のaccept関数でInvalid argumentと出る"
tags: [C/C++]
excerpt_separator: <!--more-->
---

<<<<<<< HEAD
大学のソケットプログラミングの課題でクライアント・サーバモデルのTCP通信を行うプログラムを書いていたら、acceptを呼び出す時にたまにエラーが発生しました。  
=======
ソケットプログラミングでクライアント・サーバモデルのTCP通信を行うプログラムを書いていたら、acceptを呼び出す時にたまにエラーが発生しました。  
>>>>>>> 37c99ba7cc57ff4b616041e42ae337ed395b2209

```
accept: Invalid argument
```

<<<<<<< HEAD
エラーが出るときは何度でも出るし、出ないときは全然出ない。うーん、謎だ…
=======
エラーが出るときはやり直しても何度でも出るし、出ないときは全然出ない。うーん、謎だ…
>>>>>>> 37c99ba7cc57ff4b616041e42ae337ed395b2209

<!--more-->

# 環境

- Ubuntu 20.04.1 LTS (64bit)
- CPU: Intel Core i5-6500 CPU @ 3.20GHz
- RAM: 15.5GiB
- コンパイラ: gcc 9.3.0

# プログラム

複数のクライアントからのメッセージをサーバが拾い、サーバ側で受信したメッセージを表示するプログラム。今回は接続に応じて個別の通信用スレッドを生成することでマルチアクセスに対応しています。  

サーバ側（tcp_multi_server.c）:  

```c
#include <stdio.h>
#include <stdlib.h>

#include <string.h>
#include <errno.h>

#include <unistd.h>
#include <netdb.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#include <pthread.h>

<<<<<<< HEAD
#define	PORT	6000
=======
#define	PORT	50000
>>>>>>> 37c99ba7cc57ff4b616041e42ae337ed395b2209
#define BUFSIZE	2048

// 子スレッドの処理（受信＆表示）
void *child_process(int *fd_socket) {
<<<<<<< HEAD
	int fd, recv_len;
	char buf[BUFSIZE];
	fd = (int)*fd_socket;

	recv_len = recv(fd, buf, BUFSIZE, 0);
	if (recv_len > 0) {gcc ./tcp_threads_2.c -o tcp_threads_2 -lpthread
		// 標準出力に受信内容を書き込み
		write(1, buf, recv_len);
	}
	close(fd);
}

int main() {
	struct sockaddr_in saddr, caddr;
	int fd1, fd2, len;
	pthread_t pt;

	// サーバーのソケットを生成
	fd1 = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
	if (fd1 < 0) {
		perror("socket");
		return -1;
	}

	// ソケットの設定
	memset(&saddr, 0, sizeof(saddr));
	saddr.sin_family = AF_INET;
	saddr.sin_port = htons(PORT);
	saddr.sin_addr.s_addr = htonl(INADDR_ANY);

	// 設定をbind
	if (bind(fd1, (struct sockaddr*)&saddr, sizeof(saddr))) {
		perror("bind");
		return -1;
	}

	// 接続待機
	if (listen(fd1, 5)) {
		perror("listen");
		return -1;
	}

	// 接続し次第子スレッドを生成
	// 続きは子スレッドが行う -> child_process
	while (1) {
		fd2 = accept(fd1, (struct sockaddr*)&caddr, &len);
		if (fd2 < 0) {
			perror("accept");
			exit(1);
		}

		if (pthread_create(&pt, NULL, (void*)(child_process), (void*)&fd2) < 0) {
			perror("pthread_create");
			return -1;
		}
		pthread_detach(pt);
	}

	return 0;
=======
  int fd, recv_len;
  char buf[BUFSIZE];
  fd = (int)*fd_socket;

  recv_len = recv(fd, buf, BUFSIZE, 0);
  if (recv_len > 0) {gcc ./tcp_threads_2.c -o tcp_threads_2 -lpthread
    // 標準出力に受信内容を書き込み
    write(1, buf, recv_len);
  }
  close(fd);
}

int main() {
  struct sockaddr_in saddr, caddr;
  int fd1, fd2, len;
  pthread_t pt;

  // サーバーのソケットを生成
  fd1 = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
  if (fd1 < 0) {
    perror("socket");
    return -1;
  }

  // ソケットの設定
  memset(&saddr, 0, sizeof(saddr));
  saddr.sin_family = AF_INET;
  saddr.sin_port = htons(PORT);
  saddr.sin_addr.s_addr = htonl(INADDR_ANY);

  // 設定をbind
  if (bind(fd1, (struct sockaddr*)&saddr, sizeof(saddr))) {
    perror("bind");
    return -1;
  }

  // 接続待機
  if (listen(fd1, 5)) {
    perror("listen");
    return -1;
  }

  // 接続し次第子スレッドを生成
  // 続きは子スレッドが行う -> child_process
  while (1) {
    fd2 = accept(fd1, (struct sockaddr*)&caddr, &len);
    if (fd2 < 0) {
      perror("accept");
      exit(1);
    }

    if (pthread_create(&pt, NULL, (void*)(child_process), (void*)&fd2) < 0) {
      perror("pthread_create");
      return -1;
    }
    pthread_detach(pt);
  }

  return 0;
>>>>>>> 37c99ba7cc57ff4b616041e42ae337ed395b2209
}
```

クライアント側（tcp_client.c）:  

```c
#include <stdio.h>
#include <stdlib.h>

#include <string.h>
#include <errno.h>

#include <unistd.h>
#include <netdb.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <netinet/in.h>
#include <arpa/inet.h>

<<<<<<< HEAD
#define PORT 6000 

int main(int argc, char **argv) {
	struct sockaddr_in saddr;
	int fd;
	char *buf="Hello!\n";

	// ソケットを生成
	fd = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (fd < 0) {
		perror("socket");
		return -1;
	}

    // ソケットの設定
	memset(&saddr, 0, sizeof(saddr));
	saddr.sin_family      = AF_INET;
	saddr.sin_port        = htons(PORT);
	saddr.sin_addr.s_addr = inet_addr("127.0.0.1");
	
    // サーバに接続
	if (connect(fd, (struct sockaddr*)&saddr, sizeof(saddr)) < 0) {
		perror("connect");
		exit(-1);
	}

    // メッセージbufを送信
	send(fd, buf, strlen(buf), 0);

	close(fd);

	return 0;
=======
#define PORT 50000

int main(int argc, char **argv) {
  struct sockaddr_in saddr;
  int fd;
  char *buf="Hello!\n";

  // ソケットを生成
  fd = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
  if (fd < 0) {
    perror("socket");
    return -1;
  }

  // ソケットの設定
  memset(&saddr, 0, sizeof(saddr));
  saddr.sin_family      = AF_INET;
  saddr.sin_port        = htons(PORT);
  saddr.sin_addr.s_addr = inet_addr("127.0.0.1");
	
  // サーバに接続
  if (connect(fd, (struct sockaddr*)&saddr, sizeof(saddr)) < 0) {
    perror("connect");
    exit(-1);
  }

  // メッセージbufを送信
  send(fd, buf, strlen(buf), 0);

  close(fd);

  return 0;
>>>>>>> 37c99ba7cc57ff4b616041e42ae337ed395b2209
}
```

クライアントはメッセージ「Hello!」を送信したら終了します。サーバ側はプロセスを中断させるまでクライアントからの接続を無限に待機し続けます。  

<<<<<<< HEAD
ビルド方法は以下の通り。  
=======
gccでのコンパイル用のコマンドは以下の通り。  
>>>>>>> 37c99ba7cc57ff4b616041e42ae337ed395b2209

```bash
$ gcc tcp_multi_server.c -o tcp_multi_server -lpthread
$ gcc tcp_client.c -o tcp_client
```

<<<<<<< HEAD
サーバ側を先に立ち上げ、クライアント側を後に立ち上げるとメッセージ「Hello!」が自動的に送信されます。
=======
サーバ側を先に起動し、クライアント側を後に起動するとメッセージ「Hello!」が自動的にクライアントからサーバへ送信されます。
>>>>>>> 37c99ba7cc57ff4b616041e42ae337ed395b2209

# 問題点

数回に一度程度ですが、クライアントがサーバに接続されたとき、サーバ側でこんなエラーが出て強制終了してしまいます。  

```bash
$ ./tcp_multi_server
accept: Invalid argument
```

<<<<<<< HEAD
acceptに無効な引数が指定されているとのこと。

# 原因

[manページ](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/accept.2.html){:target="_blank"}によると、accept関数の引数
=======
acceptに無効な引数が指定されているとのこと。  

[man page](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/accept.2.html){:target="_blank"}によると、accept関数の引数
>>>>>>> 37c99ba7cc57ff4b616041e42ae337ed395b2209

```c
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

<<<<<<< HEAD
のうち、3番目の引数``addr_len``は入出力両用の引数で、accept関数の呼び出し時には事前に``addr``が指し示す構造体のサイズで初期化しなければならないとのこと。  
=======
のうち、3番目の引数``addrlen``は入出力両用の引数で、accept関数の呼び出し時には事前に``addr``が指し示す構造体のサイズで初期化しなければならないとのこと。  
>>>>>>> 37c99ba7cc57ff4b616041e42ae337ed395b2209

つまり、先に示したtcp_multi_server.cのうちの変数``len``に、accept関数を呼び出す前に構造体``caddr``のサイズを代入して置かなければならなかったということです。

```c
// 問題の箇所
fd2 = accept(fd1, (struct sockaddr*)&caddr, &len);
```

# 解決策

<<<<<<< HEAD
accept文を呼び出す直前に、``caddr``のサイズを``len``に代入しておけばOK。  

tcp_multi_server.c：  
=======
accept文を呼び出す直前に、``caddr``のサイズを``len``に代入しておけばOK。sizeof関数でサイズを取得しています。  
>>>>>>> 37c99ba7cc57ff4b616041e42ae337ed395b2209


```c
len = sizeof(caddr);  // 追加
fd2 = accept(fd1, (struct sockaddr*)&caddr, &len);
```


<<<<<<< HEAD
サーバ側プログラム（tcp_multi_server.c）の修正版がこちら：  
=======
サーバ側プログラム（tcp_multi_server.c）の修正版：  
>>>>>>> 37c99ba7cc57ff4b616041e42ae337ed395b2209

```c
#include <stdio.h>
#include <stdlib.h>

#include <string.h>
#include <errno.h>

#include <unistd.h>
#include <netdb.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#include <pthread.h>

<<<<<<< HEAD
#define	PORT	6000
=======
#define	PORT	50000
>>>>>>> 37c99ba7cc57ff4b616041e42ae337ed395b2209
#define BUFSIZE	2048

// 子スレッドの処理（受信＆表示）
void *child_process(int *fd_socket) {
<<<<<<< HEAD
	int fd, recv_len;
	char buf[BUFSIZE];
	fd = (int)*fd_socket;

	recv_len = recv(fd, buf, BUFSIZE, 0);
	if (recv_len > 0) {gcc ./tcp_threads_2.c -o tcp_threads_2 -lpthread
		// 標準出力に受信内容を書き込み
		write(1, buf, recv_len);
	}
	close(fd);
}

int main() {
	struct sockaddr_in saddr, caddr;
	int fd1, fd2, len;
	pthread_t pt;

	// サーバーのソケットを生成
	fd1 = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
	if (fd1 < 0) {
		perror("socket");
		return -1;
	}

	// ソケットの設定
	memset(&saddr, 0, sizeof(saddr));
	saddr.sin_family = AF_INET;
	saddr.sin_port = htons(PORT);
	saddr.sin_addr.s_addr = htonl(INADDR_ANY);

	// 設定をbind
	if (bind(fd1, (struct sockaddr*)&saddr, sizeof(saddr))) {
		perror("bind");
		return -1;
	}

	// 接続待機
	if (listen(fd1, 5)) {
		perror("listen");
		return -1;
	}

	// 接続し次第子スレッドを生成
	// 続きは子スレッドが行う -> child_process
	while (1) {
        len = sizeof(caddr);  // 追加
		fd2 = accept(fd1, (struct sockaddr*)&caddr, &len);
		if (fd2 < 0) {
			perror("accept");
			exit(1);
		}

		if (pthread_create(&pt, NULL, (void*)(child_process), (void*)&fd2) < 0) {
			perror("pthread_create");
			return -1;
		}
		pthread_detach(pt);
	}

	return 0;
}
```

結果、上記の1行を追加するだけでこのエラーが出なくなりました。

# おわりに

このacceptのエラー、出るときもあれば出ないときもあって最初は無視していたんですが、後になって頻発したので「時と場合による」現象だと思われます。  
今回はgccで検証しましたが、Cygwinでも``accept: Bad address``と出て、同様の方法で解決したそうです。一方で、Gentoo Linuxではエラーの再現ができなかったとか…うーん謎だ。  
ともかく、manページに初期化するよう明記されているので、環境依存のエラーを回避するためにも``addrlen``はきちんと初期化したほうが良いかと思います。
=======
  int fd, recv_len;
  char buf[BUFSIZE];
  fd = (int)*fd_socket;

  recv_len = recv(fd, buf, BUFSIZE, 0);
  if (recv_len > 0) {gcc ./tcp_threads_2.c -o tcp_threads_2 -lpthread
    // 標準出力に受信内容を書き込み
    write(1, buf, recv_len);
  }
  close(fd);
}

int main() {
  struct sockaddr_in saddr, caddr;
  int fd1, fd2, len;
  pthread_t pt;

  // サーバーのソケットを生成
  fd1 = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
  if (fd1 < 0) {
    perror("socket");
    return -1;
  }

  // ソケットの設定
  memset(&saddr, 0, sizeof(saddr));
  saddr.sin_family = AF_INET;
  saddr.sin_port = htons(PORT);
  saddr.sin_addr.s_addr = htonl(INADDR_ANY);

  // 設定をbind
  if (bind(fd1, (struct sockaddr*)&saddr, sizeof(saddr))) {
    perror("bind");
    return -1;
  }

  // 接続待機
  if (listen(fd1, 5)) {
    perror("listen");
    return -1;
  }

  // 接続し次第子スレッドを生成
  // 続きは子スレッドが行う -> child_process
  while (1) {
    len = sizeof(caddr);  // 追加
    fd2 = accept(fd1, (struct sockaddr*)&caddr, &len);
    if (fd2 < 0) {
      perror("accept");
      exit(1);
    }

    if (pthread_create(&pt, NULL, (void*)(child_process), (void*)&fd2) < 0) {
      perror("pthread_create");
      return -1;
    }
    pthread_detach(pt);
  }

  return 0;
}
```

クライアント側は修正する必要はありません。

# おわりに

このacceptのエラー、出るときもあれば出ないときもあって最初は無視していたんですが、後になって頻発したので、時と場合によると思われます。  
今回はUbuntuで検証しましたが、他の方もCygwinで``accept: Bad address``とエラーが出て、同様の方法で解決したようです。一方で、別の環境（Gentoo Linuxなど）ではエラーの再現ができなかったとか…何故だろう？  
ともかく、man pageに``addrlen``を初期化するよう明記されているので、環境依存のエラーを回避するためにも、前もって``addrlen``を初期化させたほうが良いかと思います。
>>>>>>> 37c99ba7cc57ff4b616041e42ae337ed395b2209
