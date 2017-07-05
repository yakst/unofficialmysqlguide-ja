---
title: クエリーリライト
date: 2017-07-04 00:00:01 -0900
article_index: 20
original_url: http://www.unofficialmysqlguide.com/query-rewrite.html
translator: taka-h (@takaidohigasi)
---

MySQLはサーバーサイドでステートメントを書き換える機能を提供しています。これはあるパターンに合致するステートメントを新しいパターンに書き換えられるサーバーサイドの正規表現のようなものであると考えることができます。

この機能の設計のゴールはデータベース管理者がクエリーヒントをステートメントに挿入できるようにすることでした。これはORMを利用していたり、アプリケーションが[プロプライエタリ](https://ja.wikipedia.org/wiki/%E3%83%97%E3%83%AD%E3%83%97%E3%83%A9%E3%82%A4%E3%82%A8%E3%82%BF%E3%83%AA%E3%83%BB%E3%82%BD%E3%83%95%E3%83%88%E3%82%A6%E3%82%A7%E3%82%A2)であったりして、アプリケーション自体を修正できない場合に状況を改善する手段を提供するものです。

クエリーリライトが正規表現とに似たものであると述べましたが、内部的にははるかに効率的なものであることはお伝えしておく必要があります。ステートメントがパースされる際に、ダイジェスト(プリペアードステートメントという形でも知られています)が内部のハッシュテーブルの書き換えるべき値と比較されます。ステートメントが「書き換えが必要である」と判断されれば、サーバーはこのクエリー書き換えのステップを実行し再度クエリーをパースします。書き換えの必要のないクエリーにもわずかなオーバーヘッドが発生し、書き換えの必要なクエリーは簡単にいうと2回パースする必要があります。

例33: クエリーリライトでサーバーサイドのクエリーを変更する

```sql
# コマンドラインからクエリーリライトをインストールします
mysql -u root -p < install_rewriter.sql

# MySQLサーバーでリライトルールを追加し、フラッシュします
INSERT INTO query_rewrite.rewrite_rules(pattern_database, pattern, replacement) VALUES (
"world",
"SELECT * FROM Country WHERE population > ? AND continent=?",
"SELECT * FROM Country WHERE population > ? AND continent=? LIMIT 1"
);
CALL query_rewrite.flush_rewrite_rules();

# クエリーリライトはリライトが発生するとき毎回warningを発生させます
SELECT * FROM Country WHERE population > 5000000 AND continent='Asia';
SHOW WARNINGS;
Query 'SELECT * FROM Country WHERE population > 5000000 AND continent='Asia'' rewritten to 'SELECT * FROM Country WHERE population > 5000000 AND continent='Asia' LIMIT 1' by a query rewrite plugin

SELECT * FROM query_rewrite.rewrite_rules\G
*************************** 1. row ***************************
                id: 2
           pattern: SELECT * FROM Country WHERE population > ? AND continent=?
  pattern_database: world
       replacement: SELECT * FROM Country WHERE population > ? AND continent=? LIMIT 1
           enabled: YES
           message: NULL
    pattern_digest: 88876bbb502cef6efddcc661cce77deb
normalized_pattern: select `*` from `world`.`Country` where ((`population` > ?) and (`continent` = ?))
```

{% include info.html info="MySQLサーバーは各種Query Rewriteプラグインをサポートしています。ここで示される例では、MySQLサーバーディストリビューション同梱のポストパース クエリーリライトプラグイン(通称Rewriter)を利用しています。クエリーによっていはプレパースプラグインを利用する必要があるかもしれません。詳細については次のMySQLのマニュアルを御覧ください。 https://dev.mysql.com/doc/refman/5.7/en/rewriter-query-rewrite-plugin.html" %}
