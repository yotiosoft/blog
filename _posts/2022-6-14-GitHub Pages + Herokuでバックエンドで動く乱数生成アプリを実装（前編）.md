---
layout: post
title: "GitHub Pages + Herokuでバックエンドで動く乱数生成アプリを実装（前編）"
tags: [GitHub, Heroku, Python]
excerpt_separator: <!--more-->
---

GitHub Pagesではサーバ側の処理を伴うバックエンドの実装はできません。これまではフロントエンドだけでなんとかしてきましたが、YappsのようなWebアプリを作り始めると、やはりバックエンドの処理も欲しい。  
そんなわけで、無料で手頃にバックエンドを実装できるHerokuを利用し、フロントエンドはGitHub Pages、バックエンドはHerokuという組み合わせとし、バックエンド処理をWeb API化してしまうことにしました。

<!--more-->  

今回は少し長くなりそうなので、前編と後編に分けて書きます。前編ではバックエンド側の処理を実装していきます。

# 目的

GitHub Pagesでバックエンド処理を伴うサイトを作りたい。大まかに言うと

1. フロントエンド側（GitHub Pages）でAPIにデータを渡し
2. バックエンド側（Heroku）のAPIでデータを処理し
3. APIのレスポンスとしてフロントエンド側に結果を返して
4. フロントエンド側で結果を表示

といった内容での実装を考えています。フロントエンドとバックエンドを別々のサーバに置くことになるので、バックエンドはWeb API化することにしました。  
Herokuでは無料版ではアクセス数の上限はあるもののサーバーサイドの処理が可能で、フロントエンドを含めてホームページそのものをHeroku上で開設することも可能です。

## じゃあ全部Herokuで良くない？

確かに…. 

とはいえ、GitHub Pagesは設定が手軽な上、アクセス数上限がないというメリットもあります。また、Herokuはしばらくアクセスしないとアプリケーションが休止状態になってしまいますので、うちのサイトみたいにアクセス数が少ないと、再起動のためしょっちゅうアクセスに時間がかかってしまいます。  
そのことを考慮すると、サイトアクセスに必ず必要なフロントエンドはGitHub Pagesで、たまにしかアクセスしなバックエンドはHerokuで実装するのが一番バランスが良いと思った次第です。  

あと、何よりサーバの移転作業がめんどくさい！（これが一番の理由）

# 今回つくるもの

とりあえず、まずは簡単なAPIを作ろうと、Pythonで乱数を生成するだけのAPIを作成することにしました。今回は、PythonのWebアプリフレームワークであるFlaskも利用します。  
このAPIをHeroku側に構築し、GitHub Pages側で自分が運営しているWebアプリ集「Yapps」にフロントエンド側のアプリを作成します。  

乱数といっても、一様乱数を作るだけならフロントエンドのJavaScriptでもできますので、今回は一様乱数だけでなく、正規分布やベータ分布など他の乱数も生成できるようにします。JavaScriptでも頑張れば擬似的に実装できますが、このあたりはPythonのRandomライブラリの方が充実していますから、Herokuの練習がてらPythonのライブラリを利用してやろうというわけです。  

せっかくAPIにするのですから、ルーティングもしておきましょうか。今回のルート構造は、こう。  

- /（ルートディレクトリ：Web APIのアクセス確認用）
  - /random/（各乱数APIの親ディレクトリ）
    - uniform（一様分布で乱数を生成し、値を返す）
    - normal（正規分布で乱数を生成し、値を返す）
    - beta（ベータ分布で乱数を生成し、値を返す）
    - triangular（三角分布で乱数を生成し、値を返す）
    - lambda（ラムダ分布で乱数を生成し、値を返す）
    - gamma（ガンマ分布で乱数を生成し、値を返す）

また、クエリで範囲や戻り値の型（int or float）を指定できるようにします。  
例）

```
xxx.herokuapp.com/random/normal?mu=0&sigma=1 							// μ=0、σ=1で正規分布で乱数生成（整数）
xxx.herokuapp.com/random/uniform?min=0&max=100&type=int		// 0〜100の間で一様分布で乱数生成（浮動小数点数）
```


クエリのパラメータはこんな感じ。  

| パラメータ | 用途                             |
| ---------- | -------------------------------- |
| min        | 最小値（一様分布、三角分布）     |
| max        | 最大値（一様分布、三角分布）     |
| mu         | 平均値μ（正規分布）              |
| sigma      | 標準偏差σ（正規分布）            |
| mode       | 最頻値（三角分布）               |
| lambd      | λ（ラムダ分布、内部で1/λに変換） |
| alpha      | α（ベータ分布、ガンマ分布）      |
| beta       | β（ベータ分布、ガンマ分布）      |
| type       | 戻り値の型（"int" or "float"）   |
| trials     | 試行回数（trials個の乱数を返す） |

  

バックエンド側はレスポンスをJSON形式で返します。そのJSONをフロントエンド側が受け取って結果を表示できれば万事オッケーです。

# 乱数API作成

## 前準備

まずは作業用の適当なディレクトリを作っておき、Pythonのバージョンを記載するruntime.txtを作成します。  
下記のページよりHerokuがサポートしているPythonのバージョンを確認。  
[Heroku の Python サポート \| Heroku Dev Center](https://devcenter.heroku.com/ja/articles/python-support){:target="_blank"}  

今回は``python-3.10.2``を採用しました。以下のようにしてruntime.txtにバージョンを書き込み。  

```zsh
% echo python-3.10.2 > runtime.txt
```


次に必要なモジュールをrequirements.txtに書き込み。今回はNumPyとFlask、gunicorn関係さえあればOKです。必要に応じていらないモジュールは削っておきましょう。  

```zsh
% pip freeze > requirements.txt
```


次にプログラミング言語や実行すべきファイルをProcfileに記載。このとき、``<実行するPythonファイル名（拡張子なし）>:app``としてPythonのプログラムファイル名を指定します。今回はPythonファイルは``app.py``としました。  

```zsh
% web: gunicorn app:app --log-file=- > Procfile
```


設定ファイル類は以上で出揃いました。次にHerokuにログイン。  

```zsh
% heroku login
```

ブラウザが開いてCLIにログインされます。  

いよいよHerokuアプリの作成。  

```zsh
% heroku create
```


git initも済ませておきます。  

```zsh
% git init
```


以上で前準備は完了。

## Flaskのテストプログラムの生成

まずは動作確認として、ルートディレクトリにアクセスしたときに「Hello World!」と表示するだけのプログラムをapp.pyに書きます。  

```python
# -*- coding: utf-8 -*-
from flask import Flask

app = Flask(__name__)

@app.route('/')
def index():
    return "Hello World!"

if __name__ == '__main__':
    app.run()
```


app.pyを保存し、git commitしてherokuにpushします。  

```zsh
% git add .
% git commit -m "test app: Hello World"
% git push heroku master
```

push時に自動的にremote側でビルドやデプロイを行ってくれます。  
うまくいけばビルド結果に``remote: <URL> deployed to Heroku``などと表示されます。完了したら、  

```zsh
% heroku open
```

でアプリのページを開いてみます。すると、  
![スクリーンショット 2022-06-10 20.57.55](../../../assets/img/post/2022-6-9-GitHub Pages + Heroku/スクリーンショット 2022-06-10 20.57.55.png)  
おお、Pythonで書いたHello World!が表示されています。  
とりあえずこれでテストは完了。いよいよ本番のAPIを作っていきます。

## API部分の作成

クエリの取得が必要なので、まずはクエリの取得部分を書いていきます。query_classというクラスにメンバ関数``get()``で各パラメータを読み込みます。このとき、確率分布によって必要なパラメータが異なりますので、``get()``の引数``distribution``で確率分布関数を指定します。  

```python
# クエリのクラス
class query_class:
    # 変数初期化
    def __init__(self):
        self.min = 0
        self.max = 0
        self.mu = 0
        self.sigma = 0
        self.mode = 0
        self.lambd = 0.0
        self.alpha = 0.0
        self.beta = 0.0
        self.trials = 1
        self.type = ""
        self.err = {}

    # クエリパラメータの取得
    def get(self, distribution):
        try:
            # type: 整数 or 浮動小数点数
            arg_type = request.args.get("type")
            if self.type is not None:
                self.type = arg_type

            # trials: 試行回数（発生させる乱数の数）
            arg_trials = request.args.get("trials")
            if arg_trials is not None:
                self.trials = int(arg_trials)

            # 指定された確率分布関数が...
            # 一様分布 or 三角分布の場合  
            if distribution == "uniform" or distribution == "triangular":
                req_min = request.args.get("min")
                req_max = request.args.get("max")

                if self.type == "int":
                    if req_min is None:
                        self.min = -sys.maxsize
                    else:
                        self.min = float(req_min)
                    
                    if req_max is None:
                        self.max = sys.maxsize
                    else:
                        self.max = float(req_max)
                else:
                    if req_min is None:
                        self.min = sys.float_info.min
                    else:
                        self.min = float(req_min)
                    
                    if req_max is None:
                        self.max = sys.float_info.max
                    else:
                        self.max = float(req_max)

                if distribution == "triangular":
                    req_mode = request.args.get("mode")
                    if req_mode is None:
                        self.mode = (self.min+self.max)/2
                    else:
                        self.mode = int(req_mode)
                        
                return

            # 正規分布の場合
            elif distribution == "normal":
                req_mu = request.args.get("mu")
                req_sigma = request.args.get("sigma")

                if req_mu is None:
                    self.mu = 0
                else:
                    self.mu = float(req_mu)
                
                if req_sigma is None:
                    self.sigma = 1
                else:
                    self.sigma = float(req_sigma)
                return

            # ラムダ分布の場合
            elif distribution == "lambda":
                req_lambd = request.args.get("lambd")

                if req_lambd is None:
                    self.lambd = 1
                else:
                    self.lambd = 1/float(req_lambd)
                return

            # ベータ分布 or ガンマ分布の場合
            elif distribution == "beta" or distribution == "gamma":
                req_alpha = request.args.get("alpha")
                req_beta = request.args.get("beta")

                if req_alpha is None:
                    self.alpha = 1.0
                else:
                    self.alpha = float(req_alpha)
                
                if req_beta is None:
                    self.beta = 2.0
                else:
                    self.beta = float(req_beta)
                return

        except ValueError:
            self.err["error_num"] = 1
            self.err["error_message"] = "Error: Arguments are incorrect"
            return
```

必要なパラメータが指定されていない場合は、既定値を代入しています。  
引数のチェック等はバックエンド側で行います。といっても、数値に変換したり関数に渡したときに例外が発生したらValueErrorとして扱うだけです。  

次に、各ディレクトリでの動作を記述していきます。  

```python
# ルートディレクトリ：接続確認用
@app.route('/')
def root_index():
    return "Successfully accessed."

# uniform: 一様分布
@app.route('/random/uniform', methods=["GET"])
def uniform_index():
    rand = []
    jrand = {}
    # クエリパラメータの取得
    query = query_class()
    query.get("uniform")

    # クエリ取得中にエラーが発生したらエラーをresponseとして返す
    if 'error_num' in query.err:
        return json.dumps(query.err)
    
    try:
        # trials回, 乱数を生成
        for i in range(query.trials):
            if query.type == "int":
                rand.append(random.randint(query.min, query.max))
            else:
                rand.append(random.uniform(query.min, query.max))
    except ValueError:
        # パラメータが間違っていることによりエラーが発生した場合はエラーを返す
        jrand["error_num"] = 2
        jrand["error_message"] = "Error: Wrong parameter"
        return json.dumps(jrand)

    # JSON形式に変換して完了
    jrand["rand_array"] = rand
    return json.dumps(jrand)

# normal: 正規分布
@app.route('/random/normal', methods=["GET"])
def normal_index():
    rand = []
    jrand = {}
    # クエリパラメータの取得
    query = query_class()
    query.get("normal")

    # クエリ取得中にエラーが発生したらエラーをresponseとして返す
    if 'error_num' in query.err:
        return json.dumps(query.err)

    try:
        # trials回, 乱数を生成
        for i in range(query.trials):
            rand_temp = random.gauss(query.mu, query.sigma)
            if query.type == "int":
                rand_temp = int(rand_temp)
            rand.append(rand_temp)
    except ValueError:
        # パラメータが間違っていることによりエラーが発生した場合はエラーを返す
        jrand["error_num"] = 2
        jrand["error_message"] = "Error: Wrong parameter"
        return json.dumps(jrand)

    # JSON形式に変換して完了
    jrand["rand_array"] = rand
    return json.dumps(jrand)

# beta: ベータ分布
@app.route('/random/beta', methods=["GET"])
def beta_index():
    rand = []
    jrand = {}
    # クエリパラメータの取得
    query = query_class()
    query.get("beta")

    # クエリ取得中にエラーが発生したらエラーをresponseとして返す
    if 'error_num' in query.err:
        return json.dumps(query.err)

    try:
        # trials回, 乱数を生成
        for i in range(query.trials):
            rand_temp = random.betavariate(query.alpha, query.beta)
            if query.type == "int":
                rand_temp = int(rand_temp)
            rand.append(rand_temp)
    except ValueError:
        # パラメータが間違っていることによりエラーが発生した場合はエラーを返す
        jrand["error_num"] = 2
        jrand["error_message"] = "Error: Wrong parameter"
        return json.dumps(jrand)

    # JSON形式に変換して完了
    jrand["rand_array"] = rand
    return json.dumps(jrand)

# triangular: 三角分布
@app.route('/random/triangular', methods=["GET"])
def triangular_index():
    rand = []
    jrand = {}
    # クエリパラメータの取得
    query = query_class()
    query.get("triangular")

    # クエリ取得中にエラーが発生したらエラーをresponseとして返す
    if 'error_num' in query.err:
        return json.dumps(query.err)

    try:
        # trials回, 乱数を生成
        for i in range(query.trials):
            rand_temp = random.triangular(query.min, query.max, query.mode)
            if query.type == "int":
                rand = int(rand_temp)
            rand.append(rand_temp)
    except ValueError:
        # パラメータが間違っていることによりエラーが発生した場合はエラーを返す
        jrand["error_num"] = 2
        jrand["error_message"] = "Error: Wrong parameter"
        return json.dumps(jrand)

    # JSON形式に変換して完了
    jrand["rand_array"] = rand
    return json.dumps(jrand)

# lambda: ラムダ分布
@app.route('/random/lambda', methods=["GET"])
def lambda_index():
    rand = []
    jrand = {}
    # クエリパラメータの取得
    query = query_class()
    query.get("lambda")

    # クエリ取得中にエラーが発生したらエラーをresponseとして返す
    if 'error_num' in query.err:
        return json.dumps(query.err)

    try:
        # trials回, 乱数を生成
        for i in range(query.trials):
            rand_temp = random.expovariate(query.lambd)
            if query.type == "int":
                rand = int(rand_temp)
            rand.append(rand_temp)
    except ValueError:
        # パラメータが間違っていることによりエラーが発生した場合はエラーを返す
        jrand["error_num"] = 2
        jrand["error_message"] = "Error: Wrong parameter"
        return json.dumps(jrand)

    # JSON形式に変換して完了
    jrand["rand_array"] = rand
    return json.dumps(jrand)

# gamma: ガンマ分布
@app.route('/random/gamma', methods=["GET"])
def gamma_index():
    rand = []
    jrand = {}
    # クエリパラメータの取得
    query = query_class()
    query.get("gamma")

    # クエリ取得中にエラーが発生したらエラーをresponseとして返す
    if 'error_num' in query.err:
        return json.dumps(query.err)

    try:
        # trials回, 乱数を生成
        for i in range(query.trials):
            rand_temp = random.gammavariate(query.alpha, query.beta)
            if query.type == "int":
                rand = int(rand_temp)
            rand.append(rand_temp)
    except ValueError:
        # パラメータが間違っていることによりエラーが発生した場合はエラーを返す
        jrand["error_num"] = 2
        jrand["error_message"] = "Error: Wrong parameter"
        return json.dumps(jrand)

    # JSON形式に変換して完了
    jrand["rand_array"] = rand
    return json.dumps(jrand)
```

基本的には、query_classを生成し、パラメータを読み込み、ランダム関数にパラメータを渡して乱数を生成し、成功したら返す、という流れです。複数個の乱数を配列で返せるよう、JSON形式に変換してレスポンスを返します。失敗した場合は値域が指定の範囲になっていないことによるValueErrorが想定されますので、その旨のエラーを返します。  

出来上がった``app.py``の全貌がこちら。  

```python
# -*- coding: utf-8 -*-
import sys
from flask import Flask, request
from flask_cors import CORS
import random
import json

app = Flask(__name__)

# 別サーバ(=GitHub Pages)からのリクエストを許可
CORS(app, supports_credentials=True)

jrand = {}

# クエリのクラス
class query_class:
    # 変数初期化
    def __init__(self):
        self.min = 0
        self.max = 0
        self.mu = 0
        self.sigma = 0
        self.mode = 0
        self.lambd = 0.0
        self.alpha = 0.0
        self.beta = 0.0
        self.trials = 1
        self.type = ""
        self.err = {}

    # クエリパラメータの取得
    def get(self, distribution):
        try:
            # type: 整数 or 浮動小数点数
            arg_type = request.args.get("type")
            if self.type is not None:
                self.type = arg_type

            # trials: 試行回数（発生させる乱数の数）
            arg_trials = request.args.get("trials")
            if arg_trials is not None:
                self.trials = int(arg_trials)

            # 指定された確率分布関数が...
            # 一様分布 or 三角分布の場合  
            if distribution == "uniform" or distribution == "triangular":
                req_min = request.args.get("min")
                req_max = request.args.get("max")

                if self.type == "int":
                    if req_min is None:
                        self.min = -sys.maxsize
                    else:
                        self.min = float(req_min)
                    
                    if req_max is None:
                        self.max = sys.maxsize
                    else:
                        self.max = float(req_max)
                else:
                    if req_min is None:
                        self.min = sys.float_info.min
                    else:
                        self.min = float(req_min)
                    
                    if req_max is None:
                        self.max = sys.float_info.max
                    else:
                        self.max = float(req_max)

                if distribution == "triangular":
                    req_mode = request.args.get("mode")
                    if req_mode is None:
                        self.mode = (self.min+self.max)/2
                    else:
                        self.mode = int(req_mode)
                        
                return

            # 正規分布の場合
            elif distribution == "normal":
                req_mu = request.args.get("mu")
                req_sigma = request.args.get("sigma")

                if req_mu is None:
                    self.mu = 0
                else:
                    self.mu = float(req_mu)
                
                if req_sigma is None:
                    self.sigma = 1
                else:
                    self.sigma = float(req_sigma)
                return

            # ラムダ分布の場合
            elif distribution == "lambda":
                req_lambd = request.args.get("lambd")

                if req_lambd is None:
                    self.lambd = 1
                else:
                    self.lambd = 1/float(req_lambd)
                return

            # ベータ分布 or ガンマ分布の場合
            elif distribution == "beta" or distribution == "gamma":
                req_alpha = request.args.get("alpha")
                req_beta = request.args.get("beta")

                if req_alpha is None:
                    self.alpha = 1.0
                else:
                    self.alpha = float(req_alpha)
                
                if req_beta is None:
                    self.beta = 2.0
                else:
                    self.beta = float(req_beta)
                return

        except ValueError:
            self.err["error_num"] = 1
            self.err["error_message"] = "Error: Arguments are incorrect"
            return


# ルートディレクトリ：接続確認用
@app.route('/')
def root_index():
    return "Successfully accessed."

# uniform: 一様分布
@app.route('/random/uniform', methods=["GET"])
def uniform_index():
    rand = []
    jrand = {}
    # クエリパラメータの取得
    query = query_class()
    query.get("uniform")

    # クエリ取得中にエラーが発生したらエラーをresponseとして返す
    if 'error_num' in query.err:
        return json.dumps(query.err)
    
    try:
        # trials回, 乱数を生成
        for i in range(query.trials):
            if query.type == "int":
                rand.append(random.randint(query.min, query.max))
            else:
                rand.append(random.uniform(query.min, query.max))
    except ValueError:
        # パラメータが間違っていることによりエラーが発生した場合はエラーを返す
        jrand["error_num"] = 2
        jrand["error_message"] = "Error: Wrong parameter"
        return json.dumps(jrand)

    # JSON形式に変換して完了
    jrand["rand_array"] = rand
    return json.dumps(jrand)

# normal: 正規分布
@app.route('/random/normal', methods=["GET"])
def normal_index():
    rand = []
    jrand = {}
    # クエリパラメータの取得
    query = query_class()
    query.get("normal")

    # クエリ取得中にエラーが発生したらエラーをresponseとして返す
    if 'error_num' in query.err:
        return json.dumps(query.err)

    try:
        # trials回, 乱数を生成
        for i in range(query.trials):
            rand_temp = random.gauss(query.mu, query.sigma)
            if query.type == "int":
                rand_temp = int(rand_temp)
            rand.append(rand_temp)
    except ValueError:
        # パラメータが間違っていることによりエラーが発生した場合はエラーを返す
        jrand["error_num"] = 2
        jrand["error_message"] = "Error: Wrong parameter"
        return json.dumps(jrand)

    # JSON形式に変換して完了
    jrand["rand_array"] = rand
    return json.dumps(jrand)

# beta: ベータ分布
@app.route('/random/beta', methods=["GET"])
def beta_index():
    rand = []
    jrand = {}
    # クエリパラメータの取得
    query = query_class()
    query.get("beta")

    # クエリ取得中にエラーが発生したらエラーをresponseとして返す
    if 'error_num' in query.err:
        return json.dumps(query.err)

    try:
        # trials回, 乱数を生成
        for i in range(query.trials):
            rand_temp = random.betavariate(query.alpha, query.beta)
            if query.type == "int":
                rand_temp = int(rand_temp)
            rand.append(rand_temp)
    except ValueError:
        # パラメータが間違っていることによりエラーが発生した場合はエラーを返す
        jrand["error_num"] = 2
        jrand["error_message"] = "Error: Wrong parameter"
        return json.dumps(jrand)

    # JSON形式に変換して完了
    jrand["rand_array"] = rand
    return json.dumps(jrand)

# triangular: 三角分布
@app.route('/random/triangular', methods=["GET"])
def triangular_index():
    rand = []
    jrand = {}
    # クエリパラメータの取得
    query = query_class()
    query.get("triangular")

    # クエリ取得中にエラーが発生したらエラーをresponseとして返す
    if 'error_num' in query.err:
        return json.dumps(query.err)

    try:
        # trials回, 乱数を生成
        for i in range(query.trials):
            rand_temp = random.triangular(query.min, query.max, query.mode)
            if query.type == "int":
                rand = int(rand_temp)
            rand.append(rand_temp)
    except ValueError:
        # パラメータが間違っていることによりエラーが発生した場合はエラーを返す
        jrand["error_num"] = 2
        jrand["error_message"] = "Error: Wrong parameter"
        return json.dumps(jrand)

    # JSON形式に変換して完了
    jrand["rand_array"] = rand
    return json.dumps(jrand)

# lambda: ラムダ分布
@app.route('/random/lambda', methods=["GET"])
def lambda_index():
    rand = []
    jrand = {}
    # クエリパラメータの取得
    query = query_class()
    query.get("lambda")

    # クエリ取得中にエラーが発生したらエラーをresponseとして返す
    if 'error_num' in query.err:
        return json.dumps(query.err)

    try:
        # trials回, 乱数を生成
        for i in range(query.trials):
            rand_temp = random.expovariate(query.lambd)
            if query.type == "int":
                rand = int(rand_temp)
            rand.append(rand_temp)
    except ValueError:
        # パラメータが間違っていることによりエラーが発生した場合はエラーを返す
        jrand["error_num"] = 2
        jrand["error_message"] = "Error: Wrong parameter"
        return json.dumps(jrand)

    # JSON形式に変換して完了
    jrand["rand_array"] = rand
    return json.dumps(jrand)

# gamma: ガンマ分布
@app.route('/random/gamma', methods=["GET"])
def gamma_index():
    rand = []
    jrand = {}
    # クエリパラメータの取得
    query = query_class()
    query.get("gamma")

    # クエリ取得中にエラーが発生したらエラーをresponseとして返す
    if 'error_num' in query.err:
        return json.dumps(query.err)

    try:
        # trials回, 乱数を生成
        for i in range(query.trials):
            rand_temp = random.gammavariate(query.alpha, query.beta)
            if query.type == "int":
                rand = int(rand_temp)
            rand.append(rand_temp)
    except ValueError:
        # パラメータが間違っていることによりエラーが発生した場合はエラーを返す
        jrand["error_num"] = 2
        jrand["error_message"] = "Error: Wrong parameter"
        return json.dumps(jrand)

    # JSON形式に変換して完了
    jrand["rand_array"] = rand
    return json.dumps(jrand)

if __name__ == '__main__':
    app.run()
```

## 乱数APIの動作確認

まずはローカルで動作確認してみます。  

```zsh
% python3 app.py
 * Serving Flask app 'app' (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on http://127.0.0.1:5000 (Press CTRL+C to quit)
```

ブラウザでhttp://127.0.0.1:5000にアクセス。  
すると、  
![スクリーンショット 2022-06-10 11.17.32](../../../assets/img/post/2022-06-02-GitHub Pages + Heroku/スクリーンショット 2022-06-10 11.17.32.png)  
まずはルートディレクトリでメッセージの表示に成功。  

次に、一様分布で乱数を生成してみます。http://127.0.0.1:5000/random/uniform?min=0&max=100にアクセスし、  
![スクリーンショット 2022-06-14 16.35.05](../../../assets/img/post/2022-6-9-GitHub Pages + Heroku/スクリーンショット 2022-06-14 16.35.05.png)  
こちらも成功。更新のたびに異なる乱数値がJSON形式で返ってきています。  

正規分布も見てみます。http://127.0.0.1:5000/random/normal?mu=0.5&sigma=0.1&type=floatにアクセスし、平均値μ=0.5、標準偏差σ=0.1とし実数で取得。  
![スクリーンショット 2022-06-14 16.35.59](../../../assets/img/post/2022-6-9-GitHub Pages + Heroku/スクリーンショット 2022-06-14 16.35.59.png)

平均値μとなる0.5近辺で多くの乱数が発生。これも問題なし。  

次にベータ分布。http://127.0.0.1:5000/random/beta?alpha=0.5&beta=0.5&type=float&trials=10にアクセスし、α=β=0.5のときのバスタブ曲線に沿うような乱数を生成してみます。今度はtrials=10として10個の乱数を発生させてみます。  
![スクリーンショット 2022-06-14 16.38.10](../../../assets/img/post/2022-6-9-GitHub Pages + Heroku/スクリーンショット 2022-06-14 16.38.10.png)  
0付近と1付近の値が多く出現している様子が見受けられることから、バスタブ曲線の特徴がよく現れています。こちらも問題なし。  

一応、min=0, max=1.0, trials=1000とし、実行結果の分布を見てみます。  

まずは一様分布から。  
<img src="../../../assets/img/post/2022-6-9-GitHub Pages + Heroku/スクリーンショット 2022-06-14 16.59.36.png" alt="スクリーンショット 2022-06-14 16.59.36" style="zoom:50%;" />  
次にガウス分布（μ=0.5, σ=0.1）  
<img src="../../../assets/img/post/2022-6-9-GitHub Pages + Heroku/スクリーンショット 2022-06-14 17.01.09.png" alt="スクリーンショット 2022-06-14 17.01.09" style="zoom:50%;" />  

ベータ分布（α=β=0.5)  
<img src="../../../assets/img/post/2022-6-9-GitHub Pages + Heroku/スクリーンショット 2022-06-14 17.05.52.png" alt="スクリーンショット 2022-06-14 17.05.52" style="zoom:50%;" />  
三角分布（mode=0.8）  

<img src="../../../assets/img/post/2022-6-9-GitHub Pages + Heroku/スクリーンショット 2022-06-14 17.08.26.png" alt="スクリーンショット 2022-06-14 17.08.26" style="zoom:50%;" />    

ラムダ分布（lambd=0.1）

<img src="../../../assets/img/post/2022-6-9-GitHub Pages + Heroku/スクリーンショット 2022-06-14 17.10.19.png" alt="スクリーンショット 2022-06-14 17.10.19" style="zoom:50%;" />  

ガンマ分布（α=7.5, β=1.0）  

<img src="../../../assets/img/post/2022-6-9-GitHub Pages + Heroku/スクリーンショット 2022-06-14 17.12.03.png" alt="スクリーンショット 2022-06-14 17.12.03" style="zoom:50%;" />  

いずれも良さげです。これでHerokuへpushしましょう。  

```zsh
% git add .
% git commit -m "add random api"
% git push heroku master
```

先程と同様、push時に自動でデプロイが開始するので、完了したらHerokuでのAPI構築は完了です。

## 後編へつづく

前編ではHeroku側にバックエンドで動く乱数生成APIを構築しました。後編ではGitHub Pages側にフロントエンド部分（主にAPIに要求を出す部分とAPIから結果を受け取る部分）を作っていきます。
