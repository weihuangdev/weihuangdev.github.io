---
layout: post
title:  "cassandra day 1 (run cassandra)"
date:   2018-09-20 10:44:17 +0800
categories: cassandra
---

#### 下載 cassandra images
```
docker pull cassandra
```

#### run cassandra
```
docker run --name test-cassandra-1 -d cassandra
```

#### run cassandra cluster
有 2 種 : 
1. 在同一台機器建不同 instances
```
docker run --name test-cassandra-2 -d -e CASSANDRA_SEEDS="$(docker inspect --format='{{ .NetworkSettings.IPAddress }}' test-cassandra-1)" cassandra
```
啟動後  
```
> docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                         NAMES
87e56257fb9e        cassandra           "docker-entrypoint.s…"   3 seconds ago       Up 2 seconds        7000-7001/tcp, 7199/tcp, 9042/tcp, 9160/tcp   test-cassandra-2
abcd5d24e322        cassandra           "docker-entrypoint.s…"   5 minutes ago       Up 5 minutes        7000-7001/tcp, 7199/tcp, 9042/tcp, 9160/tcp   test-cassandra-1
```
也可以透過 --link
```
docker run --name test-cassandra-2 -d --link test-cassandra-1 cassandra
```
2. 在不同機器建不同的 instances，這邊先不實作提供指令參考
```
docker run --name some-cassandra -d -e CASSANDRA_BROADCAST_ADDRESS=10.42.42.42 -p 7000:7000 cassandra:tag
docker run --name some-cassandra -d -e CASSANDRA_BROADCAST_ADDRESS=10.43.43.43 -p 7000:7000 -e CASSANDRA_SEEDS=10.42.42.42 cassandra:tag
```


```
root@d0447e24625d:/# nodetool status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load       Tokens       Owns (effective)  Host ID                               Rack
UN  172.17.0.3  75.19 KiB  256          52.1%             c4cd5109-83ff-4adc-81d1-7d05b7dd2bfb  rack1
UN  172.17.0.2  80.16 KiB  256          47.9%             0dc02012-3b01-4bd2-bcf4-65855f630e9a  rack1
```


#### 連線 cassandra
```
> docker run -it --link test-cassandra-1:cassandra --rm cassandra sh -c 'exec cqlsh "$CASSANDRA_PORT_9042_TCP_ADDR"'
Connected to Test Cluster at 172.17.0.2:9042.
[cqlsh 5.0.1 | Cassandra 3.11.3 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
cqlsh>
```
#### 透過 exec 連線 cassandra，再下 cqlsh

```
docker exec -it eb31ab378632 bash

root@eb31ab378632:/# cqlsh
Connected to Test Cluster at 127.0.0.1:9042.
[cqlsh 5.0.1 | Cassandra 3.11.3 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
cqlsh>
```
#### 查看目前有的 keyspaces
```
cqlsh> describe keyspaces;
system_schema  system_auth  system  system_distributed  system_traces
```
#### 建立 keyspace
```
cqlsh> CREATE KEYSPACE castest WITH REPLICATION = {'class': 'SimpleStrategy', 'replication_factor': 1 };
```
#### drop keyspace
```
cqlsh> drop keyspace castest ;
```
#### create table
```
cqlsh> CREATE TABLE castest.prod (prodid text PRIMARY KEY, count int);
```
#### 看所有 tables
```
cqlsh> describe tables ;

Keyspace castest
----------------
prod
```
#### insert data
```
cqlsh> INSERT INTO castest.prod (prodid, count) VALUES ('1', 20);
```
#### select data
```
cqlsh> select * from castest.prod ;

 prodid | count
--------+-------
      4 |    30
      3 |    40
      2 |    10
      1 |    20

(4 rows)
```
#### update data
```
cqlsh> update castest.prod set count=55 where prodid='3'
```
#### delete data
```
cqlsh> delete from castest.prod where prodid='3' ;
```


> 參考資料  
> [docker-cassandra](https://docs.docker.com/samples/library/cassandra/)  
> [Deploy Spark with Cassandra cluster](https://opencredo.com/deploy-spark-apache-cassandra/)
> [configur cassandra cluster with docker](http://abiasforaction.net/apache-cassandra-cluster-docker/)




