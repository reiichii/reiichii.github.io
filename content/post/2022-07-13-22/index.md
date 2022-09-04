---
title: "ISUCON11-qualifyのログインページが開かなかった"
date: 2022-07-13T22:21:31+09:00
draft: false
categories:
    - tech
tags:
    - ISUCON
---

ISUCON11予選環境構築時、構築したアプリケーションでログインしようとすると「このサイトにアクセスできません」が表示されます。また遷移先urlが「`http://localhost:5000/?callback=https://isucondition.t.isucon.dev`」のようにおかしな表示になります。

前提として以下の手順を参考に、クラウド環境にアプリケーションを構築し、トップページが開けるところまでを確認済みです。

[ISUCON過去問題の環境を「さくらのクラウド」で構築する | さくらのナレッジ](https://knowledge.sakura.ad.jp/31520/)

## やること1. JIA API Mockを起動する

[アプリケーションマニュアル](https://github.com/isucon/isucon11-qualify/blob/main/docs/isucondition.md#jia-api-mock-%E3%81%AB%E3%81%A4%E3%81%84%E3%81%A6)の末尾に書いてあるのですが、サーバの5000portで一部のリクエストを待ち受けるようになっているみたいです。

実際urlからも分かる通り、apiのログイン時に5000portに飛ばすようになっています。該当コードは以下です。

https://github.com/isucon/isucon11-qualify/blob/main/webapp/frontend/src/components/Home/Auth.tsx#L6

自動起動はしないため、マニュアルに書いてある手順でモックのサービスを起動してあげます。

## やること2. ポートフォワーディングの設定

このままだとアプリケーションした際にローカル環境の「localhost:5000」にアクセスされてしまいます。

ローカル環境の5000にアクセスされたら、リモートサーバの5000にアクセスされるようにポートフォワーディングの設定をしておきます。

`ssh -A -L 5000:{ip}:5000 {user}@{ip}`

ssh接続した状態で「JIAのアカウントでログイン」を押すと、「Sign in with JIA」の画面が開き、ユーザー名とパスワードを入力してログイン後の画面にすすめるようになります👏

## おわりに

分かる人には分かるのかもしれませんが、これは構築手順書に説明があった方が親切なような気がしました。

ちなみにこの辺の仕様について話されているissueも発見しました。完全に理解はしていません..。

https://github.com/isucon/isucon11-qualify/issues/1260
