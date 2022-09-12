---
layout: post
title: "Stable DiffusionをGoogle Colab上で遊ぶ"
tags: [Stable Diffusion, Google Colab, Ubuntu]
excerpt_separator: <!--more-->
---

世はまさに大AI時代。「Stable Diffusion」が先月23日に登場して以来、目まぐるしいほどに便利で画期的な新規サービスが登場していますが、今回、Google Colaboratory上で少し動かして遊んでみたので、その記録をしておきます。  
なお、この記事を書くのが激遅になってしまったため、実際にStable Diffusionを動かしたのは先月末頃で、今はもっとGUIで簡単に触れるサービスが存在します。

<!--more-->

# 参考にさせていただいたサイト

- [Stable Diffusionのサンプルコード(text2img/img2img)をGoogle Colabで動かす方法](https://zenn.dev/karaage0703/articles/22ee47b71fab9c){:target="_blank"}

- [CompVis/stable-diffusion](https://github.com/CompVis/stable-diffusion){:target="_blank"}

# 実験環境

- Google Colaboratory ([https://colab.research.google.com/](https://colab.research.google.com/){:target="_blank"})
  - OS: Ubuntu 18.04.6
  - CPU: intel Xeron CPU @ 2.00GHz
  - GPU: Nvidia Tesla T4
  - Python: 3.7.13
  - Cuda: version 11.2
  

# 手順

## 1. Hugging Faceのアカウントを作成＆トークンを取得

学習データをダウンロードするため、まずはHugging Faceのホームページを開き、アカウントを作成します。  
[https://huggingface.co](https://huggingface.co){:target="_blank"}  

このアカウントは後ほど使います。  

## 2. Google Colaboratoryでノートブックを作成

[Google Colaboratory](https://colab.research.google.com/?hl=ja){:target="_blank"}を開き、新しいノートブックを作成します。

![スクリーンショット 2022-09-01 3.11.21](../../../assets/img/post/2022-8-27/スクリーンショット 2022-09-01 3.11.21.png)  

作成されたら自動で開かれるので、ノートブックにコードを記述していきます。  
**ノートブックの内容は[こんな感じ](https://colab.research.google.com/drive/1qFk6qC963Fxkdja5WZ-Op2vpIl7t0vS1?usp=sharing){:target="_blank"}にしておきました。**以後、このノートブック上でコードを実行していきますので、適宜自分のドライブにコピーしておいてください。

## 3. 環境の導入

上から順に実行ボタンをポチポチしていきます。  

ここで注意しなければならないのが、途中でHugging Faceのtokenが必要になるところ。  
Hugging Faceの画面右上のアイコンをクリックし、Settingsを開きます。  
![スクリーンショット 2022-08-27 15.56.14](../../../assets/img/post/2022-8-27/スクリーンショット 2022-08-27 15.56.14.png)  

Tokenを開き、生成されたTokenをコピーしてGoogle Colabの実行画面に貼り付けます。  
![スクリーンショット 2022-08-27 15.58.05](../../../assets/img/post/2022-8-27/スクリーンショット 2022-08-27 15.58.05.png)

## 4. いざ実行

環境の導入が完了したら、ついに画像の生成。

### 4.1 txt2img

まずはtxt2img、すなわち「文章から画像を生成」を試してみます。  
生成画像はStable Diffusionのディレクトリ配下の``outputs/txt2img-samples``に出力されますので、Google Drive上に作業用ディレクトリを作っておけば自動的にGoogle Driveに保存されます。 

```bash
!python scripts/txt2img.py --prompt "City of the future, cars flying in the sky." --plms
```

このコードの左側の実行ボタンを押すと、「City of the future, cars flying in the sky（車が空を飛んでいる未来都市）」の画像が生成されます。画像生成に2分程かかります。  
 

で、生成された画像がこちら。  
![grid-0011](../../../assets/img/post/2022-8-27/grid-0011.png)

おー、いい感じ。  

他にも試してみる。  

#### The vast plain seen from above, with one large river flowing through it.

![f7ae4770-1fad-4d17-a323-fe0d487a84f9](../../../assets/img/post/2022-8-27/f7ae4770-1fad-4d17-a323-fe0d487a84f9.png)

#### European townscape, with trams running.

![grid-0013](../../../assets/img/post/2022-8-27/grid-0013.png)

#### Japanese landscape, Ukiyoe style

![grid-0014](../../../assets/img/post/2022-8-27/grid-0014.png)

### 4.2 img2img

次にimg2img、すなわち「画像から画像を生成」を試してみます。ここでは、入力画像に対し、画風などを指示して別の画像にアレンジすることができます。  
とりあえず、付属のsample.jpgを変換。と、その前に、元のサイズのままで進めるとout of memoryになってしまうので、512x256に縮小してsample_small.jpgとして保存しておき、縮小後の画像に対して変換を施します。生成画像は``outputs/img2img-samples``に出力されます。  

元画像がこちら。  
![ダウンロード](../../../assets/img/post/2022-8-27/sample_small.jpg)  

これに対し、"anime"と指定して変換。  

```bash
!python scripts/img2img.py --prompt "anime" --init-img sample_small.jpg --strength 0.8
```

で、変換結果がこちら。  
![d89c8b9f-bd09-47de-ae7b-ff83904cd718](../../../assets/img/post/2022-8-27/d89c8b9f-bd09-47de-ae7b-ff83904cd718.png)  
![d231e331-e9dd-410d-8668-0251f9a4a40d](../../../assets/img/post/2022-8-27/d231e331-e9dd-410d-8668-0251f9a4a40d.png)  

なんか思ってたんと違う…けど、一応アニメキャラクターみたいなのは出てきた。  
animeだけでは抽象的すぎた？  

他にも試してみる。  

#### Magnificent views in oil painting style

![68fd9aed-3261-49fa-82d5-3b64d48388b8](../../../assets/img/post/2022-8-27/68fd9aed-3261-49fa-82d5-3b64d48388b8.png)  
![46443864-e9e5-46ee-b25e-02eed9232f94](../../../assets/img/post/2022-8-27/46443864-e9e5-46ee-b25e-02eed9232f94.png)  

#### Photo, Europe

![d9480a23-a2aa-4609-bb12-5f288fd906b1](../../../assets/img/post/2022-8-27/d9480a23-a2aa-4609-bb12-5f288fd906b1.png)  
![a125f5bd-d15c-49d6-9836-5103f7814296](../../../assets/img/post/2022-8-27/a125f5bd-d15c-49d6-9836-5103f7814296.png)

#### Simcity

![35b3829a-1d60-4a70-9958-8533b08a3165](../../../assets/img/post/2022-8-27/35b3829a-1d60-4a70-9958-8533b08a3165.png)  
![a82cc2cb-aff1-456c-b066-48a7d8cdd080](../../../assets/img/post/2022-8-27/a82cc2cb-aff1-456c-b066-48a7d8cdd080.png)

# おわりに

今回はちょっとしか触っていませんが、無限に遊べそうです。呪文は長く、より具体的に書いたほうが意図した絵が出来やすいので、今後も触ってみて何か良さげな絵が出力されたら記事に書こうかと思います。せっかくオープンソースなのだから、これを利用したサービスも作りたいな。屈強なGPUを備えたサーバーが必要だけど…
