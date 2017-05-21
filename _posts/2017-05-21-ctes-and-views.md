---
title: 共通テーブル式(CTE)とビュー
date: 2017-05-21 00:00:01 -0900
article_index: 15
original_url: http://www.unofficialmysqlguide.com/ctes-and-views.html
translator: taka-h (@takaidohigasi)
---

ビューはクエリーをあとで再利用するために保存する方法の1つで、アプリケーションから見るとテーブルのようにみえます。これによって複雑なSQLクエリーをブレークダウンし、段階をおうことで単純化することが出来ます。共通テーブル式(CTE)自体はビューととても似通っており、両者の違いは共通テーブル式は寿命が短く、単一のステートメントのみに限定されることです。

MySQLのオプティマイザには、共通テーブル式およびビューを実行するにあたって主に2つの実行戦略があります。

1. **マージ** :  共通テーブル式またはビューの定義が残りのクエリーとマージされるようにクエリーを変換します。`EXPLAIN`を実行した後に`SHOW WARNINGS`を実行することでどのようにマージされたかを確認することができます。
2. **実体化** : 共通テーブル式またはビューを実行しその結果を一時テーブルに保存します。残りのクエリーは一時テーブルに対して実行されます。実体化は一般的には遅い手法であり、マージの手法が適さないと判断されたときに選択されます。早期に実体化することで工程が短縮され実行が高速化する可能性があり、このときは上記の例外となります。

### 例21: ビューに対しクエリーが発行されマージされる

```sql
CREATE VIEW vCountry_Asia AS SELECT * FROM Country WHERE Continent='Asia';
EXPLAIN FORMAT=JSON
SELECT * FROM vCountry_Asia WHERE Name='China';
{
 "query_block": {
   "select_id": 1,
   "cost_info": {
     "query_cost": "1.20"
   },
   "table": {
     "table_name": "Country",  # Table name is the base table name
     "access_type": "ref",
     "possible_keys": [
       "c",
       "c_p",
       "Name"
     ],
     "key": "Name",
     "used_key_parts": [
       "Name",
       "Continent"
     ],
     "key_length": "53",
     "ref": [
       "const",
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

SHOW WARNINGS;
/* select#1 */ select `world`.`Country`.`Code` AS `Code`,`world`.`Country`.`Name` AS `Name`,`world`.`Country`.`Continent` AS `Continent`,`world`.`Country`.`Region` AS `Region`,`world`.`Country`.`SurfaceArea` AS `SurfaceArea`,`world`.`Country`.`IndepYear` AS `IndepYear`,`world`.`Country`.`Population` AS `Population`,`world`.`Country`.`LifeExpectancy` AS `LifeExpectancy`,`world`.`Country`.`GNP` AS `GNP`,`world`.`Country`.`GNPOld` AS `GNPOld`,`world`.`Country`.`LocalName` AS `LocalName`,`world`.`Country`.`GovernmentForm` AS `GovernmentForm`,`world`.`Country`.`HeadOfState` AS `HeadOfState`,`world`.`Country`.`Capital` AS `Capital`,`world`.`Country`.`Code2` AS `Code2` from `world`.`Country` where ((`world`.`Country`.`Continent` = 'Asia') and (`world`.`Country`.`Name` = 'China'))
```

![explain-example-22](http://www.unofficialmysqlguide.com/_images/explain-example-22.png)

### 例22: ビューに対しクエリーが発行されるがマージできない

```sql
CREATE VIEW vCountrys_Per_Continent AS SELECT Continent, COUNT(*) as Count FROM Country
GROUP BY Continent;
EXPLAIN FORMAT=JSON
SELECT * FROM vCountrys_Per_Continent WHERE Continent='Asia';
{
 "query_block": {
   "select_id": 1,
   "cost_info": {
     "query_cost": "12.47"
   },
   "table": {
     "table_name": "vCountrys_Per_Continent",  # Table name is the view name
     "access_type": "ref",
     "possible_keys": [
       "<auto_key0>"
     ],
     "key": "<auto_key0>",
     "used_key_parts": [
       "Continent"
     ],
     "key_length": "1",
     "ref": [
       "const"
     ],
     "rows_examined_per_scan": 10,
     "rows_produced_per_join": 10,
     "filtered": "100.00",
     "cost_info": {
       "read_cost": "10.39",
       "eval_cost": "2.08",
       "prefix_cost": "12.47",
       "data_read_per_join": "166"
     },
     "used_columns": [
       "Continent",
       "Count"
     ],
     "materialized_from_subquery": {
       "using_temporary_table": true,
       "dependent": false,
       "cacheable": true,
       "query_block": {
         "select_id": 2,
         "cost_info": {
           "query_cost": "56.80"
         },
         "grouping_operation": {
           "using_filesort": false,
           "table": {
             "table_name": "Country",
             "access_type": "index",
             "possible_keys": [
               "PRIMARY",
               "p",
               "c",
               "p_c",
               "c_p",
               "Name"
             ],
             "key": "c",
             "used_key_parts": [
               "Continent"
             ],
             "key_length": "1",
             "rows_examined_per_scan": 239,
             "rows_produced_per_join": 239,
             "filtered": "100.00",
             "using_index": true,
             "cost_info": {
               "read_cost": "9.00",
               "eval_cost": "47.80",
               "prefix_cost": "56.80",
               "data_read_per_join": "61K"
             },
             "used_columns": [
               "Code",
               "Continent"
             ]
           }
         }
       }
     }
   }
 }
}

SHOW WARNINGS;
/* select#1 */ select `vCountrys_Per_Continent`.`Continent` AS `Continent`,`vCountrys_Per_Continent`.`Count` AS `Count` from `world`.`vCountrys_Per_Continent` where (`vCountrys_Per_Continent`.`Continent` = 'Asia')
```

