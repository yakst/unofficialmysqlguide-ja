---
title: 不可視インデックス 
date: 2017-07-04 00:00:01 -0900
article_index: 21
original_url: http://www.unofficialmysqlguide.com/invisible-indexes.html
translator: taka-h (@takaidohigasi)
---

不可視インデックスはMySQL 8.0の新機能で、インデックスに「利用できない」と印をつけこれをオプティマイザが利用するためのものです。つまりインデックスはメンテナンスされデータが更新されれば最新の状態に保たれますが、インデックスを利用してクエリーを発行することができません(`FORCE INDEX インデックス名`クエリーだとしても、です)

不可視インデックスはMyISAMストレージエンジンが実装している無効化インデックス(disabled indexes)と混同してはなりません(無効化インデックスはインデックスのメンテナンスをやめます)。不可視インデックスには特筆すべき2つのユースケースがあります。

1. **論理削除(soft delete)**。商用環境で破壊的な操作を行うときは、変更を永続化する前に様子を見ることができることが望ましいでしょう。インデックスの「ごみ箱」のようであると考えてみてください。仮にあなたが誤りを犯しインデックスが利用されていた場合、再度可視化するためにはただメタ情報を変更すればよいことになります、バックアップから復元するよりはるかに高速です。例えば、次のとおりです
   `ALTER TABLE Country ALTER INDEX c INVISIBLE;`
2. **段階的な展開(staged rollout)**。インデックスを追加するときには、望まない変更が発生する場合もありますので、既存のクエリーの実行計画に変化が発生するかどうかを考慮することが常に大切となります。不可視インデックスを利用すると、ピーク負荷をさけシステムを能動的に見守ることができる任意の時刻で、インデックスを段階的に展開することができます。例えば、次のとおりです。
    ```sql
    ALTER TABLE Country DROP INDEX c;
    ALTER TABLE Country ADD INDEX c (Continent) INVISIBLE;
    # after some time
    ALTER TABLE Country ALTER INDEX c VISIBLE;
    ```
全てのインデックスは不可視であると指定されない限り、デフォルトで可視となります。全スキーマの不可視インデックスは次のようにして探すことができます。

```sql
SELECT * FROM information_schema.statistics WHERE is_visible='NO';
*************************** 1. row ***************************
TABLE_CATALOG: def
 TABLE_SCHEMA: world
   TABLE_NAME: Country
   NON_UNIQUE: 1
 INDEX_SCHEMA: world
   INDEX_NAME: c
 SEQ_IN_INDEX: 1
  COLUMN_NAME: Continent
    COLLATION: A
  CARDINALITY: 7
     SUB_PART: NULL
       PACKED: NULL
     NULLABLE:
   INDEX_TYPE: BTREE
      COMMENT: disabled
INDEX_COMMENT:
   IS_VISIBLE: NO
```

{% include info.html info="不必要なインデックスを削除するのはいい考えです。ほとんどの方々はインデックスは修正(insert、update)の性能を損なうことはお気づきであるように、これ自体が単純化につながります。不要なインデックスは、オプティマイザがプラン選択のときに評価する必要があり読み取り性能にも影響を与えることでしょう。" %}
