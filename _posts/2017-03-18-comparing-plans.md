---
title: プランの比較
date: 2017-03-18 00:00:01 -0900
article_index: 9
original_url: http://www.unofficialmysqlguide.com/comparing-plans.html
translator: taka-h (@takaidohigasi)
---

簡単な復習: オプティマイザの役割はたくさんの選択肢の中から最良の実行計画を選択することです。これらの選択肢には、それぞれいくつかの異なるインデックスや、アクセス方法があります。ここまで、われわれは`p(population)`に対する2つの実行計画を見てきました。

1.  `p(population)`に対するレンジスキャン
2. テーブルスキャン

`p(population)`インデックスは選択性が高くないことがわかったため、`c(continent)`に対して新しいインデックスを追加しようと思います。一般的な本番環境では、`p(population)`インデックスは意味を持たなくなるため削除したくなることでしょう。しかし、オプティマイザが複数の選択肢をうまく評価できたことを確認するためにこれを残すこととします。

![explain-c.png](http://www.unofficialmysqlguide.com/_images/explain-c.png)

### 例7) continent(大陸)にインデックスを追加する

```sql
ALTER TABLE Country ADD INDEX c (continent);
EXPLAIN FORMAT=JSON
SELECT * FROM Country WHERE population > 5000000 continent='Asia';
{
  "query_block": {
     "select_id": 1,
     "cost_info": {
     "query_cost": "28.20"
     },
     "table": {
     "table_name": "Country",
     "access_type": "ref",       # このアクセス方法 ref は
     "possible_keys": [          # ユニークでないインデックスを利用していることを表します
     "p",
     "c"
     ],
     "key": "c",                 # continentインデックスが選択されています
     "used_key_parts": [
     "Continent"
     ],
     "key_length": "1",
     "ref": [
     "const"
     ],
     "rows_examined_per_scan": 51,
     "rows_produced_per_join": 23,
     "filtered": "45.19",
     "cost_info": {
     "read_cost": "18.00",
     "eval_cost": "4.61",
     "prefix_cost": "28.20",
     "data_read_per_join": "5K"
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
     "attached_condition": "(`world`.`Country`.`Population` > 5000000)"
     }
  }
}
```

例7では、`c(continent)`インデックスが`p(population)`インデックスおよびテーブルスキャンよりも優先されていることがわかります。これでジョブはインデックスを利用しますが、これは作業量を減らすためです。オプティマイザはインデックスが利用されたあとに、51行(`rows_examined_per_scan`)のみを検査すれば良いと見積もってます。Asiaの大陸(continent)には国(country)の数が少ないので、インデックスを利用する方が選択性が高くて良いと判断された、とも解釈できます。

例8ではクエリーを少し修正し、クエリーの選択条件を >500万 から >5億 に変更することで、`p(population)`インデックスを選択するように変わったことがわかります。これは理にかなっており、人口(population) > 5億の国は2つしかないため、さらにインデックスの選択性が高くなるわけです。

![explain-example-8.png](http://www.unofficialmysqlguide.com/_images/explain-example-8.png)

### 例8: 人口が多い場合はpへのrangeアクセスがcに対するそれより優先される

```sql
EXPLAIN FORMAT=JSON
SELECT * FROM Country WHERE continent='Asia' and population > 500000000;
{
  "query_block": {
    "select_id": 1,
    "cost_info": {
      "query_cost": "7.01"
    },
    "table": {
      "table_name": "Country",
      "access_type": "range",    # refではなくrangeが利用される
      "possible_keys": [
        "p",
        "c"
      ],
      "key": "p",                # 例7とkeyが異なる
      "used_key_parts": [
        "Population"
      ],
      "key_length": "4",
      "rows_examined_per_scan": 2,
      "rows_produced_per_join": 0,
      "filtered": "21.34",
      "index_condition": "(`world`.`Country`.`Population` > 500000000)",
      "cost_info": {
        "read_cost": "6.58",
        "eval_cost": "0.43",
        "prefix_cost": "7.01",
        "data_read_per_join": "112"
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
      "attached_condition": "(`world`.`Country`.`Continent` = 'Asia')"
    }
  }
}
```

continentへインデックスを追加することで、少なくとも4つも実行計画をとりうるようになりました。

1. `p(population): "500000000 < Population"`へのレンジスキャン
2. テーブルスキャン
3. `c(continent)`へのrefによるアクセス
4. `c(continent): "Asia <= Continent <= Asia"`へのrangeによるアクセス

これらのプランに加え、インデックスマージを使って、`p(population)`と`c(continent)`を両方を使うこともできます。例9の`OPTIMIZER_TRACE`で、`PRIMARY`へのレンジスキャンも検討されてたものの、採用されなかったことがわかります。

`rows_examined_per_scan`がインデックスの選択性を示していることを説明しましたが、例7と例8の違いを説明するのために有用な2つの他の統計があります。

1. **Key_length** continentのデータ型は、populationのそれが4バイトであるのに対し、1バイトのenumです。選択性が同じ場合、キー長が短いほうがページ内に多くのキーを格納できるため有利であり、インデックスとしてもメモリーがうまく使えます。
2. **Access_type** すべての条件が同一であれば、ref によるアクセスは range によるアクセスより低コストです。

### 例9 人口が多い場合にインデックスpとcを比較したオプティマイザ トレース

```sql
SET optimizer_trace="enabled=on";
SELECT * FROM Country WHERE continent='Asia' AND population > 500000000;
SELECT * FROM information_schema.optimizer_trace;
{
  "steps": [
    {
      "join_preparation": {
        "select#": 1,
        "steps": [
          {
            "expanded_query": "/* select#1 */ select `Country`.`Code` AS `Code`,`Country`.`Name` AS `Name`,`Country`.`Continent` AS `Continent`,`Country`.`Region` AS `Region`,`Country`.`SurfaceArea` AS `SurfaceArea`,`Country`.`IndepYear` AS `IndepYear`,`Country`.`Population` AS `Population`,`Country`.`LifeExpectancy` AS `LifeExpectancy`,`Country`.`GNP` AS `GNP`,`Country`.`GNPOld` AS `GNPOld`,`Country`.`LocalName` AS `LocalName`,`Country`.`GovernmentForm` AS `GovernmentForm`,`Country`.`HeadOfState` AS `HeadOfState`,`Country`.`Capital` AS `Capital`,`Country`.`Code2` AS `Code2` from `Country` where ((`Country`.`Continent` = 'Asia') and (`Country`.`Population` > 500000000))"
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
              "original_condition": "((`Country`.`Continent` = 'Asia') and (`Country`.`Population` > 500000000))",
              "steps": [
                {
                  "transformation": "equality_propagation",
                  "resulting_condition": "((`Country`.`Population` > 500000000) and multiple equal('Asia', `Country`.`Continent`))"
                },
                {
                  "transformation": "constant_propagation",
                  "resulting_condition": "((`Country`.`Population` > 500000000) and multiple equal('Asia', `Country`.`Continent`))"
                },
                {
                  "transformation": "trivial_condition_removal",
                  "resulting_condition": "((`Country`.`Population` > 500000000) and multiple equal('Asia', `Country`.`Continent`))"
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
              }
            ]
          },
          {
            "ref_optimizer_key_uses": [
              {
                "table": "`Country`",
                "field": "Continent",
                "equals": "'Asia'",
                "null_rejecting": false
              }
            ]
          },
          {
            "rows_estimation": [
              {
                "table": "`Country`",
                "range_analysis": {
                  "table_scan": {
                    "rows": 239,
                    "cost": 247.1
                  },
                  "potential_range_indexes": [
                    {
                      "index": "PRIMARY",
                      "usable": false,            # 主キーへのレンジスキャンは
                      "cause": "not_applicable"   # 利用できない
                    },
                    {
                      "index": "p",
                      "usable": true,   # pへのレンジスキャンは利用可能
                      "key_parts": [    # "analyzing_range_alternatives"下で
                        "Population",   # 評価されている
                        "Code"
                      ]
                    },
                    {
                      "index": "c",     # cへのレンジスキャンは利用可能
                      "usable": true,   # "analyzing_range_alternatives"下で
                      "key_parts": [    # 評価されているが
                        "Continent",    # 高コストと見積もられている
                        "Code"
                      ]
                    }
                  ],
                  "setup_range_conditions": [
                  ],
                  "group_index_range": {
                    "chosen": false,
                    "cause": "not_group_by_or_distinct"
                  },
                  "analyzing_range_alternatives": {
                    "range_scan_alternatives": [
                      {
                        "index": "p",
                        "ranges": [
                          "500000000 < Population"
                        ],
                        "index_dives_for_eq_ranges": true,
                        "rowid_ordered": false,
                        "using_mrr": false,
                        "index_only": false,
                        "rows": 2,
                        "cost": 5.01,
                        "chosen": true
                      },
                      {
                        "index": "c",
                        "ranges": [
                          "Asia <= Continent <= Asia"
                        ],
                        "index_dives_for_eq_ranges": true,
                        "rowid_ordered": true,
                        "using_mrr": false,
                        "index_only": false,
                        "rows": 51,
                        "cost": 103.01,
                        "chosen": false,
                        "cause": "cost"
                      }
                    ],
                    "analyzing_roworder_intersect": {
                      "usable": false,                 # インデックスマージは利用しない
                      "cause": "too_few_roworder_scans"
                    }
                  },
                  "chosen_range_access_summary": {
                    "range_access_plan": {
                      "type": "range_scan",          # レンジオプティマイザは
                      "index": "p",                  # pへのrangeアクセスを選択
                      "rows": 2,
                      "ranges": [
                        "500000000 < Population"
                      ]
                    },
                    "rows_for_plan": 2,
                    "cost_for_plan": 5.01,
                    "chosen": true
                  }
                }
              }
            ]
          },
          {
            "considered_execution_plans": [
              {
                "plan_prefix": [
                ],
                "table": "`Country`",          # ここは最善のアクセス戦略の
                "best_access_path": {          # まとめ
                  "considered_access_paths": [ # (refおよびrangeオプティマイザより)
                    {
                      "access_type": "ref",    # cへのrefアクセス
                      "index": "c",
                      "rows": 51,
                      "cost": 69,
                      "chosen": true
                    },
                    {
                      "rows_to_scan": 2,
                      "access_type": "range",  # pへのrangeアクセス
                      "range_details": {
                        "used_index": "p"
                      },
                      "resulting_rows": 2,
                      "cost": 7.01,
                      "chosen": true
                    }
                  ]
                },
                "condition_filtering_pct": 100,
                "rows_for_plan": 2,
                "cost_for_plan": 7.01,
                "chosen": true
              }
            ]
          },
          {
            "attaching_conditions_to_tables": {
              "original_condition": "((`Country`.`Continent` = 'Asia') and (`Country`.`Population` > 500000000))",
              "attached_conditions_computation": [
              ],
              "attached_conditions_summary": [
                {
                  "table": "`Country`",
                  "attached": "((`Country`.`Continent` = 'Asia') and (`Country`.`Population` > 500000000))"
                }
              ]
            }
          },
          {
            "refine_plan": [
              {
                "table": "`Country`",
                "pushed_index_condition": "(`Country`.`Population` > 500000000)",
                "table_condition_attached": "(`Country`.`Continent` = 'Asia')"
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
