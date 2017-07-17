---
title: 文字セット
date: 2017-07-10 00:00:01 -0900
article_index: 24
original_url: http://www.unofficialmysqlguide.com/character-sets.html
translator: taka-h (@takaidohigasi)
---

MySQL 8.0では最新のUnicode 9.0が`utf8mb4`という名前でサポートされています。`utf8mb4`はそれぞれの文字が1から4バイトの*可変長*です。
可変長の文字数とバイト数が少し異なるケースはたくさんあり例えば次のようなケースがあげられます。

1. 列を`VARCHAR(n)`で作成した時、*n*は文字列長を表します。データの保存に必要なバイト数は(多くの場合はこれより小さいのですが)最大この4倍となりえます。
2. 内部的にはInnoDBストレージエンジンは常に[^1]`utf8mb4`を`VARCHAR`、`CHAR`、`TEXT`データ型の(インデックス内およびテーブルの行にて)可変長として保存しています。
3. マテリアライズの一部として利用されるメモリ内の一時テーブルは固定長です。`utf8mb4`文字セットを利用している場合は一時テーブルがより大きくなったり、より早くディスクにあふれたりすることになる場合があります。
4. データをソートするために利用されているバッファーは可変長です(MySQL 5.7より)。
5. `EXPLAIN`は常にインデックスの可変長(バイト長)の最大長を示します。もっと少ない領域しか必要とならない場合がしばしば発生します。

例37は、latin1文字セットで`CHAR(52)`の列へのインデックスを利用していることを伝えている`EXPLAIN`となります。テーブルを`utf8mb4`に変換した後、必要な保存容量は増えませんが、`EXPLAIN`では`key_length`が増えていることが確認できます。

### 例37: `EXPLAIN`がインデックスの最大キー長を示す(latin1文字セットにて)

```sql
     EXPLAIN FORMAT=JSON
     SELECT * FROM Country WHERE name='Canada';
     {
       "query_block": {
         "select_id": 1,
         "cost_info": {
           "query_cost": "1.20"
         },
         "table": {
           "table_name": "Country",
           "access_type": "ref",
           "possible_keys": [
             "Name"
           ],
           "key": "Name",
           "used_key_parts": [
             "Name"
           ],
           "key_length": "52",  # CHAR(52)
           "ref": [
             "const"
           ],
           "rows_examined_per_scan": 1,
           "rows_produced_per_join": 1,
           "filtered": "100.00",
           "cost_info": {
             "read_cost": "1.00",
             "eval_cost": "0.20",
             "prefix_cost": "1.20",
             "data_read_per_join": "264"
           },
           "used_columns": [
             "Code",
             "Name",
             "Continent",
             "Region",
             "SurfaceArea",
             "IndepYear",
             "Population",
             "LifeExpectancy",
             "GNP",
             "GNPOld",
             "LocalName",
             "GovernmentForm",
             "HeadOfState",
             "Capital",
             "Code2"
           ]
         }
       }
     }
```

## 例38: `EXPLAIN`がインデックスの最大キー長を示す(utfmb4文字セットにて)

```sql
     ALTER TABLE Country CONVERT TO CHARACTER SET utf8mb4;
     EXPLAIN FORMAT=JSON
     SELECT * FROM Country WHERE name='Canada';
     {
       "query_block": {
         "select_id": 1,
         "cost_info": {
           "query_cost": "1.20"
         },
         "table": {
           "table_name": "Country",
           "access_type": "ref",
           "possible_keys": [
             "Name"
           ],
           "key": "Name",
           "used_key_parts": [
             "Name"
           ],
           "key_length": "208", # CHAR(52) * 4 = 208
           "ref": [
             "const"
           ],
           "rows_examined_per_scan": 1,
           "rows_produced_per_join": 1,
           "filtered": "100.00",
           "cost_info": {
             "read_cost": "1.00",
             "eval_cost": "0.20",
             "prefix_cost": "1.20",
             "data_read_per_join": "968"
           },
           "used_columns": [
             "Code",
             "Name",
             "Continent",
             "Region",
             "SurfaceArea",
             "IndepYear",
             "Population",
             "LifeExpectancy",
             "GNP",
             "GNPOld",
             "LocalName",
             "GovernmentForm",
             "HeadOfState",
             "Capital",
             "Code2"
           ]
         }
       }
     }
```

[^1]: 行形式に`DYNAMIC`、`COMPACT`および`COMPRESSED`を利用しているときは常に。通常は、以前の`REDUNDANT`行形式は実用的な利用例がありません。
