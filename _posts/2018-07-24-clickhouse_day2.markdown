---
layout: post
title:  "clickhouse day 2 (Nested data structures)"
date:   2018-07-24 10:44:17 +0800
categories: clickhouse
---

clickhouse 的 Data types 支援很多格式，這邊介紹 [Nested](https://clickhouse.yandex/docs/en/single/#nested-data-structures)．  
目前 Nested data 只支援 a single nesting level．


#### create table 並建立 Nested column(Attrs 包含了 key 和 value) 

```console
f16aa4707e9c :) CREATE TABLE testdb.nested (EventDate Date, UserID UInt64, Attrs Nested(Key String, Value String)) ENGINE = MergeTree(EventDate, UserID, 8192)

CREATE TABLE testdb.nested
(
    EventDate Date,
    UserID UInt64,
    Attrs Nested(
    Key String,
    Value String)
)
ENGINE = MergeTree(EventDate, UserID, 8192)

Ok.

0 rows in set. Elapsed: 0.005 sec.
```
#### 新增資料 Attrs 的 key : ['price', 'color','name'], value : ['high', 'red','product1']
```console
f16aa4707e9c :) INSERT INTO testdb.nested VALUES ('2016-01-01', 123, ['price', 'color','name'], ['high', 'red','product1'])

INSERT INTO testdb.nested VALUES

Ok.

1 rows in set. Elapsed: 0.003 sec.
```
#### 資料結構如下
```console
f16aa4707e9c :) select * from testdb.nested

SELECT *
FROM testdb.nested

┌──EventDate─┬─UserID─┬─Attrs.Key────────────────┬─Attrs.Value───────────────┐
│ 2016-01-01 │    123 │ ['price','color','name'] │ ['high','red','product1'] │
└────────────┴────────┴──────────────────────────┴───────────────────────────┘

1 rows in set. Elapsed: 0.003 sec.
```
#### 使用 ARRAY JOIN 將資料分成 3 筆

```console
f16aa4707e9c :) select * from testdb.nested array join Attrs

SELECT *
FROM testdb.nested
ARRAY JOIN Attrs

┌──EventDate─┬─UserID─┬─Attrs.Key─┬─Attrs.Value─┐
│ 2016-01-01 │    123 │ price     │ high        │
│ 2016-01-01 │    123 │ color     │ red         │
│ 2016-01-01 │    123 │ name      │ product1    │
└────────────┴────────┴───────────┴─────────────┘

3 rows in set. Elapsed: 0.003 sec.
```
#### 針對 Attrs.Value 下條件查詢

```console
f16aa4707e9c :) select * from testdb.nested array join Attrs where Attrs.Value='red'

SELECT *
FROM testdb.nested
ARRAY JOIN Attrs
WHERE Attrs.Value = 'red'

┌──EventDate─┬─UserID─┬─Attrs.Key─┬─Attrs.Value─┐
│ 2016-01-01 │    123 │ color     │ red         │
└────────────┴────────┴───────────┴─────────────┘

1 rows in set. Elapsed: 0.004 sec.
```
#### 再新增一筆資料
```console
f16aa4707e9c :) INSERT INTO testdb.nested VALUES ('2016-01-02', 456, ['price', 'color'], ['low', 'black'])

INSERT INTO testdb.nested VALUES

Ok.

1 rows in set. Elapsed: 0.003 sec.
```
#### 資料結構如下

```console
f16aa4707e9c :) select * from testdb.nested

SELECT *
FROM testdb.nested

┌──EventDate─┬─UserID─┬─Attrs.Key─────────┬─Attrs.Value─────┐
│ 2016-01-02 │    456 │ ['price','color'] │ ['low','black'] │
└────────────┴────────┴───────────────────┴─────────────────┘
┌──EventDate─┬─UserID─┬─Attrs.Key────────────────┬─Attrs.Value───────────────┐
│ 2016-01-01 │    123 │ ['price','color','name'] │ ['high','red','product1'] │
└────────────┴────────┴──────────────────────────┴───────────────────────────┘

2 rows in set. Elapsed: 0.004 sec.
```
#### array join

```console
f16aa4707e9c :) select * from testdb.nested array join Attrs

SELECT *
FROM testdb.nested
ARRAY JOIN Attrs

┌──EventDate─┬─UserID─┬─Attrs.Key─┬─Attrs.Value─┐
│ 2016-01-02 │    456 │ price     │ low         │
│ 2016-01-02 │    456 │ color     │ black       │
└────────────┴────────┴───────────┴─────────────┘
┌──EventDate─┬─UserID─┬─Attrs.Key─┬─Attrs.Value─┐
│ 2016-01-01 │    123 │ price     │ high        │
│ 2016-01-01 │    123 │ color     │ red         │
│ 2016-01-01 │    123 │ name      │ product1    │
└────────────┴────────┴───────────┴─────────────┘

5 rows in set. Elapsed: 0.003 sec.
```
#### 查詢 Attrs.Value 

```console
f16aa4707e9c :) select * from testdb.nested array join Attrs where Attrs.Value='product1'

SELECT *
FROM testdb.nested
ARRAY JOIN Attrs
WHERE Attrs.Value = 'product1'

┌──EventDate─┬─UserID─┬─Attrs.Key─┬─Attrs.Value─┐
│ 2016-01-01 │    123 │ name      │ product1    │
└────────────┴────────┴───────────┴─────────────┘

1 rows in set. Elapsed: 0.004 sec.
```



