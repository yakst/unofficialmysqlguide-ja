---
title: JSONと生成列
date: 2017-07-09 00:00:01 -0900
article_index: 23
original_url: http://www.unofficialmysqlguide.com/query-rewrite.html
translator: taka-h (@takaidohigasi)
---

MySQLサーバーは次の機能追加によってスキーマレスなデータの保存をサポートしています。

1. **JSONデータ型**: JSONの値がINSERT/UPDATEの際にパースされ、バリデートされバイナリーの最適化された形式で保存されます。JSONデータは値を読み込む際には、パースやバリデートは一切必要ありません。
2. **JSON関数**: 20を超えるJSON値の検索、操作および生成に関するSQL関数があります。
3. **生成列(generated columns)**: これはJSONに限ったことではありませんが、生成列は*関数インデックス*のように動作し、これによってJSONドキュメントの一部を抽出したりあるいはインデックスを作成したりすることができます。

オプティマイザはJSONデータにクエリーを発行する場合、生成列から条件に一致するインデックスを自動的に探します[^1]。例25では、ユーザーの志向がJSON列に保存されています。初期状態では、更新があったらお知らせしてほしい(`notify_on_updates`)と希望しているユーザーを抽出するクエリーで、テーブルスキャンが行われています。インデックス付きの仮想生成列を追加することで、インデックスが利用できるようになったことが`EXPLAIN`によりわかります。

### 例35:  ユーザー志向のスキーマレスな表現

```sql
CREATE TABLE users (
  id INT NOT NULL auto_increment,
  username VARCHAR(32) NOT NULL,
  preferences JSON NOT NULL,
  PRIMARY KEY (id),
  UNIQUE (username)
);

INSERT INTO users
 (id,username,preferences)
VALUES
 (NULL, 'morgan', '{"layout": "horizontal", "warn_before_delete": false, "notify_on_updates": true}'),
 (NULL, 'wes', '{"layout": "horizontal", "warn_before_delete": false, "notify_on_updates": false}'),
 (NULL, 'jasper', '{"layout": "horizontal", "warn_before_delete": false, "notify_on_updates": false}'),
 (NULL, 'gus', '{"layout": "horizontal", "warn_before_delete": false, "notify_on_updates": false}'),
 (NULL, 'olive', '{"layout": "horizontal", "warn_before_delete": false, "notify_on_updates": false}');

EXPLAIN FORMAT=JSON
SELECT * FROM users WHERE preferences->"$.notify_on_updates" = true;
{
  "query_block": {
    "select_id": 1,
    "cost_info": {
      "query_cost": "2.00"
    },
    "table": {
      "table_name": "users",        # < --
      "access_type": "ALL",         # < --
      "rows_examined_per_scan": 5,
      "rows_produced_per_join": 5,
      "filtered": "100.00",
      "cost_info": {
        "read_cost": "1.00",
        "eval_cost": "1.00",
        "prefix_cost": "2.00",
        "data_read_per_join": "280"
      },
      "used_columns": [
        "id",
        "username",
        "preferences"
      ],
      "attached_condition": "(json_extract(`test`.`users`.`preferences`,'$.notify_on_updates') = TRUE)"
    }
  }
}
```

### 例36: インデックス付きの仮想生成列を追加する

```sql
ALTER TABLE users ADD notify_on_updates TINYINT AS (preferences->"$.notify_on_updates"),
 ADD INDEX(notify_on_updates);

EXPLAIN FORMAT=JSON SELECT * FROM users WHERE preferences->"$.notify_on_updates" = true;
{
  "query_block": {
    "select_id": 1,
    "cost_info": {
      "query_cost": "1.20"
    },
    "table": {
      "table_name": "users",        # < --
      "access_type": "ref",         # < --
      "possible_keys": [
        "notify_on_updates"
      ],
      "key": "notify_on_updates",
      "used_key_parts": [
        "notify_on_updates"
      ],
      "key_length": "2",
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
        "data_read_per_join": "56"
      },
      "used_columns": [
        "id",
        "username",
        "preferences",
        "notify_on_updates"
      ]
    }
  }
}
```

[^1]: 例としては`JSON_EXTRACT`演算子の短縮形(->)の利用が挙げられます。文字列を抽出する際は、この省略演算子とクォート除去演算子(->>)を利用する必要があるでしょう。
