---
title: オプティマイザ トレース 
date: 2017-03-09 00:00:01 -0900
article_index: 5
original_url: http://www.unofficialmysqlguide.com/optimizer-trace.html
translator: taka-h (@takaidohigasi)
---

`EXPLAIN`はクエリーの実行計画以外は表示しません。すなわち、なぜ他の代替となる実行戦略が選ばれなかったのか、については表示しません。これを理解するためには次の観点を考える必要がありややこしいものです。

* 選ばれなかった実行計画は適切でなかったのか？(例えば、ある最適化は特定のユースケースのみに利用できる、など)
* 選ばれなかった実行計画の方が高コストと評価されたのか？
* 高コストだとすると、どの程度か？

`OPTIMIZER_TRACE` はこれらの疑問に答えます。オプティマイザ トレースは、オプティマイザのより詳細なデータを表示するように作られており、
オプティマイザのコストモデルがどのように動作するかを学ぶことはもとより、日々のトラブルシューティングにもとても役に立ちます。

### 例2) EXPLAINにより新しく追加されたインデックスが使われていないことがわかる

```sql
ALTER TABLE Country ADD INDEX p (population);
EXPLAIN FORMAT=JSON
SELECT * FROM Country WHERE continent='Asia' and population > 5000000;

{
  "query_block": {
   "select_id": 1,
   "cost_info": {
   "query_cost": "53.80"
   },
   "table": {
   "table_name": "Country",
   "access_type": "ALL",       # このクエリーはテーブルスキャンとして実行されている
   "possible_keys": [          # オプティマイザは
      "p"                      # インデックスが利用可能と判断しているのに!
   ],
   "rows_examined_per_scan": 239,
   "rows_produced_per_join": 15,
   "filtered": "6.46",
   "cost_info": {
      "read_cost": "50.71",
      "eval_cost": "3.09",
      "prefix_cost": "53.80",
      "data_read_per_join": "3K"
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
   "attached_condition": "((`world`.`Country`.`Continent` = 'Asia') and (`world`.`Country`.`Population` > 5000000))"
   }
  }
}
```

例2では、インデックス`p(population)`がテーブルに追加された後であるにもかかわらず、選択されていないことがわかります。`EXPLAIN`からこれが候補であったことはわかりますが、なぜ選択されなかったかはわかりません。この理由を知るためには、`OPTIMIZER_TRACE`を利用する必要があります。

### 例3: OPTIMIZER_TRACEによりインデックスが利用されなかった理由がわかる

```sql
SET optimizer_trace="enabled=on"
SELECT * FROM Country WHERE continent='Asia' and population > 5000000;
SELECT * FROM information_schema.optimizer_trace;
{
  "steps": [
   {
   "join_preparation": {
      "select#": 1,
      "steps": [
         {
         "expanded_query": "/* select#1 */ select `Country`.`Code` AS `Code`,`Country`.`Name` AS `Name`,`Country`.`Continent` AS `Continent`,`Country`.`Region` AS `Region`,`Country`.`SurfaceArea` AS `SurfaceArea`,`Country`.`IndepYear` AS `IndepYear`,`Country`.`Population` AS `Population`,`Country`.`LifeExpectancy` AS `LifeExpectancy`,`Country`.`GNP` AS `GNP`,`Country`.`GNPOld` AS `GNPOld`,`Country`.`LocalName` AS `LocalName`,`Country`.`GovernmentForm` AS `GovernmentForm`,`Country`.`HeadOfState` AS `HeadOfState`,`Country`.`Capital` AS `Capital`,`Country`.`Code2` AS `Code2` from `Country` where ((`Country`.`Continent` = 'Asia') and (`Country`.`Population` > 5000000))"
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
            "original_condition": "((`Country`.`Continent` = 'Asia') and (`Country`.`Population` > 5000000))",
            "steps": [
               {
               "transformation": "equality_propagation",
               "resulting_condition": "((`Country`.`Population` > 5000000) and multiple equal('Asia', `Country`.`Continent`))"
               },
               {
               "transformation": "constant_propagation",
               "resulting_condition": "((`Country`.`Population` > 5000000) and multiple equal('Asia', `Country`.`Continent`))"
               },
               {
               "transformation": "trivial_condition_removal",
               "resulting_condition": "((`Country`.`Population` > 5000000) and multiple equal('Asia', `Country`.`Continent`))"
               }
            ]
         }
         },
         {
         "substitute_generated_columns": {}
         },
         {
         "table_dependencies": [
            {
               "table": "`Country`",
               "row_may_be_null": false,
               "map_bit": 0,
               "depends_on_map_bits": []
            }
         ]
         },
         {
         "ref_optimizer_key_uses": []
         },
         {
         "rows_estimation": [
            {
               "table": "`Country`",
               "range_analysis": {
               "table_scan": {
                  "rows": 239,
                  "cost": 55.9
               },
               "potential_range_indexes": [
                  {
                     "index": "PRIMARY",
                     "usable": false,
                     "cause": "not_applicable"
                  },
                  {
                     "index": "p",
                     "usable": true,
                     "key_parts": [
                     "Population",
                     "Code"
                     ]
                  }
               ],
               "setup_range_conditions": [],
               "group_index_range": {
                  "chosen": false,
                  "cause": "not_group_by_or_distinct"
               },
               "analyzing_range_alternatives": {
                  "range_scan_alternatives": [
                     {
                     "index": "p",
                     "ranges": [
                        "5000000 < Population"
                     ],
                     "index_dives_for_eq_ranges": true,
                     "rowid_ordered": false,
                     "using_mrr": false,
                     "index_only": false,
                     "rows": 108,
                     "cost": 130.61,             # これはインデックスを利用した場合のコスト。
                     "chosen": false,            # この実行計画は選択されなかった。なぜならば、
                     "cause": "cost"             # テーブルスキャンより高コストであるから
                     }
                  ],
                  "analyzing_roworder_intersect": {
                     "usable": false,
                     "cause": "too_few_roworder_scans"
                  }
               }
               }
            }
         ]
         },
         {
         "considered_execution_plans": [
            {
               "plan_prefix": [],
               "table": "`Country`",
               "best_access_path": {
               "considered_access_paths": [
                  {
                     "rows_to_scan": 239,
                     "access_type": "scan",
                     "resulting_rows": 239,
                     "cost": 53.8,
                     "chosen": true
                  }
               ]
               },
               "condition_filtering_pct": 100,
               "rows_for_plan": 239,
               "cost_for_plan": 53.8,
               "chosen": true
            }
         ]
         },
         {
         "attaching_conditions_to_tables": {
            "original_condition": "((`Country`.`Continent` = 'Asia') and (`Country`.`Population` > 5000000))",
            "attached_conditions_computation": [],
            "attached_conditions_summary": [
               {
               "table": "`Country`",
               "attached": "((`Country`.`Continent` = 'Asia') and (`Country`.`Population` > 5000000))"
               }
            ]
         }
         },
         {
         "refine_plan": [
            {
               "table": "`Country`"
            }
         ]
         }
      ]
   }
   },
   {
   "join_execution": {
      "select#": 1,
      "steps": []
   }
   }
  ]
}
```


例3の`range_scan_alternatives` では`p(population)`は考慮されている一方で、コストに基いて選択肢から除外(`"chosen": false and "cause": "cost"`)されたことを示しています。出力にはインデックスを利用する場合のコスト見積もりも表示されており、130.61コストユニットとあります。これがテーブルスキャンの場合の55.9コストユニットと比較されます。値は小さいほうがよいため、テーブルスキャンの方が良いわけです。

なぜこのようになるかを説明するには、インデックスにはやることが多く、これを減らす必要があることを理解する必要があります。このデータセットの大多数の国の人口は500万人以上となります。オプティマイザがインデックスとデータを行ったりきたりするより、テーブルスキャンしたほうが速いと判断したことになります。上級ユーザーの方々は、この意思決定を行うためのコストを調整することができます。

![選択されないインデックス](http://www.unofficialmysqlguide.com/_images/non-selective-index.png)
