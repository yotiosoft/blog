---
layout: post
title: Static Formsでコンタクトフォームを作成した
tags: [お知らせ,ホームページ]
excerpt_separator: <!--more-->
---

現在テスト運用中の[新しいホームページ](https://yotiosoft.github.io/support/){:target="_blank"}にコンタクトフォームを設置しました。  

コンタクトフォームを含む送信フォームをhtmlで作成する場合、一般的には指定したメールアドレスに転送するためにphpを利用しますが、Github Pagesでは動的サイトを作成できないためphpをサポートしていません。  
そこで今回はStatic Formsと呼ばれる、phpが不要でhtml（とcss）だけで送信フォームの実装を可能にするAPIサービスを利用しました。  
<!--more-->

Static（静的な）Forms（フォーム）ということで、まさに静的サイトのためにあるようなものです。  
JQueryで実現する方法もありますが、こっちのほうが手っ取り早いです。

# 利用方法

Static Formsのホームページ（[https://www.staticforms.xyz/](https://www.staticforms.xyz/){:target="_blank"}）を見れば大体わかります。  
一応まとめておくと、  

1. メールアドレスを登録してAccess Keyを取得する。
2. ホームページに掲載されているサンプルコードをコピペする。
3. サンプルコードのうちAccess Keyを取得したものに、送信先メールアドレスを自分のアドレスに置き換える。
4. 好みに合わせてコンタクトフォームを調整する。不要な項目は削除する。  
   

といった手順で利用できます。

# 実装例

実装例というか、今回実装した例です。  
**index.html**

```html
<!--コンタクトフォーム-->
<div class="container">
  <div class="columns">
    <div class="column is-half">
      <form action="https://api.staticforms.xyz/submit" method="post" id="staticform">
        <input type="hidden" name="accessKey" value="＜ここをAccess Keyに置き換え＞" required>
        <div class="form-element">
          <label class="label">お名前</label>
          <input type="text" name="name" required>
        </div>
        <div class="form-element">
          <label class="label">E-mail</label>
          <input type="text" name="email" required>
        </div>
        <div class="form-element">
          <label class="label">タイトル</label>
          <input type="text" name="subject" required>
        </div>
        <div class="form-element">
          <label class="label">内容</label>
          <textarea name="message" required></textarea>
        </div>
        <input type="hidden" name="replyTo" value="＜ここを送信先メールアドレスに置き換え＞">
        <input type="hidden" name="redirectTo" value="＜ここを遷移先のページのURLに置き換え＞">
        <input type="submit" value="送信" style="width: 50px;" />
      </form>
    </div>
  </div>
</div>
```

**contact-form.css**

```css
.column.is-half {
    width: 50%;
}

.form-element {
    margin-top: 10px;
    margin-bottom: 10px;
}

.form-element input {
    width: 50%;
}

.form-element textarea {
    width: 100%;
    height: 100px;
}

.form-element label {
    display: flex;
}
```

入力内容としては名前、送信者のメールアドレス、タイトル、お問い合わせ内容です。  
実装の際に置き換えるべき点はAccess Key、送信先メールアドレス、遷移先URLの3点です。  

ここで気をつけなければならないのは、遷移先のURLは相対パス（./thanks.htmlなど）ではなく絶対パス（すなわち、http://やhttps://から始まるURL全体）で示さなければならないという点です。  
一旦StaticFormsのAPIサーバーに遷移するので、相対パスで指定すると、ブラウザはStaticForms上のURLとして解釈してしまいます。  

出来上がったのが送信フォームがこちらです。  
![スクリーンショット 2021-09-13 16.21.28](../../../assets/img/post/スクリーンショット 2021-09-13 16.21.28.png) 



## 送信エラーの取得

Static Formsでは、送信ボタン（submit）を押した後、送信エラーがあった場合はjsonでその内容を知らせてくれます。  
そのjsonを取得し、パースしてくれるプログラムもホームページで公開されています。  
「Examples on JSFiddle」の「JQuery Example」をご覧ください。



# 動作確認

以下の内容で送信してみます。  
![スクリーンショット 2021-09-13 16.01.51](../../../assets/img/post/スクリーンショット 2021-09-13 16.01.51.png)  

送信ボタンを押すと、予め作成しておいたthanks.htmlに遷移します。  
![スクリーンショット 2021-09-13 16.02.07](../../../assets/img/post/スクリーンショット 2021-09-13 16.02.07.png)  

するとStatic Formsのサーバーを経由し、指定したメールアドレスにメールが送信されます。  
![スクリーンショット 2021-09-13 15.55.50](../../../assets/img/post/スクリーンショット 2021-09-13 15.55.50.png)  
送信元はStatic Formsになっていますので、Static Fromsのサーバーが転送しているようです。  
![スクリーンショット 2021-09-13 16.02.29](../../../assets/img/post/スクリーンショット 2021-09-13 16.02.29.png)  

テスト成功です。どうやらうまく動いているようですね。
