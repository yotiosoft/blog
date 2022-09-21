---
layout: post
title: "GitHub Pages + Herokuでバックエンドで動く乱数生成アプリを実装（後編）"
tags: [GitHub, Heroku, HTML, CSS, JavaScript, Yapps, 開発日記]
excerpt_separator: <!--more-->
---

前回：[GitHub Pages + Herokuでバックエンドで動く乱数生成アプリを実装（前編）](../14/GitHub-Pages-+-Herokuでバックエンドで動く乱数生成アプリを実装-前編.html)  

前編では、Herokuで動くバックグラウンド側をPython（Flask）で作成しました。今回は後編ということで、GitHub Pagesで動くフロントエンド側を作っていきます。

<!--more-->  

> 注意  
>
> Herokuの無料プランの廃止が発表されました。  
> 2022年11月28日以降、Herokuの無料での利用ができなくなります。ご注意ください。  
>
> 参考：[Heroku's Next Chapter \| Heroku](https://blog.heroku.com/next-chapter){:target"_blank"}

# 今回つくるもの

「Yapps」という自作Webツール集のWebサイトに、新しいツールとして「乱数生成」というものを作成しており、今回はそのフロントエンド部分、すなわちユーザ側が操作する部分を作ります。  

APIとのデータのやり取りはJavaScriptで実装します。いくつか手法がありますが、今回はFetch APIを利用します。  
[Fetch API - Web API \| MDN](https://developer.mozilla.org/ja/docs/Web/API/Fetch_API){:target="_blank"}  
Internet Explorerが全面非対応となっていますが、今やそんなことを気にする必要はありません。


手順としてはこんな感じ。  

1. 連想配列にクエリのパラメータを格納
2. 連想配列をもとにURLSearchParams()でクエリの文字列を生成
3. fetch()でAPIのURLにクエリを結合してアクセス、レスポンスを待機
4. レスポンスがあり次第jsonデータに変換
5. jsonデータに乱数の配列があれば乱数を表示、なければエラーメッセージをalertで表示

4.で「レスポンスがあり次第jsonデータに変換」と書いていますが、前回作成したAPIではレスポンス自体がjson形式の文字列として返されます。よって、この文字列をjsonの文字列として受け取り、.json()でjson構造体にパースする作業を行うのが4.で行う内容です。  

fetch()は非同期処理ですので、サーバから返答があるまでは「乱数を生成中..」などと表示しておきます。

# フロントエンド側の作成

HTMLとJavaScriptで作っていきますが、全部載せると長くなってしまう＆分かりづらくなってしまうので、今回はコードの一部だけ載せておきます。全部見たい場合は[GitHubリポジトリ](https://github.com/YotioSoft/Yapps/tree/main/random){:target="_blank"}からご覧ください。  

## HTML側

今回は、プルダウンメニューで確率分布関数を選択し、選んだ関数に応じてオプションを表示できるようにします。  
まずはプルダウンメニュー、試行回数、「整数で生成」チェックボックスの部分のオプションのコードから。  

```html
<table class="input-table">
    <!--確率分布の指定-->
        <tr>
            <td class="summary">
                <div class="checkbox">
                    確率分布関数
                </div>
            </td>
            <td class="input">
                <select name="select_random" id="id_select_random" class="simple-select" style="width: 300px;">
                    <option value="uniform">一様分布</option>
                    <option value="normal">正規分布</option>
                    <option value="beta">ベータ分布</option>
                    <option value="triangular">三角分布</option>
                    <option value="lambda">ラムダ分布</option>
                    <option value="gamma">ガンマ分布</option>
                </select>
            </td>
        </tr>
        <tr>
            <td class="summary">
                <div class="checkbox">
                    試行回数
                </div>
            </td>
        <td class="input">
            <input type="number" id="id_trials" name="trials" placeholder="試行回数" class="simple-inputtext" value="5">
        </td>
    </tr>
    <tr>
        <td colspan="2" class="wide">
            <div class="checkbox">
                <input type="checkbox" id="id_integer_mode" name="integer_mode">
                <label for="integer_mode">整数で生成</label>
            </div>
        </td>
    </tr>
</table>
```

table上にプルダウンメニューやinputボックスなどを配置しています。  

外観としてはこんな感じ。  
![スクリーンショット 2022-06-24 23.09.10](../../../assets/img/post/2022-6-16-GitHub Pages + Heroku/スクリーンショット 2022-06-24 23.09.10.png)  
この「一様分布」と表示されているプルダウンメニューをクリックして確率分布関数を選びます。  
![スクリーンショット 2022-06-24 23.09.31](../../../assets/img/post/2022-6-16-GitHub Pages + Heroku/スクリーンショット 2022-06-24 23.09.31.png)  
例えば「正規分布」を選ぶと、  
![スクリーンショット 2022-06-24 23.10.57](../../../assets/img/post/2022-6-16-GitHub Pages + Heroku/スクリーンショット 2022-06-24 23.10.57.png)  
このように、メニューの右半分が正規分布用のオプションに表示が切り替わります。  

この表示の切り替えをどのように実現しているかというと、ただ単に、オプション部分をCSSで既定で``display: none``として隠しておき、表示するときは``display: block``に切り替えて可視化しているだけです。  
色々なやり方がありますが、今回はCSSのclassを利用し、``is_show``というクラスがある要素は可視化され、それ以外の要素は隠されるという仕組みを取っています。  

```css
.random_options {
    display: none;      /* 隠しておく（デフォルト） */
}
.random_options.is_show {
    display: block;     /* 可視化 */
}
```

この``is_show``の付与・削除は、プルダウンメニューの選択が切り替わったときにJavaScriptでクラスの要素の切り替えを行っています。  

```javascript
// 選択が切り替わったときに処理を実行(addEventListener)
const select_random = document.getElementById("id_select_random");
select_random.addEventListener('change', function() {
  // 現在のオプションを隠す
  const current_options_column = document.getElementsByClassName("is_show")[0];
  if (current_options_column != null) {
      current_options_column.classList.remove("is_show");
  }

  if (select_random.value == "uniform") {
      // uniformのオプション部分を可視化
      const options = document.getElementsByClassName("uniform_options")[0];
          options.classList.add("is_show");
          options.style.height = "auto";
      }
  else if (select_random.value == "normal") {
      // normalのオプション部分を可視化
      const options = document.getElementsByClassName("normal_options")[0];
      options.classList.add("is_show");
      options.style.height = "auto";
  }
（中略）
});
```

``document.getElementsByClassName("is_show")``で``is_show``というクラスが付与された要素を抽出し、``.classList.remove("is_show")``で``is_show``をその要素から削除することで隠します。  
次に、新たに選択された関数のオプションがある要素に対し、``.classList.add("is_show")``で``is_show``クラスを付与することで可視化します。  

各オプションはこんな感じで定義しています。

```html
<!--パラメータ設定：一様分布-->
<table class="input-table random_options is_show uniform_options">
    <tr>
        <td class="summary">
            <div class="checkbox">
                <input type="radio" id="id_range_limit_uniform" name="range_uniform" checked>
                <label for="id_range_limit">範囲指定</label>
            </div>
        </td>
        <td class="input">
                <input type="number" id="id_input_min_uniform" name="input_min" placeholder="最小値" class="simple-inputtext" value="0">
                <p style="margin-left: 10px; margin-right:10px;">〜</p>
                <input type="number" id="id_input_max_uniform" name="input_mac" placeholder="最大値" class="simple-inputtext" value="100">
        </td>
    </tr>
    <tr>
        <td class="summary">
                <div class="checkbox">
                    <input type="radio" id="id_range_digit_uniform" name="range_uniform">
                    <label for="id_range_digit"_uniform>桁数指定</label>
                </div>
        </td>
        <td class="input">
                <input type="number" id="id_input_digit_uniform" name="input_digit" placeholder="桁数" class="simple-inputtext" value="5">
        </td>
    </tr>
    <tr>
        <td class="summary">
                <div class="checkbox">
                    <input type="radio" id="id_range_nolimit_uniform" name="range_uniform">
                    <label for="id_range_nolimit_uniform">範囲無制限</label>
                </div>
        </td>
    </tr>
</table>
<!--パラメータ設定：正規分布-->
<table class="input-table random_options normal_options">
    <tr>
        <td class="summary">
            <div class="checkbox">
                平均値μ
            </div>
        </td>
        <td class="input">
            <input type="number" id="id_input_mu" name="input_mu" placeholder="平均値μ" class="simple-inputtext" value="0">
        </td>
    </tr>
    <tr>
        <td class="summary">
            <div class="checkbox">
                標準偏差σ
            </div>
        </td>
        <td class="input">
            <input type="number" id="id_input_sigma" name="input_sigma" placeholder="標準偏差σ" class="simple-inputtext" value="1">
        </td>
    </tr>
</table>
（後略）
```

ページアクセス時にデフォルトで表示させたい``uniform_options``に``is_show``を付加させておき、以後はプルダウンメニューで選択が変化した際にクラスを先程のJavaScriptのコードによって書き換える、といった感じです。  

生成ボタンが押されたらJavaScriptの``OnMakeButtonClick()``という関数を実行します（これについては後述）。また、複数の乱数が表示できるよう、乱数の出力部分はtextareaにしています。  

```html
<!--生成ボタン-->
<div class="button-area-center">
    <a href="javascript:OnMakeButtonClick();" id="make-button">
        <div class="round-rect-button">
            <div class="big-text">
                乱数を生成
            </div>
        </div>
    </a>
</div>
                    
<!--生成した乱数の表示-->
<div class="output-area">
    <img src="/img/common/arrow_down.svg" style="width: 64px; height: 64px; margin-bottom: 20px;">
    <textarea id="id_output" placeholder="乱数" class="simple-inputtext" style="width: 500px; max-width: 100%; height: 150px; margin: 0 auto;"></textarea>
</div>
```



html部分の説明すべき点は以上です。全体像はこちらをご覧ください。  
[Yapps/random/index.html](https://github.com/YotioSoft/Yapps/blob/main/random/index.html){:target="_blank"}

## 接続確認用の関数

前述のとおり、JavaScriptのFetch APIを利用します。Herokuは久々にアクセスすると起動に10秒くらいかかるので、（Herokuサーバへの接続確認も兼ねて）フロントエンド側のページにアクセスしたと同時に、以下に示す``wakeup()``という関数を実行させることでHeroku側のアプリを起動させます。  

```javascript
// 接続確認用
function wakeup() {
    fetch('https://murmuring-taiga-39514.herokuapp.com/')
    .then(response => {
        if (!response.ok) {
            alert("サーバエラーが発生しました。しばらくお待ちいただき、後でもう一度お試しください。");
        }
    })
    .catch(error => {
        alert(`乱数生成APIにアクセスできません。\nしばらくお待ちいただき、後でもう一度お試しください。\n\n${error}`);
    });
}
```

``wakeup()``ではfetchでアクセスしたときにサーバエラーが発生したり、そもそもURLにアクセスできなかったときにalertでエラーメッセージを表示しておきます。  

index.htmlへのページアクセス時にhtml側で``wakeup()``を実行させておきます。

```html
<script>
    wakeup();
</script>
```



## バックエンド側とのやり取りを行う関数

手順として、ページ側の「乱数を生成」ボタンが押されたら、

1. ``OnMakeButtonClick()``で選ばれた確率分布関数名に応じて呼び出す関数を振り分け
2. 所定の関数でパラメータを受け取り、連想配列にクエリを格納
3. ``send_and_get()``でフロントエンドサーバにクエリを渡し、レスポンスを待機
4. レスポンスがあったら``update_output()``で出力欄に受け取った乱数を表示

という手順で行います。  

以下、順を追って各関数を示します。まずは生成ボタンが押されたときに実行される``OnMakeButtonClick()``から。  

```javascript
function OnMakeButtonClick() {
    const select_random = document.getElementById("id_select_random");
                            
    if (select_random.value == "uniform") {
        uniform();
    }
    else if (select_random.value == "normal") {
        normal();
    }
    else if (select_random.value == "beta") {
        beta();
    }
    else if (select_random.value == "triangular") {
        triangular();
    }
    else if (select_random.value == "lambda") {
        lambda();
    }
    else if (select_random.value == "gamma") {
        gamma();
    }
}
```

プルダウンメニューで選択している関数名に応じて実行する関数を振り分けます。``uniform()``が一様分布、``normal()``が正規分布…といったように、ここではそれぞれのクエリを受け取るための関数を呼び出します。  

次にクエリを受け取る関数を示します。例として一様分布の``uniform()``と正規分布の``normal()``を示します。  

```javascript
// 一様分布
function uniform() {
    // パラメータの取得
    const trials              = document.getElementById('id_trials');
    const integer_mode        = document.getElementById('id_integer_mode');

    const range               = document.getElementsByName('range_uniform');

    const input_limit_min     = document.getElementById('id_input_min_uniform');
    const input_limit_max     = document.getElementById('id_input_max_uniform');

    const input_digit         = document.getElementById('id_input_digit_uniform');

    // クエリパラメータの登録
    const params = {};

    if (integer_mode.checked) {
        params["type"] = "int";
    }
    else {
        params["type"] = "float";
    }

    params["trials"] = trials.value;
 
    if (range[0].checked) {
        params["max"] = parseInt(input_limit_max.value);
        params["min"] = parseInt(input_limit_min.value);
    }
    else if (range[1].checked) {
        let digits = parseInt(input_digit.value);
        let base_num = Math.pow(10, digits-1);
        let max_num = Math.pow(10, digits) - 1;
        
        params["max"] = base_num;
        params["min"] = max_num;
    }

    // 乱数APIと送受信
    send_and_get("uniform", params);
}

// 正規分布
function normal() {
    // パラメータの取得
    const trials              = document.getElementById('id_trials');
    const integer_mode        = document.getElementById('id_integer_mode');

    const input_mu            = document.getElementById('id_input_mu');
    const input_sigma         = document.getElementById('id_input_sigma');

    // クエリパラメータの登録
    const params = {};

    if (integer_mode.checked) {
        params["type"] = "int";
    }
    else {
        params["type"] = "float";
    }

    params["trials"] = trials.value;

    params["mu"]    = input_mu.value;
    params["sigma"] = input_sigma.value;

    // 乱数APIと送受信
    send_and_get("normal", params);
}
```

``params``という連想配列にクエリ名をキーに値を入力ボックスなどから代入していき、それを``set_and_get()``という関数に渡します。  

``set_and_get()``はまさにHerokuのバックエンドサーバとのやり取りを行う部分で、ここでFetch APIを使って乱数の生成のリクエストとレスポンスの受け取りを行います。  

```javascript
function send_and_get(distribution, params) {
    // 「乱数を生成中..」と出力しておく
    const output = document.getElementById('id_output');
    output.value = "乱数を生成中..";

    // パラメータからクエリを生成
    const query = new URLSearchParams(params);

    // JSONをフェッチ
    fetch(`https://murmuring-taiga-39514.herokuapp.com/random/${distribution}?${query}`)
    .then(response => {
        if (!response.ok) {
            alert("サーバエラーが発生しました。しばらくお待ちいただき、後でもう一度お試しください。");
            return;
        }

        return response.json();
    })
    .then(data => {
        if (data.hasOwnProperty('rand_array')) {
            update_output(data.rand_array);     // 結果を出力
        }
        else if (data.hasOwnProperty('error_message')) {
            alert(data.error_message);
        }
    })
    .catch(error => {
        alert(`乱数生成APIにアクセスできません。\nしばらくお待ちいただき、後でもう一度お試しください。\n\n${error}`);
    });
}
```

引数で確率分布関数名``distribution``とクエリパラメータの連想配列``params``を受け取り、これをURLに書き込んでfetchでリクエストを送信、レスポンスを待機します。レスポンスがあり次第、正しい結果が得られていたら``update_output()``という関数で出力に乱数を表示します。エラーメッセージが返ってきた場合はアラートに表示します。  
ここでもサーバエラー時やサーバにアクセスできない場合にエラーメッセージを表示しておきます。  

最後に、出力内容を更新する``update_output()``を示します。  

```javascript
function update_output(rand_array) {
    const output = document.getElementById('id_output');
    output.value = "";
    for (i=0; i<rand_array.length; i++) {
        output.value += rand_array[i] + '\n';
    }
}
```

これは至って単純で、引数に配列として渡された乱数の配列``rand_array``をtextareaに追加しているだけです。  

以上でJavaScript側の説明も完了です。全体像はこちらをご覧ください。  
[Yapps/random/js/random.js](https://github.com/YotioSoft/Yapps/blob/main/random/js/random.js){:target="_blank"}

# 完成品

[乱数生成 - Yapps](https://yapps.yotiosoft.com/random/){:target="_blank"}  

初回アクセス時に乱数生成APIにアクセスするのに時間がかかってしまうのが少し気になりますが、これはHeroku側の仕様なので仕方ありません。概ねいい感じに動いてます。

# 感想

とりあえず、今回はバックエンド側に簡単なAPIを作ってフロントエンド側でリクエストを送信・レスポンスを受け取る部分が作れたので、もっと高度な処理にも挑戦してみたいです。
