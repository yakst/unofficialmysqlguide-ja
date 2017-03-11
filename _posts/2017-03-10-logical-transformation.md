---
title: 論理変換
date: 2017-03-10 00:00:01 -0900
---

# 論理変換

MySQLのオプティマイザは実行結果に影響を与えないように、クエリーを変換することがあります。余分な操作を取り除き、クエリーをより速く実行できるようクエリーを書き換えることがこの「変換」のゴールです。例えば、次のクエリーについて考えてみましょう。

```sql
SELECT * FROM Country
WHERE population > 5000000 AND continent='Asia' AND 1=1;
```

「1は常に1」ですので、この条件と`AND`(`OR`ではありません)は完全に冗長です。
クエリーを実行中に各行に対して「1が1のままである」ことを確認することには何にも意味がありませんし、条件を削除しても同じ結果がえられます。`OPTIMIZER_TRACE`を利用すると、MySQLがこの「変換」およびその他の数々の変換を実施していることを確認できます。

```
..
   {
      "join_optimization": {
        "select#": 1,
        "steps": [
          {
            "condition_processing": {
              "condition": "WHERE",
              "original_condition": "((`Country`.`Population` > 5000000) and (1 = 1))",
              "steps": [
                {
                  "transformation": "equality_propagation",
                  "resulting_condition": "((`Country`.`Population` > 5000000) and (1 = 1))"
                },
                {
                  "transformation": "constant_propagation",
                  "resulting_condition": "((`Country`.`Population` > 5000000) and (1 = 1))"
                },
                {
                  "transformation": "trivial_condition_removal",
                  "resulting_condition": "(`Country`.`Population` > 5000000)"
                }
              ]
            }
          },
..
```

すべての変換が終わったあとの書き換えられたクエリーは、`EXPLAIN`が実行されたあとの`SHOW WARNINGS`でも確認できます。上記のステートメントは次のように書き換えられます。

```sql
EXPLAIN FORMAT=JSON SELECT * FROM Country WHERE population > 5000000 AND 1=1;
SHOW WARNINGS;
/* select#1 */ select `world`.`Country`.`Code` AS `Code`,`world`.`Country`.`Name` AS `Name`,`world`.`Country`.`Continent` AS `Continent`,`world`.`Country`.`Region` AS `Region`,`world`.`Country`.`SurfaceArea` AS `SurfaceArea`,`world`.`Country`.`IndepYear` AS `IndepYear`,`world`.`Country`.`Population` AS `Population`,`world`.`Country`.`LifeExpectancy` AS `LifeExpectancy`,`world`.`Country`.`GNP` AS `GNP`,`world`.`Country`.`GNPOld` AS `GNPOld`,`world`.`Country`.`LocalName` AS `LocalName`,`world`.`Country`.`GovernmentForm` AS `GovernmentForm`,`world`.`Country`.`HeadOfState` AS `HeadOfState`,`world`.`Country`.`Capital` AS `Capital`,`world`.`Country`.`Code2` AS `Code2` from `world`.`Country` where (`world`.`Country`.`Population` > 5000000)
```

## 変換の例

下記に論理変換されたクエリーの例をいくつか示します。いくつかの例で`PRIMARY`および`UNIQUE`インデックスがクエリーの実行フェーズの前に永続的に変換されていることに注意してください。クエリーのこの部分は既に最も効率的な形式となっているため、オプティマイザーは実行計画を考慮する前に永続的な変換を行っています。
ただし、考慮する実行計画の数を減らすための「他の最適化」は引き続き適用されます。

### Codeが主キー。クエリーは定数値に変換された値に書き換えられる

#### 元のクエリー

```sql
SELECT * FROM Country WHERE code='CAN'
```

#### 書き換え後のクエリー

```sql
/* select#1 */ select 'CAN' AS `Code`,'Canada' AS `Name`,'North America'
AS `Continent`, 'North America' AS `Region`,'9970610.00' AS
`SurfaceArea`,'1867' AS `IndepYear`, '31147000' AS `Population`,'79.4' AS
`LifeExpectancy`,'598862.00' AS `GNP`,'625626.00' AS `GNPOld`, 'Canada' AS
`LocalName`,'Constitutional Monarchy, Federation' AS `GovernmentForm`,
'Elisabeth II' AS `HeadOfState`,'1822' AS `Capital`,'CA' AS `Code2` from
`world`.`Country` where 1
```

### Codeは主キーだが、主キーがテーブルにない場合(不可能なwhere)

#### 元のクエリー

```sql
SELECT * FROM Country WHERE code='XYZ'
```

#### 書き換え後のクエリー

```sql
/* select#1 */ select NULL AS `Code`,NULL AS `Name`,NULL AS
`Continent`,NULL AS `Region`, NULL AS `SurfaceArea`,NULL AS
`IndepYear`,NULL AS `Population`,NULL AS `LifeExpectancy`,NULL AS `GNP`,
NULL AS `GNPOld`,NULL AS `LocalName`,NULL AS `GovernmentForm`,NULL AS
`HeadOfState`,NULL AS `Capital`, NULL AS `Code2` from `world`.`Country`
where multiple equal('XYZ', NULL)
```

### 不可能なwhereの他の例。Codeは存在するが1=0が不可能な条件


#### 元のクエリー

```sql
SELECT * FROM Country WHERE code='CAN' AND 1=0
```

#### 書き換え後のクエリー

```sql
/* select#1 */ select `world`.`Country`.`Code` AS `Code`,
`world`.`Country`.`Name` AS `Name`, `world`.`Country`.`Continent`
AS `Continent`,`world`.`Country`.`Region` AS `Region`,
`world`.`Country`.`SurfaceArea` AS　
`SurfaceArea`,`world`.`Country`.`IndepYear` AS `IndepYear`,
`world`.`Country`.`Population` AS
`Population`,`world`.`Country`.`LifeExpectancy` AS `LifeExpectancy`,
`world`.`Country`.`GNP` AS `GNP`,`world`.`Country`.`GNPOld` AS `GNPOld`,
`world`.`Country`.`LocalName` AS
`LocalName`,`world`.`Country`.`GovernmentForm` AS `GovernmentForm`,
`world`.`Country`.`HeadOfState` AS
`HeadOfState`,`world`.`Country`.`Capital` AS `Capital`,
`world`.`Country`.`Code2` AS `Code2` from `world`.`Country` where 0
```

### 派生テーブルのサブクエリーがCountryテーブルのjoinに直接マージされる

#### 元のクエリー

```sql
SELECT City.* FROM City, (SELECT * FROM Country WHERE continent='Asia') as
Country WHERE Country.code=City.CountryCode AND Country.population >
5000000;
```

####  書き換え後のクエリー[^1]

```sql
/* select#1 */ select `world`.`City`.`ID` AS `ID`,`world`.`City`.`Name` AS
`Name`, `world`.`City`.`CountryCode` AS
`CountryCode`,`world`.`City`.`District` AS `District`,
`world`.`City`.`Population` AS `Population` from `world`.`City` join
`world`.`Country` where ((`world`.`Country`.`Continent` = 'Asia') and
(`world`.`City`.`CountryCode` = ` world`.`Country`.`Code`) and
(`world`.`Country`.`Population` > 5000000))
```

### ビューの定義がクエリーで利用されるとともにマージされる

#### 元のクエリー

```sql
CREATE VIEW Countries_in_asia AS SELECT * FROM Country WHERE
continent='Asia'; SELECT * FROM Countries_in_asia WHERE population >
5000000;
```

#### 書き換え後のクエリー

```sql
/* select#1 */ select `world`.`Country`.`Code` AS
`Code`,`wo`rld`.`Country`.`Name` AS `Name`, `world`.`Country`.`Continent`
AS `Continent`,`world`.`Country`.`Region` AS `Region`,
`world`.`Country`.`SurfaceArea` AS `SurfaceArea`,`world`.`Country`.`IndepYear`
AS `IndepYear`, `world`.`Country`.`Population` AS
`Population`,`world`.`Country`.`LifeExpectancy` AS `LifeExpectancy`, `world`.`Country`.`GNP` AS `GNP`,`world`.`Country`.`GNPOld` AS `GNPOld`,
`world`.`Country`.`LocalName` AS
`LocalName`,`world`.`Country`.`GovernmentForm` AS `GovernmentForm`,
`world`.`Country`.`HeadOfState` AS `HeadOfState`,`world`.`Country`.`Capital` AS `Capital`, `world`.`Country`.`Code2` AS `Code2` from `world`.`Country` where ((`world`.`Country`.`Continent` = 'Asia') and (`world`.`Country`.`Population` > 5000000))
```

[^1]: この挙動は`optimizer_switch`の`derived_merge`(デフォルトではon)の設定に依存します
