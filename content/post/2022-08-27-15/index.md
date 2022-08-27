---
title: "isucon12予選問題解き直し"
date: 2022-08-27T14:57:37+09:00
draft: false
categories:
    - tech
tags:
    - ISUCON
    - python
---

8月は[ISUCON12 予選の解説 (Node.jsでSQLiteのまま10万点行く方法) | ISUCON公式Blog](https://isucon.net/archives/56842718.html)を見ながらISUCON12予選問題の解き直しをしていました。まだ全部施策をやり切れておらず、点数も上がりきってはいないのですが、1ヶ月経ったので途中までまとめることに。

実施できたもの

* adminDB visit_history にINDEXを張る
* dispenseIDでMySQLを使うのをやめる
* Ranking APIのループクエリをなくす
* Score APIの追加のループクエリをなくす
* アトミック書き込みのためのflockをトランザクションに変更する(※怪しい)
* adminDB visit_historyの初期データをコンパクトにする
* db用サーバを投入し、2台構成にする
* Finish APIでBillingReportを生成する(※怪しい)
* Player APIをなんとかする

まだできていないもの

* tenantDB player_scoreにINDEXをはる
* Ranking APIでランキング集計するのをやめる
* AddTenant APIでSQLite DBを作るのをやめる
* nginxで複数台に振り分ける
* nginxをupstream keepaliveする
* MySQLをちょっとチューニングする

半分以上は実施しているのに未だ点数が6000点代という...思ったより厳しかった。

## adminDB visit_history にINDEXを張る

去年の問題ならinitialエンドポイントテーブルが作り直しているのでschemaにindexを追加していたのですが、今回は対象テーブルではdrop createは実行されていないのでここに書いても意味なかったという🙂

covering indexという概念を初めて知りました。indexって貼れていればいいと思っていたのですが、張り方によっても性能(点数)に差が出てしまうんですね。せっかくなので3パターンで実行計画を比較してみました。

```
# 既存
EXPLAIN SELECT player_id, MIN(created_at) AS min_created_at FROM visit_history WHERE tenant_id = 1 AND competition_id = 'S' GROUP BY player_id;
+----+-------------+---------------+------------+------+---------------+---------------+---------+-------+---------+----------+------------------------------+
| id | select_type | table         | partitions | type | possible_keys | key           | key_len | ref   | rows    | filtered | Extra                        |
+----+-------------+---------------+------------+------+---------------+---------------+---------+-------+---------+----------+------------------------------+
|  1 | SIMPLE      | visit_history | NULL       | ref  | tenant_id_idx | tenant_id_idx | 8       | const | 1292937 |    10.00 | Using where; Using temporary |
+----+-------------+---------------+------------+------+---------------+---------------+---------+-------+---------+----------+------------------------------+

# covering index
+----+-------------+---------------+------------+------+-----------------------------+---------------+---------+-------------+------+----------+-------------+
| id | select_type | table         | partitions | type | possible_keys               | key           | key_len | ref         | rows | filtered | Extra       |
+----+-------------+---------------+------------+------+-----------------------------+---------------+---------+-------------+------+----------+-------------+
|  1 | SIMPLE      | visit_history | NULL       | ref  | tenant_id_idx,idx_all_cover | idx_all_cover | 1030    | const,const |    1 |   100.00 | Using index |
+----+-------------+---------------+------------+------+-----------------------------+---------------+---------+-------------+------+----------+-------------+

# createdなし
+----+-------------+---------------+------------+------+-----------------------------+---------------+---------+-------------+------+----------+-------+
| id | select_type | table         | partitions | type | possible_keys               | key           | key_len | ref         | rows | filtered | Extra |
+----+-------------+---------------+------------+------+-----------------------------+---------------+---------+-------------+------+----------+-------+
|  1 | SIMPLE      | visit_history | NULL       | ref  | tenant_id_idx,idx_all_cover | idx_all_cover | 1030    | const,const |    1 |   100.00 | NULL  |
+----+-------------+---------------+------------+------+-----------------------------+---------------+---------+-------------+------+----------+-------+
```

* indexを追加すると、possible_keys,keyにidx_all_coverが追加され、filteredが100%になる
* covering indexにすると、ExtraにUsing indexが表示される
* createdありとなしではスコアには200点ほど差がでた

mysqlのconvering indexとは

* クエリーによって取得されたすべてのカラムを含む***インデックス***
* 検索を索引内で完結でき、表からデータを読み取る必要がないため効率が良い
* 表のサイズがメモリに保持しきれないほど大きい場合の検索で有効

+500点ほど

## dispenseIDでMySQLを使うのをやめる

一意なidを生成するために以下のようにわざわざDBにアクセスしているが、これをuuidを生成するようにする

[コミットログ](https://github.com/reiichii/isucon12q-after/commit/3409af51893bbda12ca68dc1ff1d1de914b0bb14)

SQLの`REPLACE INTO`とは

* 基本INSERTと同じだが、テーブル内の古い行にprivary keyまたはuniqueインデックスに関して新しい行と同じ値が含まれている場合その古い行は新しい行が挿入される前に削除される
* 挿入 or 削除と挿入　の違い

raise fromについて

* 例外を連鎖することができる
    ```python
    try:
        raise ConnectionError()
    except ConnectionError as e:
        raise RuntimeError("Failed to open database") from e
    ```
  - 出力は以下：`The above exception was the direct cause of the following exception`
```python
Traceback (most recent call last):
  File "~/ghq/github.com/reiichii/isucon12q-after/tmp.py", line 2, in <module>
    raise ConnectionError()
ConnectionError

The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/~/ghq/github.com/reiichii/isucon12q-after/tmp.py", line 4, in <module>
    raise RuntimeError("Failed to open database") from e
RuntimeError: Failed to open database
```
* from を使わないと、`During handling of the above exception, another exception occurred` のようになる

+200点ほど

## Ranking APIのループクエリをなくす

リクエストの合計時間が一番長い /api/player/competition/<competition_id>/ranking をなんとかする。

N+1になっているのでjoinを使う。

[コミットログ](https://github.com/reiichii/isucon12q-after/commit/62e3d7bd4bdaca3b14cc682e0bce6605de907014)

+1000点になりました😳

## Score APIの追加のループクエリをなくす

> rankingの次にレスポンスタイム合計が大きいのはscoreなので

Node.jsで解いていたブログ記事では上記のように書いてあったが、私の環境(Python)ではscoreよりも/api/player/player/<player_id> の方が重かったです。

自分では最後のinsertのところをbulk insertにすればいいのかなと思っていたが、存在しないplayer_idを返す必要はないので数を比較するだけで十分という考えには至れませんでした。

[コミットログ](https://github.com/reiichii/isucon12q-after/commit/8f787574d5a847a7f1cc33dc7ecdb4e35a1403d8)


こちらも+1000点ほど

## アトミック書き込みのためのflockをトランザクションに変更する(※怪しい)

既存コードではテナントDB更新の際に、排他制御をするためにファイルをロックすることをしていますが、トランザクションを使うようにします。
delete-insertの部分をトランザクションにしてflockを外す。他のflockは参照のみなので外すだけで良かった。

[コミットログ](https://github.com/reiichii/isucon12q-after/commit/69b53bfb6318f46cdc0db67e38c1fb64271693a0)

この部分を実装したところ、数回に1回整合性チェックが通らなくなりました😢おそらくトランザクションがちゃんと効いていない模様で、なんでか全然分からなかったのですがおそらくAuto Commitが効いてしまっているところに思い至ったのでこれから確認する段階です。

そしてなぜか点数はそれほど上がらないどころか実行するたびに数百点の振り幅が出るように。

## adminDB visit_historyの初期データをコンパクトにする

アプリケーションの作りがアクセスしたかどうかが分かればいいため、visit_historyのテナントID、大会ID、プレイヤーIDをgroup byしてmin(created_at) / min(updated_at)のデータのみが残るようにして重複したデータを減らす。

ちなみに対象テーブルのMySQLの初期化の部分は以下のようになっていて、一定のデータが消えないようになっています。

```sql
DELETE FROM visit_history WHERE created_at >= '1654041600';
```

念の為既存データを残しておきたかったので、私は以下の手順で実施しました。

1. 一時テーブルを作成（visit_history_tmpとする）
2. INSERT SELECT
    ```sql
    INSERT INTO visit_history_tmp
    SELECT player_id, tenant_id, competition_id, MIN(created_at), MIN(updated_at)
    FROM visit_history
    GROUP BY 1,2,3;
    ```
3. 古いテーブルをrename
    ```sql
    RENAME TABLE visit_history TO visit_history_backup;
    ```
4. 一時テーブルをvisit_historyにrename
    ```sql
    RENAME TABLE visit_history_tmp TO visit_history;
    ```

初期化時点の行数：3,224,839 → 削減後のデータ数：200,474（0.06%にまで削減された）

ただし私の場合スコアは変わらず

## DB用サーバを投入し、2台構成にする

ブログの方では複数台構成準備のための施策に突入するのですが、私は先にappとdbの二台構成にすることにしました。

* mysqlで他サーバからのアクセスを許容する
```
CREATE USER `isucon`@`192.168.%` IDENTIFIED BY 'isucon';
GRANT ALL PRIVILEGES ON `isuports`.* TO `isucon`@`192.168.%`;
```
* application側で参照先dbを変更
  - 今回はdocker-composeにホストが書いてあったのでそこの値を変更する

前のベンチマークの時点でCPUが余っていたので、これやっても点数が大して変わらないのは予想通りでした。

## Finish APIでBillingReportを生成する

> 今回の当日マニュアルにあった、「Finish APIを呼び出したあとにAdmin/OrganizerのBilling APIに結果が反映されるまで3秒の猶予があるの意味は、「初期実装だとBilling APIで請求額を計算しているけど、大会ごとにfinishするときに大会の請求額が確定するので、BillingReportをそこで生成してストレージにいれてね!」です。

分からん...😇

finish が呼ばれた時にbilling_report_by_competitionを呼び出して、その結果をinsertします。

* テーブルを作成
    ```sql
    CREATE TABLE `billing_report` (
    `tenant_id` BIGINT UNSIGNED NOT NULL,
    `competition_id` VARCHAR(255) NOT NULL,
    `competition_title` VARCHAR(255) NOT NULL,
    `player_count` BIGINT NOT NULL,
    `visitor_count` BIGINT NOT NULL,
    `billing_player_yen` BIGINT NOT NULL,
    `billing_visitor_yen` BIGINT NOT NULL,
    `billing_yen` BIGINT NOT NULL,
    PRIMARY KEY(`tenant_id`, `competition_id`)
    ) ENGINE=InnoDB DEFAULT CHARACTER SET=utf8mb4;
    ```
* finish apiの時にbilling_report_by_competitionを呼び出して結果をinsertする
* admin/organizationのbillingの参照先をdbからselectして取ってくる
* 初期データ生成処理を改修
  * 初期データを入れ直したあとに全ての終了済み大会について billingReportByCompetition を実行してINSERTしなおす必要がある
  * billing report初期データ生成スクリプトを作成
  * `mysqldump -uroot -proot isuports billing_report > initial_billing_report.dump`
  * initial時に初期データをimportさせる

[コミットログ](https://github.com/reiichii/isucon12q-after/commit/64b7251efa85305e548fbfac2c19fed82d2379f9)

スコアはそれほど変わらず、不安定さが増してしまったように見受けられました。(編集とは関係ないエンドポイントでエラーが発生する)
ただ`api/admin/tenants/billing`, `api/organizer/billing`の呼び出し回数と合計レスポンスタイムが大幅に改善されているので一旦よしとします。

## Player APIをなんとかする

上記のメトリクスを眺めているときにPlayer APIがものすごく重たくなっている(MAX 5s程度だったものがMAX 30sになっていた笑)ことに気づき、あまりにも気になったので先に直すことにしました。

[コミットログ](https://github.com/reiichii/isucon12q-after/commit/17801b6a6af423f4ea6bc0670ba91af8c0111660)

これもN+1を直すだけです。必要な情報に対して多くクエリを発行しているのでスリムに書き直してあげます。

今まで4000点代で伸び悩んでいたスコアが6000点台まで届きました👏

## おわりに

スコアが伸び悩んで、また他のことをやりたくなってきたのもあり、8月いっぱいで一旦やめにしようかなと思いかけていたのですが、月末の週に突入して解決の兆しが見えてきたので、もう少し粘ってみようかと思います。

複数台構成にしたら10万点まで届くのだろうか...

続きも書けたら書きます。