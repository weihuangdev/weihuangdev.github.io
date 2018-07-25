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
#### --ulimit nofile=262144:262144 的意思是限制 open files 的數量為 262144

```console
clickhouse@d2d3c8ffba95:/var/lib/clickhouse/data/default/testTables$ ulimit -a
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 7862
max locked memory       (kbytes, -l) 82000
max memory size         (kbytes, -m) unlimited
open files                      (-n) 262144
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
max user processes              (-u) unlimited
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited
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


#### 如果要自己安裝 clickhouse 只要執行下列步驟
* check SSE 4.2 supported

```console
grep -q sse4_2 /proc/cpuinfo && echo "SSE 4.2 supported" || echo "SSE 4.2 not supported"
```
* 照步驟執行

```console
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv E0C56BD4    # optional

sudo apt-add-repository "deb http://repo.yandex.ru/clickhouse/deb/stable/ main/"
sudo apt-get update

sudo apt-get install -y clickhouse-server clickhouse-client

sudo service clickhouse-server start
clickhouse-client

```

> 參考資料  
> [clickhouse-docs](http://clickhouse-docs.readthedocs.io/en/latest/getting_started.html)




