---
title: カバリングインデックス
date: 2017-04-08 00:00:01 -0900
article_index: 11
original_url: http://www.unofficialmysqlguide.com/covering-indexes.html
translator: taka-h (@takaidohigasi)
---

カバリングインデックスは複合インデックスの特異な形式で、インデックスに全ての列を包含するものです。このシナリオでは、MySQLはテーブルの行にアクセスすることがなく「インデックスから」データを返すという最適化をすることができます。

![explan-cpn](http://www.unofficialmysqlguide.com/_images/explain-cpn.png)

`SELECT * FROM Country`以外で、`population > 5M and continent='Asia'`の条件を満たす国の名前のみが必要な場合を考えてみましょう。例12で示されるように、`c_p_n (Continent,Population,Name)`へのインデックスは、最初の2列を行を絞込むために、3つ目の列を値を返すために利用できます。

### 例12: c_p_nへのカバリングインデックス

```sql
ALTER TABLE Country ADD INDEX c_p_n (Continent,Population,Name);
EXPLAIN FORMAT=JSON
SELECT Name FROM Country WHERE continent='Asia' and population > 5000000;
{
  "query_block": {
     "select_id": 1,
     "cost_info": {
     "query_cost": "8.07"   # コストが67%削減されている
     },
     "table": {
     "table_name": "Country",
     "access_type": "range",
     "possible_keys": [
     "p",
     "c",
     "p_c",
     "c_p",
     "c_p_n"
     ],
     "key": "c_p_n",
     "used_key_parts": [
     "Continent",
     "Population"
     ],
     "key_length": "5",
     "rows_examined_per_scan": 32,
     "rows_produced_per_join": 15,
     "filtered": "100.00",
     "using_index": true,      # Using indexは「カバリングインデックス」を意味する
     "cost_info": {
     "read_cost": "1.24",
     "eval_cost": "3.09",
     "prefix_cost": "8.07",
     "data_read_per_join": "3K"
     },
     "used_columns": [
     "Name",
     "Continent",
     "Population"
     ],
     "attached_condition": "((`world`.`Country`.`Continent` = 'Asia') and (`world`.`Country`.`Population` > 5000000))"
     }
  }
}
```

カバリングインデックスが利用されたことが`EXPLAIN`の`"using_index": true`によって示されています。カバリングインデックスは正当に評価されない最適化の一つです。多くの方々が、カバリングインデックスではインデックスが利用される一方で、テーブルの行にアクセスしないため「コストが半分になる」と誤ってお考えです。例12では例11に示されるカバリングインデックスでないインデックスの場合と比べて約1/3のコストになっていることが分かります。

本番環境ではインデックスのクラスタリング効果によって、他のクエリーに比べメモリーをよりうまく使えるでしょう。アクセスされるセカンダリインデックスがクラスタインデックス(主キー)と「相関[^1]」がなかったらより多くのクラスタ化されたキーのページへのアクセスが必要となるためです。

[^1]: 相関というのは「だいたい同じ順序」ということを意図しています。例えば、セカンダリインデックスがtimestap型で挿入された場合、auto_incrementの主キーと高い相関があります。また、populationとcontinentへのインデックスは3文字の国コードである主キーと相関がありそうにありません。
