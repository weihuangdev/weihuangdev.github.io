---
layout: post
title:  "clickhouse day 1 (run clickhouse)"
date:   2018-07-23 10:44:17 +0800
categories: clickhouse
---

clickhouse 是 column-oriented DBMS 所以效能會比 row-oriented DBMS 還來得快，因為當要 aggregate 時可以不用像 row-oriented full scan．

> 參考資料
> [clickhouse](https://clickhouse.yandex/docs/en/)

#### 使用 docker 測試
 參考 [clickhouse-docker](https://hub.docker.com/r/yandex/clickhouse-server/)

```console
docker pull yandex/clickhouse-server
```
#### start server instance
```console
docker run -d --name some-clickhouse-server --ulimit nofile=262144:262144 yandex/clickhouse-server
```

#### connect to it from native client

```console
daniel@Danielde-MacBook-Pro > ~/test > docker run -it --rm --link some-clickhouse-server:clickhouse-server yandex/clickhouse-client --host clickhouse-server
ClickHouse client version 1.1.54390.
Connecting to clickhouse-server:9000 as user default.
Connected to ClickHouse server version 1.1.54390.

f16aa4707e9c :) show databases

SHOW DATABASES

┌─name────┐
│ default │
│ system  │
│ testdb  │
└─────────┘

3 rows in set. Elapsed: 0.004 sec.

f16aa4707e9c :)
```

#### create database : 

```console
f16aa4707e9c :) create database testdb
CREATE DATABASE testdb
Ok.
0 rows in set. Elapsed: 0.005 sec.
```

#### create table with MergeTree 須給日期做 partition : 

```console
CREATE TABLE testTables (
    date Date,
    value String
) ENGINE=MergeTree(date, (value), 8192);
```

#### date format 
* toDate -> YYYY-MM-DD
* toDateTime -> YYYY-MM-DD hh:mm:ss

#### insert data
```console
insert into testTables(date,value) values(toDate('2018-07-23') , 'message 1')
```

#### clickhouse table path :  
```console
/var/lib/clickhouse/data/default/testTables
```
#### clickhouse config file :  
```console
/etc/clickhouse-server/config.xml
```

#### system.clusters
```console
f16aa4707e9c :) select * from system.clusters ;

SELECT *
FROM system.clusters

┌─cluster─────────────────────┬─shard_num─┬─shard_weight─┬─replica_num─┬─host_name─┬─host_address─┬─port─┬─is_local─┬─user────┬─default_database─┐
│ test_shard_localhost        │         1 │            1 │           1 │ localhost │ 127.0.0.1    │ 9000 │        1 │ default │                  │
│ test_shard_localhost_secure │         1 │            1 │           1 │ localhost │ 127.0.0.1    │ 9440 │        0 │ default │                  │
└─────────────────────────────┴───────────┴──────────────┴─────────────┴───────────┴──────────────┴──────┴──────────┴─────────┴──────────────────┘

2 rows in set. Elapsed: 0.005 sec.
```

#### create table with ENGINE File 並使用 TabSeparated
```console
CREATE TABLE testFileTable (name String, value String) ENGINE=File(TabSeparated)
```
#### 到 /var/lib/clickhouse/data/default/testFileTable 建立 data.TabSeparated 檔案
```console
clickhouse@f16aa4707e9c:/var/lib/clickhouse/data/default/testFileTable$ ls
data.TabSeparated
```
#### 塞資料到 data.TabSeparated 並用 tab 做分割
```console
echo -e "a\tb\nc\td\ne\tf" >> data.TabSeparated
```
#### select data
```console
f16aa4707e9c :) select * from testFileTable

SELECT *
FROM testFileTable

┌─name─┬─value─┐
│ a    │ b     │
│ c    │ d     │
│ e    │ f     │
└──────┴───────┘
```
#### 可以根據不同的 [Input and output formats](https://clickhouse.yandex/docs/en/interfaces/formats/) 做查詢
```console
f16aa4707e9c :) select * from testFileTable FORMAT JSON

SELECT *
FROM testFileTable
FORMAT JSON

{
	"meta":
	[
		{
			"name": "name",
			"type": "String"
		},
		{
			"name": "value",
			"type": "String"
		}
	],

	"data":
	[
		{
			"name": "a",
			"value": "b"
		},
		{
			"name": "c",
			"value": "d"
		},
		{
			"name": "e",
			"value": "f"
		}
	],

	"rows": 3,

	"statistics":
	{
		"elapsed": 0.0004059,
		"rows_read": 3,
		"bytes_read": 60
	}
}

3 rows in set. Elapsed: 0.002 sec.
```


