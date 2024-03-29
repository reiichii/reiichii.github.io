---
title: "ISUCON12事前講習"
date: 2022-06-07T20:56:13+09:00
draft: false
categories:
    - tech
tags:
    - ISUCON
    - 参加レポ
---

ISUCONハンズオン目的で申し込んだのですが、去年一昨年の事前講習レポートには書いていない内容が盛り込まれていて普通に楽しかったです。

[ISUCON12 事前講習 - Speaker Deck](https://speakerdeck.com/rosylilly/isucon12-shi-qian-jiang-xi)

ほぼ資料に書いてあるのですが、記念に手元のメモも残します。

```
## 強いチームがしていること
* なんとなくで手を動かさない。
  - 優勝者インタビューで「何が効いたのかわからない」というチームはいない
* デプロイのリードタイムをに1分以上かけない
  - GUIでgit操作しがちなご時世だけど、gitコマンドを使った方がいいよ
* 使い慣れたミドルウェアのconfigを1から書かない
  - 事前に用意しておく
* やったことがないことをやらない
  - 大会中に実務で触っていないgoに移植しようとしてボロ負けした経験がある
```

なんとなく手を動かすな、仮説をベースに動くことはISUCON以外の仕事でも言える。

```
## タイムライン
10:00
* マニュアルとレギュレーションを読む
* ブラウザでサービスを見て、アプリケーションを把握する
* 各コンポートネントがどう起動されているか、設定やconfigの場所を確認
  - init.dかsystemcnfかdockerかなど
* 自分が必要なruntimeをさっとインストールできるようにしておく
* dbスキーマがどう定義されているか調べる
* デプロイ方法を構築する
* 使われているミドルウェアの種類とバージョンを調べる
  - 過去にmemcacheかと思ったらmemcacheのplaginを入れたmysqlでそれがすごく重い、という罠があったらしい
* 使っているサーバのスペックを各台調査する
  - サーバによってスペックが異なるケースがある
* ベンチマークを実行する
11:00
* 得点源が何かを確認する
* 減点の要因を把握する
* プロファイリングツールを入れる
* 初期状態の完全なバックアップを作成する
  - tarで固めておく
12:00
* ちゃんとご飯を食べる
* わからないことが出たらリストにしておく
* やること、やらないことを明確にする
13:00
* デプロイが1コマンドでできるように
* デプロイ→性能計測→プロファイルまで一気通貫で行える仕組みを用意しておく
  - line_profile
  - リクエスト単位　どちらも
14:00~17:00
* 1コミット1ベンチマーク
* 気にする指標を明確に把握してプロファイルする
17:00
* 再起動試験をする
* apmを入れていたら停止する
  - newrelicのapm止めるの意外と難しかったりする
* デバックログの出力を止める
* プロファイル用に差し込んだものを止める
18:00
* 作業ログをブログに書く準備をする
* 記憶が明確な間に振り返りをする
```

優勝経験チームの行動をトレスしたタイムラインは、考え方など参考にできるところが多い貴重な資料です。

前にISUCON11予選過去問に挑戦した時、私の場合マニュアルとレギュレーションを読むだけで1時間はかかったので10:00代きっっっつて思いながら聞いていました🙂

```
### おすすめの練習
* デプロイ方法セットアップ
  - リポジトリ作って、git initして、チェックインして、deploy
* ansibleを最速で回せるようになっておく
* ベンチマークから集計を1コマンドでできるようにする
  - 集計スクリプトを作っておく
* サーバの役割変更
  - 起動を止める(systemctlならdisableし忘れない)、接続先を変更する
* 使いたいツールのインストール
  - 使いたいツールは一発で入れられるようにする(alp,pt-query-digest)
  - prebuilt binaryが用意できるなら用意するのも手
```

もし今年出れるなら、最低限これだけは準備していきたい。

また最後に同じ問題5回くらい解くと、新しい発見があって楽しいよといったこともあって、また過去問解き直そうと思いました。

ちなみにまだ参加申し込みできていませんorz