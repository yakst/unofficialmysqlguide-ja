---
title: クエリープロファイリング
date: 2017-07-08 00:00:01 -0900
article_index: 22
original_url: http://www.unofficialmysqlguide.com/profiling.html
translator: taka-h (@takaidohigasi)
---

`EXPLAIN`はクエリーのコストに関する実行前にえられる情報のみを表示します。これにはもう少し全体的な俯瞰をするためのクエリー実行に関する統計は一切含まれません。例えば、オプティマイザがインデックスで行を除外(`EXPLAIN`内でテーブルに対しattached\_conditionを付与)できなかった場合、何行が除外されたのかがわかりません。結合条件に対して[トリクルダウン理論](https://ja.wikipedia.org/wiki/%E3%83%88%E3%83%AA%E3%82%AF%E3%83%AB%E3%83%80%E3%82%A6%E3%83%B3%E7%90%86%E8%AB%96)が適用され、関連テーブルへの参照がとても多くなるのか、少なくなるのかわからない可能性があります。

MySQLは実行の各ステップでどの程度時間がかかったかを表す基本的なプロファイリング機能をサポートしており、`performance_schema`からこの情報をえることができます。これは、以前は`SHOW PROFILES`コマンドでしたが、将来的に廃止予定となったためこれに取ってかわるものです。例としてご紹介するために、私は`SHOW PROFILES`を`SYS`スキーマベースで置き換えたものを利用しており、これは次のようにインストールできます。

```
# 下記はシェルにて実行
wget http://www.tocker.ca/files/ps-show-profiles.sql
mysql -u root -p < ps-show-profiles.sql
```

### 例34: Performance Schemaでクエリーをプロファイリングする

```
# 下記はMySQLにて実行
CALL sys.enable_profiling();
CALL sys.show_profiles;
*************************** 1. row ***************************
Event_ID: 22
Duration: 495.02 us
   Query: SELECT * FROM Country WHERE co ... Asia' and population > 5000000
1 row in set (0.00 sec)

CALL sys.show_profile_for_event_id(22);
+----------------------+-----------+
| Status               | Duration  |
+----------------------+-----------+
| starting             | 64.82 us  |
| checking permissions | 4.10 us   |
| Opening tables       | 11.87 us  |
| init                 | 29.74 us  |
| System lock          | 5.63 us   |
| optimizing           | 8.74 us   |
| statistics           | 139.38 us |
| preparing            | 11.94 us  |
| executing            | 348.00 ns |
| Sending data         | 192.59 us |
| end                  | 1.17 us   |
| query end            | 4.60 us   |
| closing tables       | 4.07 us   |
| freeing items        | 13.60 us  |
| cleaning up          | 734.00 ns |
+----------------------+-----------+
15 rows in set (0.00 sec)
```

`SLEEP()`関数を利用すれば、実行の特定のステップでたくさんの時間を利用していることを実際に確認することができます。このクエリーでは、MySQLは`continent='Antarctica'`の条件に行が合致する毎に5秒スリープします。

```sql
SELECT * FROM Country WHERE Continent='Antarctica' and SLEEP(5);
CALL sys.show_profiles();
CALL sys.show_profile_for_event_id(<event_id>);
+----------------------+-----------+
| Status               | Duration  |
+----------------------+-----------+
| starting             | 103.89 us |
| checking permissions | 4.48 us   |
| Opening tables       | 17.78 us  |
| init                 | 45.75 us  |
| System lock          | 8.37 us   |
| optimizing           | 11.98 us  |
| statistics           | 144.78 us |
| preparing            | 15.78 us  |
| executing            | 634.00 ns |
| Sending data         | 116.15 us |
| User sleep           | 5.00 s    |
| User sleep           | 5.00 s    |
| User sleep           | 5.00 s    |
| User sleep           | 5.00 s    |
| User sleep           | 5.00 s    |
| end                  | 2.05 us   |
| query end            | 5.63 us   |
| closing tables       | 7.30 us   |
| freeing items        | 20.19 us  |
| cleaning up          | 1.20 us   |
+----------------------+-----------+
20 rows in set (0.01 sec)
```

プロファイリングの出力が必ずしもきめ細かくなっているわけではないことが分かるでしょう。例えば、`Sending Data`は*ストレージエンジンとサーバー間でデータを転送している*ということしかわかりません。重要な点としては、一時テーブルの作成とソートの実行時間がブレークダウンされています。

```sql
ELECT region, count(*) as c FROM Country GROUP BY region;
CALL sys.show_profiles();
CALL sys.show_profile_for_event_id(<event_id>);
+----------------------+-----------+
| Status               | Duration  |
+----------------------+-----------+
| starting             | 87.43 us  |
| checking permissions | 4.93 us   |
| Opening tables       | 17.35 us  |
| init                 | 25.81 us  |
| System lock          | 9.04 us   |
| optimizing           | 3.37 us   |
| statistics           | 18.31 us  |
| preparing            | 10.94 us  |
| Creating tmp table   | 35.57 us  |   # < --
| Sorting result       | 2.38 us   |   # < --
| executing            | 741.00 ns |
| Sending data         | 446.03 us |   # < --
| Creating sort index  | 49.45 us  |   # < --
| end                  | 1.71 us   |
| query end            | 4.85 us   |
| removing tmp table   | 4.71 us   |
| closing tables       | 6.12 us   |
| freeing items        | 17.17 us  |
| cleaning up          | 1.00 us   |
+----------------------+-----------+
19 rows in set (0.01 sec)
```

`performance_schema`はこれらのヘルパープロシージャで示されるプロファイル情報に加えて、返却された行数だけでなくソートに必要とされた実際の行数に関する追加の統計情報を保持しています。実行レベルの分析は、`EXPLAIN`で確認できる実行前の解析に対して次のように追加の情報を提供します。

```sql
SELECT * FROM performance_schema.events_statements_history_long
WHERE event_id=<event_id>\G
*************************** 1. row ***************************
              THREAD_ID: 3062
               EVENT_ID: 1566
           END_EVENT_ID: 1585
             EVENT_NAME: statement/sql/select
                 SOURCE: init_net_server_extension.cc:80
            TIMER_START: 588883869566277000
              TIMER_END: 588883870317683000
             TIMER_WAIT: 751406000
              LOCK_TIME: 132000000
               SQL_TEXT: SELECT region, count(*) as c FROM Country GROUP BY region
                 DIGEST: d3a04b346fe48da4f1f5c2e06628a245
            DIGEST_TEXT: SELECT `region` , COUNT ( * ) AS `c` FROM `Country` GROUP BY `region`
         CURRENT_SCHEMA: world
            OBJECT_TYPE: NULL
          OBJECT_SCHEMA: NULL
            OBJECT_NAME: NULL
  OBJECT_INSTANCE_BEGIN: NULL
            MYSQL_ERRNO: 0
      RETURNED_SQLSTATE: NULL
           MESSAGE_TEXT: NULL
                 ERRORS: 0
               WARNINGS: 0
          ROWS_AFFECTED: 0
              ROWS_SENT: 25           # < --
          ROWS_EXAMINED: 289          # < --
CREATED_TMP_DISK_TABLES: 0
     CREATED_TMP_TABLES: 1
       SELECT_FULL_JOIN: 0
 SELECT_FULL_RANGE_JOIN: 0
           SELECT_RANGE: 0
     SELECT_RANGE_CHECK: 0
            SELECT_SCAN: 1
      SORT_MERGE_PASSES: 0
             SORT_RANGE: 0
              SORT_ROWS: 25           # < --
              SORT_SCAN: 1
          NO_INDEX_USED: 1
     NO_GOOD_INDEX_USED: 0
       NESTING_EVENT_ID: NULL
     NESTING_EVENT_TYPE: NULL
    NESTING_EVENT_LEVEL: 0
```
