---
title: "ISUCON12予選問題解き直し2"
date: 2022-09-04T22:28:00+09:00
categories:
    - tech
tags:
    - ISUCON
    - Python
---

[前回の記事](http://localhost:1313/post/2022-08-27-15/)の続きです。

[ISUCON12 予選の解説 (Node.jsでSQLiteのまま10万点行く方法) : ISUCON公式Blog](https://isucon.net/archives/56842718.html)を参考にPythonで解き直していました。アプリケーションサーバを分ける手前まで改善したのですがmax6500点までしかいかず、分けても10万点どころか予選突破相当の24000点に届くかさえ怪しかったので頓挫しました。

追加で実施できたもの

* tenantDB player_scoreにINDEXをはる
* Ranking APIでランキング集計するのをやめる

自分で追加で行ったこと

* scoreエンドポイントのトランザクション見直し
* Finish APIでBillingReportを生成する の修正
* lockによるエラーが多発したので一旦timeoutを伸ばす
* players/addの改善

実施しなかったもの
* AddTenant APIでSQLite DBを作るのをやめる
* nginxで複数台に振り分ける
* nginxをupstream keepaliveする
* MySQLをちょっとチューニングする

## scoreエンドポイントのトランザクション見直し

整合性チェック時に3回に1回くらいの頻度でエラーが発生していたので修正に着手しました。
AUTOCOMMITの設定がちゃんと効いていなかった模様。sqlalchemyはデフォルトでautocommitが効いており、scoreの時だけ設定を上書きするようにしました。

[コミットログ](https://github.com/reiichii/isucon12q-after/commit/91045ac410fa11ce0fbf7b6bfabf3b08bfe9a3f1)

エラー解消が目的だったのでスコアに影響はありませんでした。

参考

* [Transactions and Connection Management — SQLAlchemy 1.4 Documentation](https://docs.sqlalchemy.org/en/14/orm/session_transaction.html#setting-isolation-for-individual-transactions)
* [SQLAlchemyのautocommitについて - Qiita](https://qiita.com/tosizo/items/7a3e2d5b6f2f34867274)

## Finish APIでBillingReportを生成する の修正

整合性チェックは通るのですがベンチマーク全体の中で1~3回ほど `GET /api/organizer/billing 請求レポートの数が違います (want: 5, got: 1)のようなエラーが出る。` のようなエラーが出る状態でした。

終わっていない大会の情報も出してあげる必要があったのですが、それらの情報がDBには存在していないのが原因でした。存在しなければscore等を0を入れてレスポンスデータを生成します。

[コミットログ](https://github.com/reiichii/isucon12q-after/commit/9a515f9acb6249dabdc8f1752bbc2f4a56517e5c)

上記二つを行なってエラーもなくなり、スコアが安定するようになりました。ただし負荷走行中にSQLite3でlockエラーが多発するようになりました。

## tenantDB player_scoreにINDEXをはる

初期化時にinitial_dataをtenant_db配下にコピーしているのでinitial_dataのテーブルに対してINDEXを追加します。
テナントごとにdbがあるのでシェルでまとめて適用してあげます（ブログに書いてあったコマンドをそのまま実行しました）

クエリ：`create index idx_score on player_score (tenant_id, competition_id, player_id);`

`for db in *.db; do echo "CREATE INDEX..." | sqlite3 $db; done`

ちなみにplayer_score以外のテーブルはデータ量が100件程度しかなく、貼っても意味なさそうなのでそのままにしました。
SQLite3の実行計画は クエリの頭に`EXPLAIN QUERY PLAN` を付けます。

```
# player/<player_id>時
EXPLAIN QUERY PLAN SELECT c.title AS title, p.score AS score
FROM player_score AS p
INNER JOIN competition AS c ON c.id = p.competition_id
WHERE c.tenant_id = ?
AND p.player_id = ?
ORDER BY c.created_at ASC

# 結果
|--SCAN p
|--SEARCH c USING INDEX sqlite_autoindex_competition_1 (id=?)
`--USE TEMP B-TREE FOR ORDER BY
```

点数は500点ほど上がったのですが、それ以上にDBのlockによるエラーがひどく、41%失点している有様でした。

## lockによるエラーが多発したので一旦timeoutを伸ばす

タイムアウトを伸ばすしか思い浮かばなかったのでデフォルト値を調べてみることにしました。

ソースコードを見た感じPythonのSQLite3の標準ライブラリの設定がそのまま反映されているようでそれが5sでした。
30sに設定してみたところlockによる500エラーは大幅に減らせました。ただしclient側でconnection timeoutが発生しているのですがひとまず1件程度まで抑えられたので一旦よしとしました。

[コミットログ](https://github.com/reiichii/isucon12q-after/commit/74691d3703b159842f71a2c409156028e87b142b)

## Ranking APIでランキング集計するのをやめる

> ranking APIの呼び出される回数とscoreが入稿される回数は10～20倍くらい差がある
> rankingはscoreを入稿したときしか変わらない

言われてみれば確かに。

大会中にこのボトルネックに気づいていたらまず間違いなくDELETE+bulk insertで対処していたと思うのですが、 `ON DUPLICATE KEY UPDATE` を初めて知ったのでこっちで実装してみることにしました。

* ON DUPLICATE KEY UPDATE
  * ON DUPLICATE KEY UPDATE を指定した時、UNIQUEインデックスまたは PRIMARY KEY
 に重複した値を発生させる行が挿入された場合、mysqlによって古い行の値が実行される
  * 存在していればupdate する

やることとしては以下です。

* rankingテーブルを作成する
    ```sql
    CREATE TABLE ranking (
    `tenant_id` BIGINT UNSIGNED NOT NULL,
    `competition_id` VARCHAR(255) NOT NULL,
    `rank` INT NOT NULL,
    `score` BIGINT NOT NULL,
    `player_id` VARCHAR(255) NOT NULL,
    `player_display_name` TEXT NOT NULL,
    PRIMARY KEY (`tenant_id`, `competition_id`, `rank`)
    ) ENGINE=InnoDB DEFAULT CHARACTER SET=utf8mb4;
    ```
  * row_numは不要だから消したと思われる。competition_idさえ分かればtenant_idはなくても良さそうに思える
* scoreエンドポイントでrankingを生成し、insertする
* 初期化対応
  * が必要とのことでしたが、データを入れ直さなくてもベンチマークが通ったのでしませんでした。データが溜まっていってしまうのを防ぐために削除だけ行うように修正しました。

[コミットログ](https://github.com/reiichii/isucon12q-after/commit/2be866a002c2e71dbc2c8b94367c8b3f34b7ed4a)

ベンチマークを何度か実行していたのですが6500~5600と振り幅が大きい...。

## players/addの改善

alpの結果を眺めていたら上記エンドポイントが異常に重たくなっていました。スコアログを見返すとflockをトランザクションにしたあたりからずっとひどい状態でした笑

スコアが伸び悩んでいたのもあり、気になったので改善してみようとコードを読んだら、こちらもfor文の中で逐一クエリが発行されていました。sqliteの負荷が懸念だったのもあり以下のようにそれぞれまとめて取得してPython側で頑張るように修正しました。

[コミットログ](https://github.com/reiichii/isucon12q-after/commit/3016f0fe8a3198163c6153168ae0159a892990da)

alpを見た感じ改修の効果は得られた(25s→2sになった)のですが、点数には影響せず...。

## おわりに

10万はいかなくとも2万くらいはいきたいなと思っていたのですが、今のまま複数台分散してもそこまで上がる見込みがなく、だれてきてしまったのもあり一旦一区切りにしようと思います😓

全体の改善のログは以下に。

[スコア推移のログ · Issue #1 · reiichii/isucon12q-after](https://github.com/reiichii/isucon12q-after/issues/1)

ISUCON11予選問題解説のやり方を参考に残していました。

