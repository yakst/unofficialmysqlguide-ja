---
title: Visual Explain 
date: 2017-04-09 00:00:01 -0900
article_index: 12
original_url: http://www.unofficialmysqlguide.com/visual-explain.html
translator: taka-h (@takaidohigasi)
---

MySQL WorkbenchにはVisual Explainというツールが同梱されており、これは複雑な実行計画を人間にとって読みやすくするものです。内部的にはこの機能は`EXPLAIN FORMAT=JSON`を利用しているため、通常の`EXPLAIN FORMAT=JSON`以上の機能を「もたない」ことを述べておきます。実際のところ、シンプル化するためにカバリングインデックスの利用などのいくつかの出力を省略しています。

Visual Explainの色はアクセスメソッドを示しています。

* `ref`は緑
* `range`はオレンジ
* `ALL` (テーブルスキャン) と`INDEX` (インデックススキャン)は赤

これまで例の中で見てきたように、選択性の高いrangeアクセスは選択性の低いrefアクセスより望ましく、これはおそらくちょっとした単純化でしょう。

![explain-c.png](http://www.unofficialmysqlguide.com/_images/explain-c.png)

![explain-cp.png](http://www.unofficialmysqlguide.com/_images/explain-cp.png)

![explain-cpn.png](http://www.unofficialmysqlguide.com/_images/explain-cpn.png)

![explain-force-p.png](http://www.unofficialmysqlguide.com/_images/explain-force-p.png)

![explain-fts.png](http://www.unofficialmysqlguide.com/_images/explain-fts.png)
