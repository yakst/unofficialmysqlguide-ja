---
title: 集約
date: 2017-06-25 00:00:01 -0900
article_index: 17
original_url: http://www.unofficialmysqlguide.com/aggregation.html
translator: taka-h (@takaidohigasi)
---

## GROUP BY

`GROUP BY`の操作をする際には、行がソート順に読み込まれるかあるいは一時テーブルが集約を行うための中間結果をバッファーする必要があります。言いかえれば、MySQLは`GROUP BY`で次のようにインデックスを利用することができます。

1. **ルースインデックススキャン**: `GROUP BY`条件にインデックスが付与されていれば、MySQLはインデックスを最初から最後までスキャンすることを選択し、中間的な結果の実体化を回避します。この操作は述部の選択性が非常に高く、作成される一時テーブルが非常に大きい、というわけではない場合には良い選択となります。
![loose-index-scan-continent-index](http://www.unofficialmysqlguide.com/_images/loose-index-scan-continent-index.png)
2. **行の抽出**: インデックスは対象行があるかどうかを判別するのに利用され、その結果は一時テーブルに保存されます。結果は一時テーブルに集約され、デフォルトでは`GROUP BY`条件によってソートされます。[^1]
![group-by-filtering-rows](http://www.unofficialmysqlguide.com/_images/group-by-filtering-rows.png)
3. **行の抽出とソートの両方**: この最適化はインデックスを利用し行を抽出した結果が`GROUP BY`操作における正しい順序となっている場合にこの最適化が適用されます。
![group-by-filtering-and-sort](http://www.unofficialmysqlguide.com/_images/group-by-filtering-and-sort.png)

### 例24: ルースインデックススキャンを利用する`GROUP BY`

```sql
EXPLAIN FORMAT=JSON
SELECT count(*) as c, continent FROM Country GROUP BY continent;
{
  "query_block": {
    "select_id": 1,
    "cost_info": {
      "query_cost": "56.80"
    },
    "grouping_operation": {
      "using_filesort": false,   # <--
      "table": {
        "table_name": "Country",
        "access_type": "index",  # <--
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
```

### 例25: インデックスを利用後ソートをする`GROUP BY`

```sql
EXPLAIN FORMAT=JSON
SELECT count(*) as c, continent FROM Country WHERE population > 500000000 GROUP BY continent;
{
  "query_block": {
    "select_id": 1,
    "cost_info": {
      "query_cost": "3.81"
    },
    "grouping_operation": {
      "using_temporary_table": true,   # <--
      "using_filesort": true,          # <--
      "cost_info": {
        "sort_cost": "2.00"
      },
      "table": {
        "table_name": "Country",
        "access_type": "range",        # <--
        "possible_keys": [
          "PRIMARY",
          "p",
          "c",
          "p_c",
          "c_p",
          "Name"
        ],
        "key": "p",
        "used_key_parts": [
          "Population"
        ],
        "key_length": "4",
        "rows_examined_per_scan": 2,
        "rows_produced_per_join": 2,
        "filtered": "100.00",
        "using_index": true,
        "cost_info": {
          "read_cost": "1.41",
          "eval_cost": "0.40",
          "prefix_cost": "1.81",
          "data_read_per_join": "528"
        },
        "used_columns": [
          "Code",
          "Continent",
          "Population"
        ],
        "attached_condition": "(`world`.`Country`.`Population` > 500000000)"
      }
    }
  }
}
```

### 例26: インデックスを行のソートと抽出に利用する`GROUP BY`

```sql
EXPLAIN FORMAT=JSON
SELECT count(*) as c, continent FROM Country WHERE continent='Asia' GROUP BY continent;
{
  "query_block": {
    "select_id": 1,
    "cost_info": {
      "query_cost": "11.23"
    },
    "grouping_operation": {
      "using_filesort": false,     # <--
      "table": {
        "table_name": "Country",
        "access_type": "ref",      # <--
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
        "ref": [
          "const"
        ],
        "rows_examined_per_scan": 51,
        "rows_produced_per_join": 51,
        "filtered": "100.00",
        "using_index": true,
        "cost_info": {
          "read_cost": "1.03",
          "eval_cost": "10.20",
          "prefix_cost": "11.23",
          "data_read_per_join": "13K"
        },
        "used_columns": [
          "Continent"
        ]
      }
    }
  }
}
```

## UNION

`UNION`は2つのクエリーの結果を結合し重複を削除するものですが、MySQLは`UNION`には特別な最適化は行いません。例27に示されるように、中間の一時テーブルにて重複排除が実行されます。一時テーブルは全ての`UNION`クエリーで利用されるため、コストが割当られません(コストベースの最適化はできません)。

### 簡単なUnionの例

```sql
SELECT * FROM City WHERE CountryCode = 'CAN'
UNION
SELECT * FROM City WHERE CountryCode = 'USA'
```

### 仮説上の最適化

```sql
SELECT * FROM City WHERE CountryCode IN ('CAN', 'USA')
```

![explain-example-29](http://www.unofficialmysqlguide.com/_images/explain-example-29.png)

サブクエリーやビューでは同一テーブルへの複数アクセスが内部で単一アクセスに統合されますが、他方でMySQLは`UNION`には同様の最適化を行いません。重複行は許容しないものの`UNION`が`UNION ALL`に書き換えられるケースに関する検討もまた実施しません。すなわち、スキルのある人が手動でクエリーを修正(アプリケーション内、あるいはクエリーリライトだったり)することで性能が向上できるケースが多数あることになります。

### 例27: 一時テーブルを必要とする`UNION`

```sql
EXPLAIN FORMAT=JSON
SELECT * FROM City WHERE CountryCode = 'CAN'
UNION
SELECT * FROM City WHERE CountryCode = 'USA'
{
  "query_block": {
    "union_result": {
      "using_temporary_table": true,  # 一時テーブルが必要
      "table_name": "<union1,2>",     # 2つのクエリーの結果を結合
      "access_type": "ALL",
      "query_specifications": [
        {
          "dependent": false,
          "cacheable": true,
          "query_block": {
            "select_id": 1,
            "cost_info": {
              "query_cost": "58.80"
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
                "eval_cost": "9.80",
                "prefix_cost": "58.80",
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
        },
        {
          "dependent": false,
          "cacheable": true,
          "query_block": {
            "select_id": 2,
            "cost_info": {
              "query_cost": "129.80"
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
              "rows_examined_per_scan": 274,
              "rows_produced_per_join": 274,
              "filtered": "100.00",
              "cost_info": {
                "read_cost": "75.00",
                "eval_cost": "54.80",
                "prefix_cost": "129.80",
                "data_read_per_join": "19K"
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
      ]
    }
  }
}
```


## UNION ALL

`UNION ALL`は意味上は`UNION`と似ていますが、重複排除が必要がないという重要な違いがあります。これによって、いくつかのケースでMySQLは`UNION ALL`の結果を中間テーブルで実体化したり重複排除したりすることなく、そのまま伝えることができることになります。

内部的には`UNION ALL`のクエリーには毎回一時テーブルが作成されますが、`EXPLAIN`で行の実体化に必要だったかどうかを確認できます。例28に`UNION ALL`を利用したクエリー例を示します。`ORDER BY`を追加するとクエリーは中間一時テーブルを利用する必要がでてきます。

![](http://www.unofficialmysqlguide.com/_images/explain-example-30.png)

### 例28: 一時テーブルを利用しない`UNION ALL`

```sql
EXPLAIN FORMAT=JSON
SELECT * FROM City WHERE CountryCode = 'CAN'
UNION ALL
SELECT * FROM City WHERE CountryCode = 'USA';
{
  "query_block": {
    "union_result": {
      "using_temporary_table": false,   # 一時テーブルは不要!
      "query_specifications": [
        {
          "dependent": false,
          "cacheable": true,
          "query_block": {
            "select_id": 1,
            "cost_info": {
              "query_cost": "58.80"
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
                "eval_cost": "9.80",
                "prefix_cost": "58.80",
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
        },
        {
          "dependent": false,
          "cacheable": true,
          "query_block": {
            "select_id": 2,
            "cost_info": {
              "query_cost": "129.80"
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
              "rows_examined_per_scan": 274,
              "rows_produced_per_join": 274,
              "filtered": "100.00",
              "cost_info": {
                "read_cost": "75.00",
                "eval_cost": "54.80",
                "prefix_cost": "129.80",
                "data_read_per_join": "19K"
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
      ]
    }
  }
}
```

### 例29: `ORDER BY`により一時テーブルが必要となる`UNION ALL`

```sql
EXPLAIN FORMAT=JSON
SELECT * FROM City WHERE CountryCode = 'CAN'
UNION ALL
SELECT * FROM City WHERE CountryCode = 'USA' ORDER BY Name;
{
  "query_block": {
    "union_result": {
      "using_temporary_table": true,  # UNION ALLは一時テーブルを必要とする
      "table_name": "<union1,2>",     # ORDER BYが指定されているため
      "access_type": "ALL",
      "query_specifications": [
        {
          "dependent": false,
          "cacheable": true,
          "query_block": {
            "select_id": 1,
            "cost_info": {
              "query_cost": "58.80"
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
                "eval_cost": "9.80",
                "prefix_cost": "58.80",
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
        },
        {
          "dependent": false,
          "cacheable": true,
          "query_block": {
            "select_id": 2,
            "cost_info": {
              "query_cost": "129.80"
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
              "rows_examined_per_scan": 274,
              "rows_produced_per_join": 274,
              "filtered": "100.00",
              "cost_info": {
                "read_cost": "75.00",
                "eval_cost": "54.80",
                "prefix_cost": "129.80",
                "data_read_per_join": "19K"
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
      ]
    }
  }
}
```

[^1]: この動作は廃止予定ですので将来的にはなくなります。順序の指定は必須ではありませんが明示的に`GROUP BY`に`ORDER BY NULL`を指定することが推奨されます。
