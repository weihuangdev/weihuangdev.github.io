---
layout: post
title:  "clickhouse day 3 (Array)"
date:   2018-07-24 11:44:17 +0800
categories: clickhouse
---

介紹 clickhouse 的一些 Array Type 的使用
[array-functions](https://clickhouse.yandex/docs/en/query_language/functions/array_functions/)  
operators 的一些操作 [clickhouse-operators](https://clickhouse.yandex/docs/en/query_language/operators/)

#### create table 使用 File ENGIN 的 CSV 格式

```console
f16aa4707e9c :) CREATE TABLE t1
:-] (
:-]     v1 Int32,
:-]     a1 Array(Int32),
:-]     s2 Array(String)
:-] ) ENGINE = File(CSV)

CREATE TABLE t1
(
    v1 Int32,
    a1 Array(Int32),
    s2 Array(String)
)
ENGINE = File(CSV)

Ok.

0 rows in set. Elapsed: 0.005 sec.
```
#### 新增幾筆資料，可以用 format CSV 的方式

```console
f16aa4707e9c :) insert into t1 format CSV 1,"[1,2]","['a1','a2']"

INSERT INTO t1 FORMAT CSV

Ok.

1 rows in set. Elapsed: 0.002 sec.

f16aa4707e9c :) insert into t1 format CSV 2,"[2,3]","['a3','a4']"

INSERT INTO t1 FORMAT CSV

Ok.

1 rows in set. Elapsed: 0.002 sec.

f16aa4707e9c :) insert into t1 format CSV 3,"[4,5,6]","['a5','a6','a7']"

INSERT INTO t1 FORMAT CSV

Ok.

1 rows in set. Elapsed: 0.002 sec.

```
#### 資料結構如下

```console
f16aa4707e9c :) select * from t1

SELECT *
FROM t1

┌─v1─┬─a1──────┬─s2───────────────┐
│  1 │ [1,2]   │ ['a1','a2']      │
│  2 │ [2,3]   │ ['a3','a4']      │
│  3 │ [4,5,6] │ ['a5','a6','a7'] │
└────┴─────────┴──────────────────┘

3 rows in set. Elapsed: 0.003 sec.
```
#### 看 clickhouse data 底下產生的 data 格式

```console
clickhouse@f16aa4707e9c:/var/lib/clickhouse/data/default/t1$ cat data.CSV
1,"[1,2]","['a1','a2']"
2,"[2,3]","['a3','a4']"
3,"[4,5,6]","['a5','a6','a7']"
```

#### 測試 Array(Array(String)) 

```console
f16aa4707e9c :) CREATE TABLE t2
:-] (
:-]     v1 Int32,
:-]     a1 Array(Int32),
:-]     s2 Array(String),
:-]     s3 Array(Array(String))
:-] ) ENGINE = File(CSV)

CREATE TABLE t2
(
    v1 Int32,
    a1 Array(Int32),
    s2 Array(String),
    s3 Array(Array(String))
)
ENGINE = File(CSV)

Ok.

0 rows in set. Elapsed: 0.004 sec.

```

#### 新增資料 Array(Array(String)) 資料格式為 "[['a3','a4'],['a5','a6','a7']]"
```console
f16aa4707e9c :) insert into t2 format CSV 1,"[1,2]","['a1','a2']","[['a3','a4'],['a5','a6','a7']]"

INSERT INTO t2 FORMAT CSV

Ok.

1 rows in set. Elapsed: 0.002 sec.


f16aa4707e9c :) insert into t2 format CSV 2,"[3,4]","['a8','a9']","[['a10'],['a11','a12','a13','a14']]"

INSERT INTO t2 FORMAT CSV

Ok.

1 rows in set. Elapsed: 0.002 sec.
```
#### 資料結構如下
```console
f16aa4707e9c :) select * from t2

SELECT *
FROM t2

┌─v1─┬─a1────┬─s2──────────┬─s3──────────────────────────────────┐
│  1 │ [1,2] │ ['a1','a2'] │ [['a3','a4'],['a5','a6','a7']]      │
│  2 │ [3,4] │ ['a8','a9'] │ [['a10'],['a11','a12','a13','a14']] │
└────┴───────┴─────────────┴─────────────────────────────────────┘

2 rows in set. Elapsed: 0.002 sec.
```

#### 可以使用 Array join 再展開
```console
f16aa4707e9c :) select * from t2 Array join s3

SELECT *
FROM t2
ARRAY JOIN s3

┌─v1─┬─a1────┬─s2──────────┬─s3────────────────────────┐
│  1 │ [1,2] │ ['a1','a2'] │ ['a3','a4']               │
│  1 │ [1,2] │ ['a1','a2'] │ ['a5','a6','a7']          │
│  2 │ [3,4] │ ['a8','a9'] │ ['a10']                   │
│  2 │ [3,4] │ ['a8','a9'] │ ['a11','a12','a13','a14'] │
└────┴───────┴─────────────┴───────────────────────────┘

4 rows in set. Elapsed: 0.004 sec.

```
#### 運用一些 clckhouse 提供的 array function 找到資料，例如 has(targeArray,findElement)

```console
f16aa4707e9c :) select has(s3,'a6') from t2 Array join s3

SELECT has(s3, 'a6')
FROM t2
ARRAY JOIN s3

┌─has(s3, 'a6')─┐
│             0 │
│             1 │
│             0 │
│             0 │
└───────────────┘

4 rows in set. Elapsed: 0.003 sec.
```
#### 當成 where 條件

```console
f16aa4707e9c :) select * from t2 Array join s3 where has(s3,'a6') != 0

SELECT *
FROM t2
ARRAY JOIN s3
WHERE has(s3, 'a6') != 0

┌─v1─┬─a1────┬─s2──────────┬─s3───────────────┐
│  1 │ [1,2] │ ['a1','a2'] │ ['a5','a6','a7'] │
└────┴───────┴─────────────┴──────────────────┘

1 rows in set. Elapsed: 0.003 sec.
```
#### 多個條件下

```console
f16aa4707e9c :) select * from t2 Array join s3 where has(s3,'a6') != 0 or has(s3,'a12')

SELECT *
FROM t2
ARRAY JOIN s3
WHERE (has(s3, 'a6') != 0) OR has(s3, 'a12')

┌─v1─┬─a1────┬─s2──────────┬─s3────────────────────────┐
│  1 │ [1,2] │ ['a1','a2'] │ ['a5','a6','a7']          │
│  2 │ [3,4] │ ['a8','a9'] │ ['a11','a12','a13','a14'] │
└────┴───────┴─────────────┴───────────────────────────┘

2 rows in set. Elapsed: 0.003 sec.

```
#### 檔案實際存放結構如下

```console
clickhouse@f16aa4707e9c:/var/lib/clickhouse/data/default/t2$ cat data.CSV
1,"[1,2]","['a1','a2']","[['a3','a4'],['a5','a6','a7']]"
2,"[3,4]","['a8','a9']","[['a10'],['a11','a12','a13','a14']]"
```







