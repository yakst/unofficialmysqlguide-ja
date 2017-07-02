---
title: パーティショニング
date: 2017-07-03 00:00:01 -0900
article_index: 19
original_url: http://www.unofficialmysqlguide.com/partitioning.html
translator: taka-h (@takaidohigasi)
---

オプティマイザは*パーティションプルーニング*ができます。つまり、入力のクエリーを分析し、ディクショナリ情報と比較し、必要なテーブルパーティションだけにアクセスさせることができます。

パーティショニングは、それが1つのテーブルの論理的な表現であり、一連のテーブルがその下にあるいう点で、ビューと似たように考えることができます。パーティショニングはインデックスを利用した上で、全てのクエリーが共通のパターンに従う場合に適しています。例えば、

* **論理削除(soft delete)**がある場合: 多くのアプリケーションは論理削除を実装しており(例: is\_deleted)、削除済みあるいは削除されていないデータのみによくアクセスします。
* **マルチバージョンスキームが適用されたスキーマ[^1]**がある場合:  アプリケーションによってはデータを決して削除せず、古い世代の行を履歴を残す目的で保存するポリシーをとるものがあります。例えばあるユーザーの住所を更新するとき、全ての以前の住所を保持する場合です。住所が古く無効なものかどうかでパーティショニングするとメモリーを効率的に利用できます。
* **時系列的(time oriented)**な場合: 例えば、パーティションが四半期あるいは会計周期で作成される場合です。
* **局所性(locality)**がある場合: 例えば、パーティションがstore\_id(店舗ID)あるいは地域で作成される場合です。

![partition-countrylanguage.png](http://www.unofficialmysqlguide.com/_images/partition-countrylanguage.png)

パーティショニングの性能は、ほとんどのクエリーが1つあるいはごく一部のパーティションにしか一度にアクセスしない場合に最善となります。我々が例でみてきたスキーマで考えてみると、`CountryLanguage`テーブルへの全てのクエリーが公用語のみに対して発行されるケースが該当するでしょう。もしこれが本当であれば、`isOfficial`によるパーティショニングは次のように実現できます。

```sql
# IsOfficialはENUMですが、CHARに変換することができます
# ENUM型はKEYパーティショニングのみをサポートしています
# パーティショニングキーは主キーの一部であり、かつ全てユニークキーである必要があります

ALTER TABLE CountryLanguage MODIFY IsOfficial CHAR(1) NOT NULL DEFAULT 'F', DROP PRIMARY KEY, ADD PRIMARY KEY(CountryCode, Language, IsOfficial);
ALTER TABLE CountryLanguage PARTITION BY LIST COLUMNS (IsOfficial) (
 PARTITION pUnofficial VALUES IN ('F'),
 PARTITION pOfficial VALUES IN ('T')
);
```

### 例32: オプティマイザでパーティションプルーニングされている

```sql
EXPLAIN FORMAT=JSON
SELECT * FROM CountryLanguage WHERE isOfficial='T' AND CountryCode='CAN';
{
  "query_block": {
    "select_id": 1,
    "cost_info": {
      "query_cost": "2.40"
    },
    "table": {
      "table_name": "CountryLanguage",
      "partitions": [ 
        "pOfficial"                # 公用語が含まれるパーティションのみがアクセスされる
      ],
      "access_type": "ref",
      "possible_keys": [
        "PRIMARY",
        "CountryCode"
      ],
      "key": "PRIMARY",
      "used_key_parts": [
        "CountryCode"
      ],
      "key_length": "3",
      "ref": [
        "const"
      ],
      "rows_examined_per_scan": 2,
      "rows_produced_per_join": 0,
      "filtered": "10.00",
      "cost_info": {
        "read_cost": "2.00",
        "eval_cost": "0.04",
        "prefix_cost": "2.40",
        "data_read_per_join": "8"
      },
      "used_columns": [
        "CountryCode",
        "Language",
        "IsOfficial",
        "Percentage"
      ],
      "attached_condition": "(`world`.`CountryLanguage`.`IsOfficial` = 'T')"
    }
  }
}
```

{% include info.html info="MySQLはパーティションプルーニングにくわえて、選択されたパーティションのみを明示的に指定する構文をサポートしています。これは次のようにアプリケーションからクエリーヒントと似たように使うすることができます。SELECT * FROM CountryLanguage PARTITION (pOfficial) WHERE CountryCode='CAN'; " %}

[^1]: データウェアハウスにおいて、これはslowly changing dimensionsと呼ばれます。[https://en.wikipedia.org/wiki/Slowly_changing_dimension](https://en.wikipedia.org/wiki/Slowly_changing_dimension)
