---
title: サブクエリー 
date: 2017-05-15 00:00:01 -0900
article_index: 14
original_url: http://www.unofficialmysqlguide.com/subqueries.html
translator: taka-h (@takaidohigasi)
---

オプティマイザーにはサブクエリーを最適化する多数の実行戦略があり、例えばjoinあるいはsemi-join(準結合)にクエリーを書き換える手法、あるいは実体化があります。利用される最適化戦略はどのようなサブクエリーであるか、やその配置に依存します。

## スカラーサブクエリー

スカラーサブクエリーとは、サブクエリーのうちきっかり1行を返すもののことです。スカラーサブクエリーは、実行の最中に最適化により除去することができ、結果がキャッシュされます。例13で、Torontoの`CountryCode`を取得するスカラーサブクエリーの例を示しています。オプティマイザーがこれを2つのクエリーとみなしているのは重要で、それぞれコストが1.00および4213.00となっています(訳注: コストの値は修正もれで1.20および862.60と思われます)。

![explain-scalar](http://www.unofficialmysqlguide.com/_images/explain-scalar.png)

2番目のクエリー(`select_id: 2`)では利用できるインデックスがないため、テーブルスキャンが実行されています。attached\_condition(City.Name)にインデックスが付与されていないのが分かり、インデックスを追加した後クエリーが最適化されています。

### 例13: スカラーサブクエリー

```sql
EXPLAIN FORMAT=JSON
SELECT * FROM Country WHERE Code = (SELECT CountryCode FROM City WHERE name='Toronto');
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
        "PRIMARY"
      ],
      "key": "PRIMARY",
      "used_key_parts": [
        "Code"
      ],
      "key_length": "3",
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
      ],
      "attached_condition": "(`world`.`Country`.`Code` = (/* select#2 */ select `world`.`City`.`CountryCode` from `world`.`City` where (`world`.`City`.`Name` = 'Toronto')))",
      "attached_subqueries": [
        {
          "dependent": false,
          "cacheable": true,
          "query_block": {
            "select_id": 2,
            "cost_info": {
              "query_cost": "862.60"
            },
            "table": {
              "table_name": "City",
              "access_type": "ALL",
              "rows_examined_per_scan": 4188,
              "rows_produced_per_join": 418,
              "filtered": "10.00",
              "cost_info": {
                "read_cost": "778.84",
                "eval_cost": "83.76",
                "prefix_cost": "862.60",
                "data_read_per_join": "29K"
              },
              "used_columns": [
                "Name",
                "CountryCode"
              ],
              "attached_condition": "(`world`.`City`.`Name` = 'Toronto')"
            }
          }
        }
      ]
    }
  }
}
```

### 例14: スカラーサブクエリーを改善するためにインデックスを追加

```sql
# インデックスを追加
ALTER TABLE City ADD INDEX n (Name);
EXPLAIN FORMAT=JSON
SELECT * FROM Country WHERE Code = (SELECT CountryCode FROM City WHERE name='Toronto');
{
"query_block": {
"select_id": 1,
"cost_info": {
   "query_cost": "1.00"
},
"table": {
   "table_name": "Country",
   "access_type": "const",
   "possible_keys": [
   "PRIMARY"
   ],
   "key": "PRIMARY",
   "used_key_parts": [
   "Code"
   ],
   "key_length": "3",
   "ref": [
   "const"
   ],
   "rows_examined_per_scan": 1,
   "rows_produced_per_join": 1,
   "filtered": "100.00",
   "cost_info": {
   "read_cost": "0.00",
   "eval_cost": "1.00",
   "prefix_cost": "0.00",
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
},
"optimized_away_subqueries": [
   {
   "dependent": false,
   "cacheable": true,
   "query_block": {
   "select_id": 2,
   "cost_info": {
      "query_cost": "2.00"
   },
   "table": {
      "table_name": "City",
      "access_type": "ref",
      "possible_keys": [
         "n"
      ],
      "key": "n",
      "used_key_parts": [
         "Name"
      ],
      "key_length": "35",
      "ref": [
         "const"
      ],
      "rows_examined_per_scan": 1,
      "rows_produced_per_join": 1,
      "filtered": "100.00",
      "cost_info": {
         "read_cost": "1.00",
         "eval_cost": "1.00",
         "prefix_cost": "2.00",
         "data_read_per_join": "72"
      },
      "used_columns": [
         "Name",
         "CountryCode"
      ]
   }
   }
   }
]
}
}
```

## INサブクエリー(結果がユニークな場合)

例15は、サブクエリーで返される行が主キーであるため、結果が一意で重複がない(ユニークな)ことが保証されます。これによって、サブクエリーは`INNER JOIN`クエリーに安全に変換され、同じ結果を返すことが出来ます。`EXPLAIN`を実行した後に`SHOW WARNINGS `を実行すれば実際に実行されているクエリーが確認できます。

```sql
/* select#1 */ select `world`.`City`.`ID` AS `ID`,`world`.`City`.`Name` AS `Name`,`world`.`City`.`CountryCode` AS `CountryCode`,`world`.`City`.`District` AS `District`,`world`.`City`.`Population` AS `Population` from `world`.`Country` join `world`.`City` where ((`world`.`City`.`CountryCode` = `world`.`Country`.`Code`) and (`world`.`Country`.`Continent` = 'Asia'))
1 row in set (0.00 sec)
```

このサブクエリーはまずまず効率的です。Countryテーブルに最初にアクセス(カバリングインデックスを利用し、インデックスオンリースキャンを利用)し、一致した各行に対して一連の行がCityテーブル内を`c(CountryCode)`インデックスを利用して参照されます。

![explain-example-15](http://www.unofficialmysqlguide.com/_images/explain-example-15.png)

### 例15: 変換可能なINサブクエリー

```
EXPLAIN FORMAT=JSON
SELECT * FROM City WHERE CountryCode IN (SELECT Code FROM Country WHERE Continent = 'Asia');
{
 "query_block": {
   "select_id": 1,
   "cost_info": {
   "query_cost": "1893.30"
   },
   "nested_loop": [
   {
      "table": {
      "table_name": "Country",
      "access_type": "ref",
      "possible_keys": [
         "PRIMARY",
         "c"
      ],
      "key": "c",
      "used_key_parts": [
         "Continent"
      ],
      "key_length": "1",
      "ref": [
         "const"
      ],
      "rows_examined_per_scan": 51,
      "rows_produced_per_join": 51,
      "filtered": "100.00",
      "using_index": true,
      "cost_info": {
         "read_cost": "1.02",
         "eval_cost": "51.00",
         "prefix_cost": "52.02",
         "data_read_per_join": "13K"
      },
      "used_columns": [
         "Code",
         "Continent"
      ]
      }
   },
   {
      "table": {
      "table_name": "City",
      "access_type": "ref",
      "possible_keys": [
         "CountryCode"
      ],
      "key": "CountryCode",
      "used_key_parts": [
         "CountryCode"
      ],
      "key_length": "3",
      "ref": [
         "world.Country.Code"
      ],
      "rows_examined_per_scan": 18,
      "rows_produced_per_join": 920,
      "filtered": "100.00",
      "cost_info": {
         "read_cost": "920.64",
         "eval_cost": "920.64",
         "prefix_cost": "1893.30",
         "data_read_per_join": "64K"
      },
      "used_columns": [
         "ID",
         "Name",
         "CountryCode",
         "District",
         "Population"
      ]
      }
   }
   ]
 }
}
```

## INサブクエリー(ユニークでない場合)

例15では、サブクエリーは一意で重複がない(ユニークな)行セットを返すため`INNER JOIN`に書き換えられていました。サブクエリーがユニークではないとき、MySQLのオプティマイザーは別の戦略を選択します。

例16では、少なくとも1つ公用語を持つ国を検索しようとしています。幾つかの国では複数の公用語があるため、サブクエリーの結果はユニークにはなりません。

![explain-example-16](http://www.unofficialmysqlguide.com/_images/explain-example-16.png)

 `OPTIMIZER_TRACE`からの出力(例17)ではクエリーが直接joinに書き換えられないとオプティマイザーが判断し、代わりにsemi-join(準結合)を要求しています。オプティマイザーには複数の準結合戦略(FirstMatch, MaterializeLookup, DuplicatedWeedout)があります。このケースでは実体化(一時的な結果を保存するためにバッファーを生成)が最もこのクエリーを実行するためのコストが低いと結論づけています。

### 例16: INNER JOINで書き換えられないサブクエリー

```sql
EXPLAIN FORMAT=JSON
SELECT * FROM Country WHERE Code IN (SELECT CountryCode FROM CountryLanguage WHERE isOfficial=1);
{
 "query_block": {
   "select_id": 1,
   "cost_info": {
     "query_cost": "407.80"
   },
   "nested_loop": [
     {
       "table": {
         "table_name": "Country",
         "access_type": "ALL",
         "possible_keys": [
           "PRIMARY"
         ],
         "rows_examined_per_scan": 239,
         "rows_produced_per_join": 239,
         "filtered": "100.00",
         "cost_info": {
           "read_cost": "9.00",
           "eval_cost": "47.80",
           "prefix_cost": "56.80",
           "data_read_per_join": "61K"
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
         ],
         "attached_condition": "(`world`.`Country`.`Code` is not null)"
       }
     },
     {
       "table": {
         "table_name": "<subquery2>",
         "access_type": "eq_ref",
         "key": "<auto_key>",
         "key_length": "3",
         "ref": [
           "world.Country.Code"
         ],
         "rows_examined_per_scan": 1,
         "materialized_from_subquery": {
           "using_temporary_table": true,
           "query_block": {
             "table": {
               "table_name": "CountryLanguage",
               "access_type": "ALL",
               "possible_keys": [
                 "PRIMARY",
                 "CountryCode"
               ],
               "rows_examined_per_scan": 984,
               "rows_produced_per_join": 492,
               "filtered": "50.00",
               "cost_info": {
                 "read_cost": "104.40",
                 "eval_cost": "98.40",
                 "prefix_cost": "202.80",
                 "data_read_per_join": "19K"
               },
               "used_columns": [
                 "CountryCode",
                 "IsOfficial"
               ],
               "attached_condition": "(`world`.`CountryLanguage`.`IsOfficial` = 1)"
             }
           }
         }
       }
     }
   ]
 }
}
```

### 例17: サブクエリーの準結合戦略を分析する

```sql
SET OPTIMIZER_TRACE="enabled=on";
SET optimizer_trace_max_mem_size = 1024 * 1024;
SELECT * FROM Country WHERE Code IN (SELECT CountryCode FROM CountryLanguage WHERE isOfficial=1);
SELECT * FROM information_schema.optimizer_trace;
{
 "steps": [
   {
     "join_preparation": {
       "select#": 1,
       "steps": [
         {
           "join_preparation": {
             "select#": 2,
             "steps": [
               {
                 "expanded_query": "/* select#2 */ select `CountryLanguage`.`CountryCode` from `CountryLanguage` where (`CountryLanguage`.`IsOfficial` = 1)"
               },
               {
                 "transformation": {
                   "select#": 2,
                   "from": "IN (SELECT)",
                   "to": "semijoin",
                   "chosen": true
                 }
               }
             ]
           }
         },
         {
           "expanded_query": "/* select#1 */ select `Country`.`Code` AS `Code`,`Country`.`Name` AS `Name`,`Country`.`Continent` AS `Continent`,`Country`.`Region` AS `Region`,`Country`.`SurfaceArea` AS `SurfaceArea`,`Country`.`IndepYear` AS `IndepYear`,`Country`.`Population` AS `Population`,`Country`.`LifeExpectancy` AS `LifeExpectancy`,`Country`.`GNP` AS `GNP`,`Country`.`GNPOld` AS `GNPOld`,`Country`.`LocalName` AS `LocalName`,`Country`.`GovernmentForm` AS `GovernmentForm`,`Country`.`HeadOfState` AS `HeadOfState`,`Country`.`Capital` AS `Capital`,`Country`.`Code2` AS `Code2` from `Country` where `Country`.`Code` in (/* select#2 */ select `CountryLanguage`.`CountryCode` from `CountryLanguage` where (`CountryLanguage`.`IsOfficial` = 1))"
         },
         {
           "transformation": {
             "select#": 2,
             "from": "IN (SELECT)",
             "to": "semijoin",
             "chosen": true,
             "evaluating_constant_semijoin_conditions": [
             ]
           }
         },
         {
           "transformations_to_nested_joins": {
             "transformations": [
               "semijoin"
             ],
             "expanded_query": "/* select#1 */ select `Country`.`Code` AS `Code`,`Country`.`Name` AS `Name`,`Country`.`Continent` AS `Continent`,`Country`.`Region` AS `Region`,`Country`.`SurfaceArea` AS `SurfaceArea`,`Country`.`IndepYear` AS `IndepYear`,`Country`.`Population` AS `Population`,`Country`.`LifeExpectancy` AS `LifeExpectancy`,`Country`.`GNP` AS `GNP`,`Country`.`GNPOld` AS `GNPOld`,`Country`.`LocalName` AS `LocalName`,`Country`.`GovernmentForm` AS `GovernmentForm`,`Country`.`HeadOfState` AS `HeadOfState`,`Country`.`Capital` AS `Capital`,`Country`.`Code2` AS `Code2` from `Country` semi join (`CountryLanguage`) where (1 and (`CountryLanguage`.`IsOfficial` = 1) and (`Country`.`Code` = `CountryLanguage`.`CountryCode`))"
           }
         }
       ]
     }
   },
   {
     "join_optimization": {
       "select#": 1,
       "steps": [
         {
           "condition_processing": {
             "condition": "WHERE",
             "original_condition": "(1 and (`CountryLanguage`.`IsOfficial` = 1) and (`Country`.`Code` = `CountryLanguage`.`CountryCode`))",
             "steps": [
               {
                 "transformation": "equality_propagation",
                 "resulting_condition": "(1 and (`CountryLanguage`.`IsOfficial` = 1) and multiple equal(`Country`.`Code`, `CountryLanguage`.`CountryCode`))"
               },
               {
                 "transformation": "constant_propagation",
                 "resulting_condition": "(1 and (`CountryLanguage`.`IsOfficial` = 1) and multiple equal(`Country`.`Code`, `CountryLanguage`.`CountryCode`))"
               },
               {
                 "transformation": "trivial_condition_removal",
                 "resulting_condition": "((`CountryLanguage`.`IsOfficial` = 1) and multiple equal(`Country`.`Code`, `CountryLanguage`.`CountryCode`))"
               }
             ]
           }
         },
         {
           "substitute_generated_columns": {
           }
         },
         {
           "table_dependencies": [
             {
               "table": "`Country`",
               "row_may_be_null": false,
               "map_bit": 0,
               "depends_on_map_bits": [
               ]
             },
             {
               "table": "`CountryLanguage`",
               "row_may_be_null": false,
               "map_bit": 1,
               "depends_on_map_bits": [
               ]
             }
           ]
         },
         {
           "ref_optimizer_key_uses": [
             {
               "table": "`Country`",
               "field": "Code",
               "equals": "`CountryLanguage`.`CountryCode`",
               "null_rejecting": false
             },
             {
               "table": "`CountryLanguage`",
               "field": "CountryCode",
               "equals": "`Country`.`Code`",
               "null_rejecting": false
             },
             {
               "table": "`CountryLanguage`",
               "field": "CountryCode",
               "equals": "`Country`.`Code`",
               "null_rejecting": false
             }
           ]
         },
         {
           "pulled_out_semijoin_tables": [
           ]
         },
         {
           "rows_estimation": [
             {
               "table": "`Country`",
               "table_scan": {
                 "rows": 239,
                 "cost": 9
               }
             },
             {
               "table": "`CountryLanguage`",
               "table_scan": {
                 "rows": 984,
                 "cost": 6
               }
             }
           ]
         },
         {
           "execution_plan_for_potential_materialization": {
             "steps": [
               {
                 "considered_execution_plans": [
                   {
                     "plan_prefix": [
                     ],
                     "table": "`CountryLanguage`",
                     "best_access_path": {
                       "considered_access_paths": [
                         {
                           "access_type": "ref",
                           "index": "PRIMARY",
                           "usable": false,
                           "chosen": false
                         },
                         {
                           "access_type": "ref",
                           "index": "CountryCode",
                           "usable": false,
                           "chosen": false
                         },
                         {
                           "rows_to_scan": 984,
                           "access_type": "scan",
                           "resulting_rows": 492,
                           "cost": 202.8,
                           "chosen": true
                         }
                       ]
                     },
                     "condition_filtering_pct": 100,
                     "rows_for_plan": 492,
                     "cost_for_plan": 202.8,
                     "chosen": true
                   }
                 ]
               }
             ]
           }
         },
         {
           "considered_execution_plans": [
             {
               "plan_prefix": [
               ],
               "table": "`Country`",
               "best_access_path": {
                 "considered_access_paths": [
                   {
                     "access_type": "ref",
                     "index": "PRIMARY",
                     "usable": false,
                     "chosen": false
                   },
                   {
                     "rows_to_scan": 239,
                     "access_type": "scan",
                     "resulting_rows": 239,
                     "cost": 56.8,
                     "chosen": true
                   }
                 ]
               },
               "condition_filtering_pct": 100,
               "rows_for_plan": 239,
               "cost_for_plan": 56.8,
               "semijoin_strategy_choice": [
               ],
               "rest_of_plan": [
                 {
                   "plan_prefix": [
                     "`Country`"
                   ],
                   "table": "`CountryLanguage`",
                   "best_access_path": {
                     "considered_access_paths": [
                       {
                         "access_type": "ref",
                         "index": "PRIMARY",
                         "rows": 4.2232,
                         "cost": 442.83,
                         "chosen": true
                       },
                       {
                         "access_type": "ref",
                         "index": "CountryCode",
                         "rows": 4.2232,
                         "cost": 1211.2,
                         "chosen": false
                       },
                       {
                         "rows_to_scan": 984,
                         "access_type": "scan",
                         "using_join_cache": true,
                         "buffers_needed": 1,
                         "resulting_rows": 492,
                         "cost": 23647,
                         "chosen": false
                       }
                     ]
                   },
                   "condition_filtering_pct": 50,
                   "rows_for_plan": 504.67,
                   "cost_for_plan": 499.63,
                   "semijoin_strategy_choice": [
                     {
                       "strategy": "FirstMatch",
                       "recalculate_access_paths_and_cost": {
                         "tables": [
                         ]
                       },
                       "cost": 499.63,
                       "rows": 239,
                       "chosen": true
                     },
                     {
                       "strategy": "MaterializeLookup",
                       "cost": 407.8,    # 実体化が最低コスト
                       "rows": 239,
                       "duplicate_tables_left": false,
                       "chosen": true
                     },
                     {
                       "strategy": "DuplicatesWeedout",
                       "cost": 650.36,
                       "rows": 239,
                       "duplicate_tables_left": false,
                       "chosen": false
                     }
                   ],
                   "chosen": true
                 }
               ]
             },
             {
               "plan_prefix": [
               ],
               "table": "`CountryLanguage`",
               "best_access_path": {
                 "considered_access_paths": [
                   {
                     "access_type": "ref",
                     "index": "PRIMARY",
                     "usable": false,
                     "chosen": false
                   },
                   {
                     "access_type": "ref",
                     "index": "CountryCode",
                     "usable": false,
                     "chosen": false
                   },
                   {
                     "rows_to_scan": 984,
                     "access_type": "scan",
                     "resulting_rows": 492,
                     "cost": 202.8,
                     "chosen": true
                   }
                 ]
               },
               "condition_filtering_pct": 100,
               "rows_for_plan": 492,
               "cost_for_plan": 202.8,
               "semijoin_strategy_choice": [
                 {
                   "strategy": "MaterializeScan",
                   "choice": "deferred"
                 }
               ],
               "rest_of_plan": [
                 {
                   "plan_prefix": [
                     "`CountryLanguage`"
                   ],
                   "table": "`Country`",
                   "best_access_path": {
                     "considered_access_paths": [
                       {
                         "access_type": "ref",
                         "index": "PRIMARY",
                         "rows": 1,
                         "cost": 590.4,
                         "chosen": true
                       },
                       {
                         "rows_to_scan": 239,
                         "access_type": "scan",
                         "using_join_cache": true,
                         "buffers_needed": 1,
                         "resulting_rows": 239,
                         "cost": 23527,
                         "chosen": false
                       }
                     ]
                   },
                   "condition_filtering_pct": 100,
                   "rows_for_plan": 492,
                   "cost_for_plan": 793.2,
                   "semijoin_strategy_choice": [
                     {
                       "strategy": "LooseScan",
                       "recalculate_access_paths_and_cost": {
                         "tables": [
                           {
                             "table": "`CountryLanguage`",
                             "best_access_path": {
                               "considered_access_paths": [
                                 {
                                   "access_type": "ref",
                                   "index": "PRIMARY",
                                   "usable": false,
                                   "chosen": false
                                 },
                                 {
                                   "access_type": "ref",
                                   "index": "CountryCode",
                                   "usable": false,
                                   "chosen": false
                                 },
                                 {
                                   "rows_to_scan": 984,
                                   "access_type": "scan",
                                   "resulting_rows": 492,
                                   "cost": 202.8,
                                   "chosen": true
                                 }
                               ]
                             },
                             "unknown_key_1": {
                               "searching_loose_scan_index": {
                                 "indexes": [
                                   {
                                     "index": "PRIMARY",
                                     "ref_possible": false,
                                     "covering_scan_possible": false
                                   },
                                   {
                                     "index": "CountryCode",
                                     "ref_possible": false,
                                     "covering_scan_possible": false
                                   }
                                 ]
                               }
                             }
                           }
                         ]
                       },
                       "chosen": false
                     },
                     {
                       "strategy": "MaterializeScan",
                       "recalculate_access_paths_and_cost": {
                         "tables": [
                           {
                             "table": "`Country`",
                             "best_access_path": {
                               "considered_access_paths": [
                                 {
                                   "access_type": "ref",
                                   "index": "PRIMARY",
                                   "rows": 1,
                                   "cost": 590.4,
                                   "chosen": true
                                 },
                                 {
                                   "rows_to_scan": 239,
                                   "access_type": "scan",
                                   "using_join_cache": true,
                                   "buffers_needed": 1,
                                   "resulting_rows": 239,
                                   "cost": 23527,
                                   "chosen": false
                                 }
                               ]
                             }
                           }
                         ]
                       },
                       "cost": 992,
                       "rows": 1,
                       "duplicate_tables_left": true,
                       "chosen": true
                     },
                     {
                       "strategy": "DuplicatesWeedout",
                       "cost": 941.4,
                       "rows": 239,
                       "duplicate_tables_left": false,
                       "chosen": true
                     }
                   ],
                   "pruned_by_cost": true
                 }
               ]
             },
             { # 実体化が最終的に選択される
               "final_semijoin_strategy": "MaterializeLookup"
             }
           ]
         },
         {
           "creating_tmp_table": {
             "tmp_table_info": {
               "row_length": 4,
               "key_length": 3,
               "unique_constraint": false,
               "location": "memory (heap)",
               "row_limit_estimate": 4194304
             }
           }
         },
         {
           "attaching_conditions_to_tables": {
             "original_condition": "((`<subquery2>`.`CountryCode` = `Country`.`Code`) and (`CountryLanguage`.`IsOfficial` = 1))",
             "attached_conditions_computation": [
               {
                 "table": "`CountryLanguage`",
                 "rechecking_index_usage": {
                   "recheck_reason": "not_first_table",
                   "range_analysis": {
                     "table_scan": {
                       "rows": 984,
                       "cost": 204.9
                     },
                     "potential_range_indexes": [
                       {
                         "index": "PRIMARY",
                         "usable": true,
                         "key_parts": [
                           "CountryCode",
                           "Language"
                         ]
                       },
                       {
                         "index": "CountryCode",
                         "usable": true,
                         "key_parts": [
                           "CountryCode",
                           "Language"
                         ]
                       }
                     ],
                     "setup_range_conditions": [
                     ],
                     "group_index_range": {
                       "chosen": false,
                       "cause": "not_single_table"
                     }
                   }
                 }
               }
             ],
             "attached_conditions_summary": [
               {
                 "table": "`Country`",
                 "attached": "(`Country`.`Code` is not null)"
               },
               {
                 "table": "``.`<subquery2>`",
                 "attached": null
               },
               {
                 "table": "`CountryLanguage`",
                 "attached": "(`CountryLanguage`.`IsOfficial` = 1)"
               }
             ]
           }
         },
         {
           "refine_plan": [
             {
               "table": "`Country`"
             },
             {
               "table": "``.`<subquery2>`"
             },
             {
               "table": "`CountryLanguage`"
             }
           ]
         }
       ]
     }
   },
   {
     "join_execution": {
       "select#": 1,
       "steps": [
       ]
     }
   }
 ]
}
```

## NOT INサブクエリー

NOT INサブクエリーは実体化、あるいはEXISTS戦略をとることができます。2つの違いをみるために、次の2つの例について考えてみましょう。

1. `SELECT * FROM City WHERE CountryCode NOT IN (SELECT code FROM Country);`
2. `SELECT * FROM City WHERE CountryCode NOT IN (SELECT code FROM Country WHERE continent IN ('Asia', 'Europe', 'North America'));`

最初のクエリーでは、内部クエリーは最適な形です。code列はテーブルの主キーで、インデックススキャンで重複のないユニーク値が処理されます。唯一の否定的な側面は(もしあるとすれば)、主キーのインデックススキャンは、主キーは行の他の列の値も保持しているため、セカンダリキーのインデックススキャンより遅いかもしれないことです。`EXPLAIN`を実行すれば、オプティマイザーはこのクエリーに対しては実体化戦略を選択したことがわかりますが、次のヒントによって上書きすることもできます。

```sql
SELECT * FROM City WHERE CountryCode NOT IN (SELECT /*+ SUBQUERY(INTOEXISTS) */ code FROM Country);
```

2番目のクエリーには、追加の述部(`continent IN ('Asia', 'Europe', 'North America')`)があります。`City`テーブルの各行をNOT INクエリーと比較する必要があるため、一致した行の一覧を実体化しキャッシュすることに合理性がうまれます。

![explain-example-18](http://www.unofficialmysqlguide.com/_images/explain-example-18.png)

###  実体化を利用するNOT INサブクエリー

```sql
EXPLAIN FORMAT=JSON
SELECT * FROM City WHERE CountryCode NOT IN (SELECT code FROM Country WHERE continent IN ('Asia', 'Europe', 'North America'));
{
 "query_block": {
   "select_id": 1,
   "cost_info": {
     "query_cost": "862.60"
   },
   "table": {
     "table_name": "City",
     "access_type": "ALL",
     "rows_examined_per_scan": 4188,
     "rows_produced_per_join": 4188,
     "filtered": "100.00",
     "cost_info": {
       "read_cost": "25.00",
       "eval_cost": "837.60",
       "prefix_cost": "862.60",
       "data_read_per_join": "294K"
     },
     "used_columns": [
       "ID",
       "Name",
       "CountryCode",
       "District",
       "Population"
     ],
     "attached_condition": "(not(<in_optimizer>(`world`.`City`.`CountryCode`,`world`.`City`.`CountryCode` in ( <materialize> (/* select#2 */ select `world`.`Country`.`Code` from `world`.`Country` where (`world`.`Country`.`Continent` in ('Asia','Europe','North America')) ), <primary_index_lookup>(`world`.`City`.`CountryCode` in <temporary table> on <auto_key> where ((`world`.`City`.`CountryCode` = `materialized-subquery`.`code`)))))))",
     "attached_subqueries": [
       {
         "table": {
           "table_name": "<materialized_subquery>",
           "access_type": "eq_ref",
           "key": "<auto_key>",
           "key_length": "3",
           "rows_examined_per_scan": 1,
           "materialized_from_subquery": {
             "using_temporary_table": true,
             "dependent": true,
             "cacheable": false,
             "query_block": {
               "select_id": 2,
               "cost_info": {
                 "query_cost": "54.67"
               },
               "table": {
                 "table_name": "Country",
                 "access_type": "range",
                 "possible_keys": [
                   "PRIMARY",
                   "c",
                   "c_p"
                 ],
                 "key": "c",
                 "used_key_parts": [
                   "Continent"
                 ],
                 "key_length": "1",
                 "rows_examined_per_scan": 134,
                 "rows_produced_per_join": 134,
                 "filtered": "100.00",
                 "using_index": true,
                 "cost_info": {
                   "read_cost": "27.87",
                   "eval_cost": "26.80",
                   "prefix_cost": "54.67",
                   "data_read_per_join": "34K"
                 },
                 "used_columns": [
                   "Code",
                   "Continent"
                 ],
                 "attached_condition": "(`world`.`Country`.`Continent` in ('Asia','Europe','North America'))"
               }
             }
           }
         }
       }
     ]
   }
 }
}
```

## 派生テーブル(Derived Table)

from句内のサブクエリーは実体化する必要はありません。MySQLは通例、ビューが自身の定義をクエリーで利用する述部にマージするのと同じように、from句内のサブクエリーを「マージ」し直します。例19により、派生クエリーが外部クエリーにマージされているのがわかります。

![explain-example-19](http://www.unofficialmysqlguide.com/_images/explain-example-19.png)

### 例19: 派生テーブルが「マージ」し直される

```sql
EXPLAIN FORMAT=JSON
SELECT * FROM Country, (SELECT * FROM City WHERE CountryCode ='CAN' ) as CityTmp WHERE Country.code=CityTmp.CountryCode
AND CityTmp.name ='Toronto';
{
 "query_block": {
   "select_id": 1,
   "cost_info": {
   "query_cost": "2.00"
   },
   "nested_loop": [
   {
      "table": {
      "table_name": "Country",
      "access_type": "const",
      "possible_keys": [
         "PRIMARY"
      ],
      "key": "PRIMARY",
      "used_key_parts": [
         "Code"
      ],
      "key_length": "3",
      "ref": [
         "const"
      ],
      "rows_examined_per_scan": 1,
      "rows_produced_per_join": 1,
      "filtered": "100.00",
      "cost_info": {
         "read_cost": "0.00",
         "eval_cost": "1.00",
         "prefix_cost": "0.00",
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
   },
   {
      "table": {
      "table_name": "City",
      "access_type": "ref",
      "possible_keys": [
         "CountryCode",
         "n"
      ],
      "key": "n",
      "used_key_parts": [
         "Name"
      ],
      "key_length": "35",
      "ref": [
         "const"
      ],
      "rows_examined_per_scan": 1,
      "rows_produced_per_join": 0,
      "filtered": "5.00",
      "cost_info": {
         "read_cost": "1.00",
         "eval_cost": "0.05",
         "prefix_cost": "2.00",
         "data_read_per_join": "3"
      },
      "used_columns": [
         "ID",
         "Name",
         "CountryCode",
         "District",
         "Population"
      ],
      "attached_condition": "(`world`.`City`.`CountryCode` = 'CAN')"
      }
   }
   ]
 }
}
```

この「マージ」の否定的な側面はいくつかのクエリーが正当(legal)なものでなくなることです。
MySQLサーバーをアップグレードした際にwarningが発生する場合は、`derived_merge`最適化を無効化することができます。実体化は高コストとなりうるため、クエリーのコストが結果的に増えることになることもあるでしょう。


![explain-example-20](http://www.unofficialmysqlguide.com/_images/explain-example-20.png)

### 例20:  マージを無効化し実体化させる

```sql
SET optimizer_switch='derived_merge=off';
EXPLAIN FORMAT=JSON
SELECT * FROM Country, (SELECT * FROM City WHERE CountryCode ='CAN' ) as CityTmp WHERE Country.code=CityTmp.CountryCode
AND CityTmp.name ='Toronto';
{
  "query_block": {
   "select_id": 1,
   "cost_info": {
   "query_cost": "19.90"
   },
   "nested_loop": [
   {
      "table": {
         "table_name": "CityTmp",
         "access_type": "ref",
         "possible_keys": [
         "<auto_key0>"
         ],
         "key": "<auto_key0>",
         "used_key_parts": [
         "Name"
         ],
         "key_length": "35",
         "ref": [
         "const"
         ],
         "rows_examined_per_scan": 5,
         "rows_produced_per_join": 5,
         "filtered": "100.00",
         "cost_info": {
         "read_cost": "4.90",
         "eval_cost": "5.00",
         "prefix_cost": "9.90",
         "data_read_per_join": "360"
         },
         "used_columns": [
         "ID",
         "Name",
         "CountryCode",
         "District",
         "Population"
         ],
         "materialized_from_subquery": {
         "using_temporary_table": true,
         "dependent": false,
         "cacheable": true,
         "query_block": {
            "select_id": 2,
            "cost_info": {
               "query_cost": "98.00"
            },
            "table": {
               "table_name": "City",
               "access_type": "ref",
               "possible_keys": [
               "CountryCode"
               ],
               "key": "CountryCode",
               "used_key_parts": [
               "CountryCode"
               ],
               "key_length": "3",
               "ref": [
               "const"
               ],
               "rows_examined_per_scan": 49,
               "rows_produced_per_join": 49,
               "filtered": "100.00",
               "cost_info": {
               "read_cost": "49.00",
               "eval_cost": "49.00",
               "prefix_cost": "98.00",
               "data_read_per_join": "3K"
               },
               "used_columns": [
               "ID",
               "Name",
               "CountryCode",
               "District",
               "Population"
               ]
            }
         }
         }
      }
   },
   {
      "table": {
         "table_name": "Country",
         "access_type": "eq_ref",
         "possible_keys": [
         "PRIMARY"
         ],
         "key": "PRIMARY",
         "used_key_parts": [
         "Code"
         ],
         "key_length": "3",
         "ref": [
         "CityTmp.CountryCode"
         ],
         "rows_examined_per_scan": 1,
         "rows_produced_per_join": 5,
         "filtered": "100.00",
         "cost_info": {
         "read_cost": "5.00",
         "eval_cost": "5.00",
         "prefix_cost": "19.90",
         "data_read_per_join": "1K"
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
   ]
  }
}
```
