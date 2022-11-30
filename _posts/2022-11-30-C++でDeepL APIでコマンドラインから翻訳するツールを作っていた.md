---
layout: post
title: "C++でDeepL APIを使ってコマンドラインから翻訳するツールを作っ（てい）た"
tags: [dptran, C/C++]
excerpt_separator: <!--more-->
---

今年の春頃、自分で使う用に、コマンドライン入力された文章を DeepL API で翻訳して結果を返すプログラムを C++ で作っていました。その名も **dptran** です。  
結局完成せず非公開のまま、しばらく開発も放棄していました。しかし、最近 Rust で dptran を作り直し始め、せっかくだから以前書いた C++ 版も、ソースコードだけでも公開しておくかと思いこの記事を書きました。

<!--more-->  

# リポジトリ

[https://github.com/YotioSoft/dptran-cpp/](https://github.com/YotioSoft/dptran-cpp/){:target="_blank"}


# 概要

コマンドラインからDeepL翻訳を利用できます。基本的な使い方はこう。  

**翻訳モード**

```bash
$ dptran Hello
こんにちは
$ dptran Hello -t FR
Bonjour
```

翻訳モードでは、引数に原文を渡すことで翻訳結果が返されます。  
``-t``オプションで翻訳先言語を指定し、``-f``オプションで翻訳元言語を指定します。  
デフォルトでは翻訳先言語は日本語に、翻訳元言語はDeepLによる自動推論になっています。  
ただ、普段の使用では、翻訳元言語は指定することはほとんどないと思います。  

**対話モード**

```bash
$ dptran
> Hello

こんにちは
> Ich stehe jeden Tag um 7 Uhr auf.

毎日7時に起きています。
:q
```

対話モードでは、対話形式で複数の原文を連続して翻訳することができます。  
翻訳モード同様、``-t``オプションで翻訳先言語を指定し、``-f``オプションで翻訳元言語を指定します。途中で変更することはできません（翻訳元の自動推論は除く）。  
``:q``で対話モードを終了します。

詳しい使用方法はreadmeをご覧ください。  
[https://github.com/YotioSoft/dptran-cpp/blob/master/README.md](https://github.com/YotioSoft/dptran-cpp/blob/master/README.md){:target="_blank"}

# 仕組み

内容は単純で、libcurl を使って DeepL API に原文とリクエストを POST し、そのレスポンスから翻訳文を取得しているだけです。  
libcurl の curl_easy で API とのやり取りをします。このあたりはいくつかの設定が必要なので、インターフェイスとなるメソッドを作って curl 自体を抽象化しています。  

```c++
#include "connect.h"

// curlの初期設定
bool setup_curl(CURL **curl) {
    *curl = curl_easy_init();

    if (curl == nullptr) {
        cerr << "Error: failed to set up curl" << endl;
        return false;
    }

    return true;
}

// curlを解放
void cleanup_curl(CURL **curl) {
    cout << "\r";

    curl_easy_cleanup(*curl);
}

// curlで受信したときの動作
size_t curl_on_receive(char *ptr, size_t size, size_t nmemb, void *stream) {
    vector<char> *recv_buffer = (vector<char>*)stream;
    const size_t sizes = size * nmemb;
    recv_buffer->insert(recv_buffer->end(), (char*)ptr, (char*)ptr + sizes);
    return sizes;
}

// curlの接続設定
bool connect_curl(CURL **curl, string url, string post_data, string &res_string) {
    vector<char> res_data;
    const char *post_data_c = post_data.c_str();
    const char *url_c = url.c_str();

    curl_easy_setopt(*curl, CURLOPT_URL, url_c);
    curl_easy_setopt(*curl, CURLOPT_POST, 1);
    curl_easy_setopt(*curl, CURLOPT_POSTFIELDS, post_data_c);
    curl_easy_setopt(*curl, CURLOPT_POSTFIELDSIZE, strlen(post_data_c));
    curl_easy_setopt(*curl, CURLOPT_SSL_VERIFYPEER, 1);
    curl_easy_setopt(*curl, CURLOPT_WRITEFUNCTION, curl_on_receive);
    curl_easy_setopt(*curl, CURLOPT_WRITEDATA, &res_data);

    // 通信開始
    CURLcode res = curl_easy_perform(*curl);
    if (res != CURLE_OK) {
        cerr << "Error: curl_easy_perform failed: " << curl_easy_strerror(res) << endl;
        cleanup_curl(curl);
        return false;
    }

    cleanup_curl(curl);

    res_string = string(res_data.data());

    return true;
}
```

```c++
// 翻訳文を取得
int translate(string api_key, string str, string &translated_text, string source_lang_code, string target_lang_code) {
    CURL *curl;

    // curlの初期設定
    if (!setup_curl(&curl)) {
        cleanup_curl(&curl);
        return 1;
    }

    string get_data;
    string post_data;
    
    post_data = "auth_key=" + api_key + "&text=" + str + "&target_lang=" + target_lang_code;
    
    if (source_lang_code != "") {
        post_data += "&source_lang=" + source_lang_code;
    }
    
    cout << "翻訳中.." << flush;
    if (!connect_curl(&curl, "https://api-free.deepl.com/v2/translate", post_data, get_data)) {
        cout << "\r" << string(8, ' ');
        cout << "\r";
        cerr << "Error: 翻訳結果の取得に失敗しました" << endl;
        cleanup_curl(&curl);
        return false;
    }
    cout << "\r" << string(8, ' ');
    cout << "\r";

    picojson::value v;
    string err = picojson::parse(v, get_data);
    if (!err.empty()) {
        cerr << "Error: picojson error - " << err << endl;
        return false;
    }

    translated_text = "";
    picojson::object& v_obj = v.get<picojson::object>();
    picojson::array& v_array = v_obj["translations"].get<picojson::array>();
    for (auto v_element : v_array) {
        picojson::object& v_element_obj = v_element.get<picojson::object>();
        translated_text += v_element_obj["text"].get<string>();
    }
    
    return true;
}
```

DeepL API のレスポンスは JSON 形式で返ってくるので、[picojson](https://github.com/kazuho/picojson){:target="_blank"} でパースしています。こうして、DeepL API に原文を送信し、翻訳結果を取得し、翻訳文を抽出してコンソールに表示するという一連の操作が実現できました。

# なぜボツにしたのか

ソースコードが長ったらしくなったのと、C++ よりも Rust で書きたくなったから、というのが正直な理由です。ちょうど同時期に Rust の習得を始め、ならば今開発中の dptran もこれで書き直してみるか、と。  
結果的に、同じことをするにもC++ で書くより Rust で書いたほうが圧倒的に簡潔に作れました。これに関しては C++、Rust それぞれで使用している curl ライブラリ（クレート）の抽象性の違いにも原因があると考えられますが。  

環境導入の手軽さも理由の一つ。C++ 版のビルドには libcurl のインストールが必要ですので、dptran をユーザがビルドする場合、ユーザの環境にも libcurl を導入してもらわなければなりません。Rust なら``cargo install``してもらえば必要なクレート（もちろん curl も）は一通り自動で導入してくれます。  

メモリ安全性が保証されるのも理由の一つ。今、卒研で書いている C 言語のシステムプログラムでも散々メモリ破壊を目の当たりにしており、もはや自分でメモリ管理するのが億劫です。趣味での開発くらい楽がしたい。  

まとめると、自分が Rust 厨になったのが理由です。いや、もちろん今でも C++ も好きですよ。

# 現状

Rust で作り直しています。だいぶできてきたので、近々ベータ版として公開できたらなぁと思っています。  
[https://github.com/YotioSoft/dptran/](https://github.com/YotioSoft/dptran/){:target="_blank"}

# おわりに

よかったら DeepL API を C++ で触りたいときの参考にでもしてください。
