---
layout: post
title:  "cassandra day 3 (spark cassandra repartitionByCassandraReplica)"
date:   2019-04-10 11:44:17 +0800
categories: cassandra
---


spark cassandra 影響項目 : 
1. spark partitioner : 
spark partition : 
spark 先去讀取 HDFS 的檔案，這時候 spark 會用 Some(org.apache.spark.HashPartitioner@1b) 來切 partition．

cassandra partition : 
cassandra 會根據 table 的 partition key (row key) 來切 partition．


```
val repartPersons = newPersons.repartitionByCassandraReplica(EnrichConfig.CASSANDRA_KEYSPACE,EnrichConfig.CASSANDRA_TABLE)
```
當 RDD 透過 repartitionByCassandraReplica，會變成使用 Some(com.datastax.spark.connector.rdd.partitioner.ReplicaPartitioner@498c988) 來切 partition．


```
val personsRdd1 = sc.cassandraTable[Person](EnrichConfig.CASSANDRA_KEYSPACE, EnrichConfig.CASSANDRA_TABLE).keyBy[Tuple1[String]]("id")
```
透過 spark cassandra connector 去查詢時，會使用 Some(com.datastax.spark.connector.rdd.partitioner.CassandraPartitioner@97b45228) 來切 partition．

當 partitioner 為 none 時，後續的 RDD 操作可能會有大量的 shuffle．

2. cassandra tabel compaction : 
因為 cassandra table 會把資料存放成 sstable，sstable 是許多的檔案，所以 table 達到一個條件後會執行 compaction 把檔案合併．
那如果剛好在做 compaction 時，對該 table 進行大量的操作不確定是否會造成問題．


3. RDD shuffle : 
根據 Data Locality 理論上 spark + cassandra 的運算是希望各自的 partition 在各自的節點上，盡量減少 shuffle．
但目前看起來程式 update 都還是會做很多 shuffle．不知道是否是因為個階段的 partitioner 不一致的原因．
如果使用了 repartitionByCassandraReplica 就一定會 shuffle 因為底層程式會跑這一段 : 
```
val partitioner = new ReplicaPartitioner[T](
  tableName,
  keyspaceName,
  partitionsPerHost,
  partitionKeyMapper,
  connector)

val repart = rdd
  .map((_,None))
  .partitionBy(partitioner)
  .mapPartitions(_.map(_._1), preservesPartitioning = true)

new CassandraPartitionedRDD[T](repart, keyspaceName, tableName)

```



#### repartitionByCassandraReplica
```
package ght.mi.imx.update

import com.datastax.spark.connector._
import ght.mi.core.mapping.{JsonInterface, UrlInterface}
import ght.mi.core.model.criteria.ProfileCritUnit
import ght.mi.core.model.map.Site
import ght.mi.core.model.person.Person
import ght.mi.core.model.setting.UpdateSetting
import ght.mi.core.util.EnrichConfig
import ght.mi.imx.parser.IMXPersonParser
import org.apache.spark.broadcast.Broadcast
import org.apache.spark.sql.SparkSession

object UpdateIMX {
    def run(spark: SparkSession, setting: UpdateSetting, inDir: String): Unit = {

        // SPARK
        val sc = spark.sparkContext

        // SINK - broadcast reference objects
        var urlSink:Broadcast[Map[String, Site]] = null
        if(setting.urlMapDir.nonEmpty) {
            val urlMapping = sc.textFile(setting.urlMapDir).collect
            urlSink = sc.broadcast(UrlInterface.readUrl(urlMapping))
        }

        var profileSink:Broadcast[Set[ProfileCritUnit]] = null
        if(setting.labelCritDir.nonEmpty) {
            val jsonFile = sc.textFile(setting.labelCritDir).collect.mkString.replace(" ", "")
            val jsonobj = JsonInterface.readJSON(jsonFile)
            profileSink = sc.broadcast(JsonInterface.json2ProfileCritUnits(jsonobj))
        }

        val newPersons = sc.wholeTextFiles(inDir).mapPartitions{ iter =>
            for (line <- iter) yield {
                val fileName = line._1
                val startTime = fileName.split("/")(0)
                val str = line._2 + "," + startTime
                val person = IMXPersonParser(setting.dataType, str, urlSink, profileSink)
                (person.id, person)
            }
        }.filter(
            p => p._1 != ""
        ).reduceByKey(
            Person.merge
        )

        // LeftJoin with Cassandra
        val repartPersons = newPersons.repartitionByCassandraReplica(
            EnrichConfig.CASSANDRA_KEYSPACE,
            EnrichConfig.CASSANDRA_TABLE)

        val joinPersons = repartPersons.leftJoinWithCassandraTable[Person](EnrichConfig.CASSANDRA_KEYSPACE,
            EnrichConfig.CASSANDRA_TABLE)

        // Merge newPerson (from data) with oldPerson (from Cassandra)
        val mergedPersons = joinPersons.mapPartitions { iter =>
            for(p <- iter) yield { p match {
                case (newPerson, oldPerson) => {
                    val mergedPerson = Person.merge(oldPerson.getOrElse(new Person(id = newPerson._2.id)), newPerson._2)
                    mergedPerson
                }
            }
            }
        }

        mergedPersons.saveToCassandra(EnrichConfig.CASSANDRA_KEYSPACE, EnrichConfig.CASSANDRA_TABLE)

        println(s"${setting.dataType} update count: ${mergedPersons.count}")

        spark.close
    }
}
```


#### leftOuterJoin
```
package ght.mi.twm.update

import com.datastax.spark.connector._
import ght.mi.core.mapping.{JsonInterface, UrlInterface}
import ght.mi.core.model.criteria.ProfileCritUnit
import ght.mi.core.model.map.Site
import ght.mi.core.model.person.Person
import ght.mi.core.model.setting.UpdateSetting
import ght.mi.core.util.{DataType, EnrichConfig}
import ght.mi.twm.parser.TWMPersonParser
import org.apache.spark.broadcast.Broadcast
import org.apache.spark.sql.SparkSession

object UpdatebyCassandra {

  def run(spark: SparkSession, setting: UpdateSetting): Unit = {

    // SPARK
    val sc = spark.sparkContext

    // SINK - broadcast reference objects
    var urlSink: Broadcast[Map[String, Site]] = null
    if (setting.urlMapDir.nonEmpty) {
      val urlMapping = sc.textFile(setting.urlMapDir).collect
      urlSink = sc.broadcast(UrlInterface.readUrl(urlMapping))
    }

    var profileSink: Broadcast[Set[ProfileCritUnit]] = null
    if (setting.labelCritDir.nonEmpty) {
      val jsonFile = sc.textFile(setting.labelCritDir).collect.mkString.replace(" ", "")
      val jsonobj = JsonInterface.readJSON(jsonFile)
      profileSink = sc.broadcast(JsonInterface.json2ProfileCritUnits(jsonobj))
    }

    // cache 拿掉測測看
    val newPersons = sc.textFile(setting.inDir).flatMap(
      str => {
        stringToPerson(str, setting, urlSink ,profileSink)
      }
    ).reduceByKey(
      Person.merge
    ).cache()

    // Some(org.apache.spark.HashPartitioner@1b)
    println("############ CHECK_PARTITIONER newPersons.partitioner --------> " + newPersons.partitioner)

    val personsRdd1 = sc.cassandraTable[Person](EnrichConfig.CASSANDRA_KEYSPACE, EnrichConfig.CASSANDRA_TABLE).keyBy[Tuple1[String]]("id")
    // Some(com.datastax.spark.connector.rdd.partitioner.CassandraPartitioner@e23cad61)
    println("############ CHECK_PARTITIONER personsRdd1.partitioner --------> " + personsRdd1.partitioner)
    println("############ CHECK_PARTITIONER cache ..........")
    val tmpNewJoinPersons = newPersons.map(p => (new Tuple1(p._1),p._2)).leftOuterJoin(personsRdd1).cache()

    // Some(com.datastax.spark.connector.rdd.partitioner.CassandraPartitioner@e23cad61)
    println("############ CHECK_PARTITIONER tmpNewJoinPersons.partitioner --------> " + tmpNewJoinPersons.partitioner)

    val newJoinPersons = tmpNewJoinPersons.mapPartitions {
      iter =>
        for (p <- iter) yield {
          p match {
            case (id , (newPerson,oldPerson)) => {
              val mergedPerson = Person.merge(oldPerson.getOrElse(new Person(id = newPerson.id)), newPerson)
              mergedPerson
            }
          }
        }
    }
    println("############ CHECK_PARTITIONER newJoinPersons.partitioner --------> " + newJoinPersons.partitioner)

    // LeftJoin with Cassandra
    val repartPersons = newJoinPersons.repartitionByCassandraReplica(
      EnrichConfig.CASSANDRA_KEYSPACE,
      EnrichConfig.CASSANDRA_TABLE)
    // Some(com.datastax.spark.connector.rdd.partitioner.ReplicaPartitioner@2332c46)
    println("############ CHECK_PARTITIONER repartPersons.partitioner --------> " + repartPersons.partitioner)
    repartPersons.saveToCassandra(EnrichConfig.CASSANDRA_KEYSPACE, EnrichConfig.CASSANDRA_TABLE)

    spark.close
  }

  def stringToPerson(str:String ,config: UpdateSetting, urlSink:Broadcast[Map[String, Site]] , profileSink:Broadcast[Set[ProfileCritUnit]]):Option[(String,Person)] = {
    val person = config.dataType match {
      case DataType.LOCATION => TWMPersonParser.locString(str.split(","))
      case DataType.DPI => TWMPersonParser.dpiString(str, urlSink.value)
      case DataType.CDR => TWMPersonParser.cdrString(str.split(","))
      case DataType.PROFILE => TWMPersonParser.profileString(str, profileSink.value)
      case DataType.KVS => TWMPersonParser.kvString(str.split(","))
      case _ => TWMPersonParser.parseString(str)
    }
    val mainId = Person.getMainId(person)
    if ("".equals(mainId)) {
      None
    } else {
      Option(mainId -> person)
    }
  }
}

```

#### 其他筆記

Basic Maintenance : 
```
nodetool flush
nodetool cleanup
nodetool repair
```
針對某個 table 做 compaction : 

```
nodetool compact miks1 person
```

檢查是否有 compact pending : 
```
nodetool compactionstats
```
table 資訊 : 

```
CREATE TABLE miks1.person (
    id text PRIMARY KEY,
    cdrs map<text, frozen<cdr>>,
    certset set<int>,
    client_labels map<int, double>,
    dpis map<text, frozen<dpi>>,
    ids map<text, text>,
    kvs map<int, int>,
    labels map<int, double>,
    last bigint,
    locations list<frozen<location>>,
    profiles map<int, text>
) WITH bloom_filter_fp_chance = 0.01
    AND caching = {'keys': 'ALL', 'rows_per_partition': 'NONE'}
    AND comment = ''
    AND compaction = {'class': 'org.apache.cassandra.db.compaction.SizeTieredCompactionStrategy', 'max_threshold': '32', 'min_threshold': '4'}
    AND compression = {'chunk_length_in_kb': '64', 'class': 'org.apache.cassandra.io.compress.LZ4Compressor'}
    AND crc_check_chance = 1.0
    AND dclocal_read_repair_chance = 0.1
    AND default_time_to_live = 0
    AND gc_grace_seconds = 864000
    AND max_index_interval = 2048
    AND memtable_flush_period_in_ms = 0
    AND min_index_interval = 128
    AND read_repair_chance = 0.0
    AND speculative_retry = '99PERCENTILE';
CREATE INDEX ids_types ON miks1.person (keys(ids));
CREATE INDEX person_locations_idx ON miks1.person (values(locations));
```

* index 會在 table 目錄底下的隱藏 folder : 
```
/data/apache-cassandra-3.11.4/data/data/miks1/person-4dc5ae104ec711e996d241985e0b2fa2>ls -al
total 33176196
drwxrwxrwx.  6 miuser miuser        4096 Apr  3 10:57 .
drwxrwxrwx. 10 miuser miuser        4096 Apr  2 19:10 ..
drwxrwxrwx.  2 miuser miuser        4096 Apr  3 01:35 .ids_types
drwxrwxrwx.  2 miuser miuser      839680 Apr  3 11:07 .person_locations_idx
drwxrwxrwx.  4 miuser miuser          63 Mar 29 10:40 backups
-rw-rw-r--.  1 miuser miuser     5205403 Apr  3 10:57 md-4404-big-CompressionInfo.db
-rw-rw-r--.  1 miuser miuser 15660696469 Apr  3 10:57 md-4404-big-Data.db
-rw-rw-r--.  1 miuser miuser          10 Apr  3 10:57 md-4404-big-Digest.crc32
-rw-rw-r--.  1 miuser miuser     3816528 Apr  3 10:57 md-4404-big-Filter.db
-rw-rw-r--.  1 miuser miuser    62152949 Apr  3 10:57 md-4404-big-Index.db
-rw-rw-r--.  1 miuser miuser       15480 Apr  3 10:57 md-4404-big-Statistics.db
-rw-rw-r--.  1 miuser miuser      573597 Apr  3 10:57 md-4404-big-Summary.db
-rw-rw-r--.  1 miuser miuser          92 Apr  3 10:57 md-4404-big-TOC.txt
-rw-rw-r--.  1 miuser miuser     6314387 Apr  3 10:51 md-4405-big-CompressionInfo.db
-rw-rw-r--.  1 miuser miuser 18077710948 Apr  3 10:51 md-4405-big-Data.db
-rw-rw-r--.  1 miuser miuser          10 Apr  3 10:51 md-4405-big-Digest.crc32
-rw-rw-r--.  1 miuser miuser     8613168 Apr  3 10:51 md-4405-big-Filter.db
-rw-rw-r--.  1 miuser miuser   144782406 Apr  3 10:51 md-4405-big-Index.db
-rw-rw-r--.  1 miuser miuser       15336 Apr  3 10:51 md-4405-big-Statistics.db
-rw-rw-r--.  1 miuser miuser     1326513 Apr  3 10:51 md-4405-big-Summary.db
-rw-rw-r--.  1 miuser miuser          92 Apr  3 10:51 md-4405-big-TOC.txt
drwxrwxrwx.  4 miuser miuser          92 Mar 26 11:53 snapshots
```

.person_locations_idx 目錄裡的檔案過大


* 修改 cassandra.ymal 設定 : 
```
# How long the coordinator should wait for read operations to complete
read_request_timeout_in_ms: 5000
# How long the coordinator should wait for seq or index scans to complete
range_request_timeout_in_ms: 10000
# How long the coordinator should wait for writes to complete
write_request_timeout_in_ms: 2000
# How long the coordinator should wait for counter writes to complete
counter_write_request_timeout_in_ms: 5000
# How long a coordinator should continue to retry a CAS operation
# that contends with other proposals for the same row
cas_contention_timeout_in_ms: 1000
```
改成 : 
```
# How long the coordinator should wait for read operations to complete
read_request_timeout_in_ms: 50000
# How long the coordinator should wait for seq or index scans to complete
range_request_timeout_in_ms: 100000
# How long the coordinator should wait for writes to complete
write_request_timeout_in_ms: 20000
# How long the coordinator should wait for counter writes to complete
counter_write_request_timeout_in_ms: 50000
# How long a coordinator should continue to retry a CAS operation
# that contends with other proposals for the same row
cas_contention_timeout_in_ms: 10000
```

給 conf 執行 spark-submit :  
```
$SPARK_HOME/bin/spark-shell --conf spark.cassandra.connection.timeout_ms=600000 \
                            --packages datastax:spark-cassandra-connector:2.4.1-s_2.11
```
或
```
val conf = new SparkConf(true)
        .set("spark.cassandra.connection.host", "192.168.123.10")
        .set("spark.cassandra.auth.username", "cassandra")            
        .set("spark.cassandra.auth.password", "cassandra")
        .config("spark.cassandra.connection.timeout_ms", 600000)
```



> 參考資料  
> [spark-cassandra-partitioning](https://github.com/datastax/spark-cassandra-connector/blob/master/doc/16_partitioning.md)





