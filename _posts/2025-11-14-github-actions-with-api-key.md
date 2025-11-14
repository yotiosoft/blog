---
layout: post
title: "GitHub Actionsで使う環境変数をログから隠したい"
tags: [GitHub]
excerpt_separator: <!--more-->
---

最近は GitHub Actions を使って、GitHub に commit するたびにビルドからテストまで自動で行うようにしています。

しかし、外部 API の API キーを扱うようなテストの場合、ログやテストコードから API キーが流出することは避けたいですよね。あるいは、パスワードがコードやログから流出することも避けたいです。今回は、GitHub Environments を使って、これらの機密情報が外部に漏れないようにしていきます。

<!--more-->	

# GitHub Environments とは

GitHub Environments は GitHub Actions で利用できる環境です。GitHub Actions が実行されるコンテナ環境に対して適用されます。

GitHub Environments は環境変数を定義することが可能です。GitHub Actions で定義したワークフローは、通常の OS の仮想環境と同じようにこの環境変数にアクセスできます。もちろん環境変数なので、変数の内容がコードから漏れることはありません。

更に嬉しいことに、GitHub Environments にはシークレット変数（Environment secrets）というものがあり、ここに機密情報を登録しておくと、GitHub Actions の実行ログからこんな感じでシークレット変数の内容を隠してくれます。

```bash
INFO:     127.0.0.1:39298 - "GET /free/v2/languages?type=source HTTP/1.1" 200 OK
Received request: auth_key=DeepL-Auth-Key ***
Type parameter: source
```

これにより、うっかり環境変数がコンソールに出力されるツールを実行しても、ログから API キーなどの機密情報が漏れることはありません。

# 手順

## Environments を登録する

まずは GitHub リポジトリの Settings を開きます。

![image-20251114221713330](../../../assets/img/post/2025-05-30-github-actions-with-api-key/image-20251114221713330.webp)

続いて、左側メニューの Code and automation から Environments を選択。

![image-20251114221805235](../../../assets/img/post/2025-05-30-github-actions-with-api-key/image-20251114221805235.webp)

右上の New environment をクリックします。

![image-20251114221841420](../../../assets/img/post/2025-05-30-github-actions-with-api-key/image-20251114221841420.webp)

任意の Environment 名（※環境変数名ではない）を設定し、Configure environment をクリック。

![image-20251114222008250](../../../assets/img/post/2025-05-30-github-actions-with-api-key/image-20251114222008250.webp)

すると、Environment の設定画面が表示されます。

![image-20251114222052272](../../../assets/img/post/2025-05-30-github-actions-with-api-key/image-20251114222052272.webp)

通常の環境変数であれば Environment variables に、ログから隠蔽したい機密情報であれば Environment secrets に変数を登録していきます。今回はシークレット変数を登録したいので、Add environment secret をクリックしましょう。

![image-20251114222215065](../../../assets/img/post/2025-05-30-github-actions-with-api-key/image-20251114222215065.webp)

するとこんなウィンドウが出てきます。ここにシークレット変数名と値を入力します。

![image-20251114222344024](../../../assets/img/post/2025-05-30-github-actions-with-api-key/image-20251114222344024.webp)

入力が終わったら Add secret をクリックします。

![image-20251114222410639](../../../assets/img/post/2025-05-30-github-actions-with-api-key/image-20251114222410639.webp)

これにてシークレット変数が登録されました。

## 作成した Environment で GitHub Actions を実行する

GitHub Actions から New workflow をクリックします。

![image-20251114222654032](../../../assets/img/post/2025-05-30-github-actions-with-api-key/image-20251114222654032.webp)

するとワークフローのテンプレート選択画面が出てきます。今回は自分で1から作るために set up a workflow yourself を例として選択しています。

![image-20251114222757291](../../../assets/img/post/2025-05-30-github-actions-with-api-key/image-20251114222757291.webp)

ワークフローはこんな感じで登録しました。

```yaml
name: Test

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build:

    runs-on: ubuntu-latest

    environment: test_api_key
    env:
      TEST_API_KEY: ${{{{ secrets.TEST_API_KEY }}}}

    steps:
    - uses: actions/checkout@v4
    - name: Print secret
      run: echo $TEST_API_KEY
```

GitHub Actions で Environment を利用するには、`environment:` で先程設定した Environment 名を指定する必要があります。また、環境変数としてコンテナ環境で利用可能にするために、`env:` にて各シークレット変数を環境変数として宣言しておきます（名前は必ずしも一致している必要はありません）。

今回は簡単な例として、先程登録した機密情報である環境変数を `echo $TEST_API_KEY` で表示させてみます。

この状態で `main` ブランチに何らかの変更をコミットして `git push` してみます。すると…

![image-20251114224211871](../../../assets/img/post/2025-05-30-github-actions-with-api-key/image-20251114224211871.webp)

シークレット変数の部分だけ伏せ字になって表示されました。このように、GitHub Actions のログからはシークレット変数の内容は隠蔽されます。

# おわりに

今回は明示的に環境変数にてシークレット変数をコンソールに出力させましたが、環境変数を介さずに同様の文字列が出力された場合でもログからは隠蔽されます。ですので、例えば自作のプログラムにデバッグとして API キーを表示するコードをうっかり消し忘れていたり、テストで使いたいツールがうっかり API キーやパスワードをコンソールに出力しちゃったようなケースでも、Environment secrets に登録しておけば流出を防ぐことが可能です。
