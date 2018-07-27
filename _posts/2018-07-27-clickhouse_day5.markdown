---
layout: post
title:  "clickhouse day 5 (clickhouse Distributed table)"
date:   2018-07-27 10:44:17 +0800
categories: clickhouse
---



#### docker-compose.yaml

```
version: "3.2"
services:
  ch-server-1:
    image: yandex/clickhouse-server
    volumes:
      - ./config.xml:/etc/clickhouse-server/config.xml
      - ./include_from.xml:/etc/clickhouse-server/include_from.xml
      - ./macros1.xml:/etc/clickhouse-server/macros.xml
      - ./hosts:/etc/hosts
    ports:
      - 8123:8123
  ch-server-2:
    image: yandex/clickhouse-server
    volumes:
      - ./config.xml:/etc/clickhouse-server/config.xml
      - ./include_from.xml:/etc/clickhouse-server/include_from.xml
      - ./macros2.xml:/etc/clickhouse-server/macros.xml
      - ./hosts:/etc/hosts
  ch-server-3:
    image: yandex/clickhouse-server
    volumes:
      - ./config.xml:/etc/clickhouse-server/config.xml
      - ./macros3.xml:/etc/clickhouse-server/macros.xml
      - ./hosts:/etc/hosts
      - ./include_from.xml:/etc/clickhouse-server/include_from.xml
  ch-client:
    image: yandex/clickhouse-client
    entrypoint:
      - /bin/sleep
    command:
      - infinity
    volumes:
      - ./hosts:/etc/hosts
  zookeeper:
    image: zookeeper
    volumes:
      - ./hosts:/etc/hosts

```


#### clickhouse 的 config.xml 設定

```xml
<?xml version="1.0"?>
<yandex>
    <logger>
        <level>trace</level>
        <log>/var/log/clickhouse-server/clickhouse-server.log</log>
        <errorlog>/var/log/clickhouse-server/clickhouse-server.err.log</errorlog>
        <size>1000M</size>
        <count>10</count>
    </logger>


    <http_port>8123</http_port>

    <!--
    <https_port>8443</https_port>
    -->
    <!-- Used only with https_port. Full ssl options list: https://github.com/yandex/ClickHouse/blob/master/contrib/libpoco/NetSSL_OpenSSL/include/Poco/Net/SSLManager.h#L71 -->
    <openSSL>
        <server>
            <!-- openssl req -subj "/CN=localhost" -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout /etc/clickhouse-server/server.key -out /etc/clickhouse-server/server.crt -->
            <certificateFile>/etc/clickhouse-server/server.crt</certificateFile>
            <privateKeyFile>/etc/clickhouse-server/server.key</privateKeyFile>
            <!-- openssl dhparam -out /etc/clickhouse-server/dhparam.pem 4096 -->
            <dhParamsFile>/etc/clickhouse-server/dhparam.pem</dhParamsFile>
            <verificationMode>none</verificationMode>
            <loadDefaultCAFile>true</loadDefaultCAFile>
            <cacheSessions>true</cacheSessions>
            <disableProtocols>sslv2,sslv3</disableProtocols>
            <preferServerCiphers>true</preferServerCiphers>
        </server>
        <client>
            <loadDefaultCAFile>true</loadDefaultCAFile>
            <cacheSessions>true</cacheSessions>
            <disableProtocols>sslv2,sslv3</disableProtocols>
            <preferServerCiphers>true</preferServerCiphers>
            <!-- Use for self-signed: <verificationMode>none</verificationMode> -->
            <invalidCertificateHandler>
                <!-- Use for self-signed: <name>AcceptCertificateHandler</name> -->
                <name>RejectCertificateHandler</name>
            </invalidCertificateHandler>
        </client>
    </openSSL>

    <!-- Default root page on http[s] server. For example load UI from https://tabix.io/ when opening http://localhost:8123 -->
    <!--
    <http_server_default_response><![CDATA[<html ng-app="SMI2"><head><base href="http://ui.tabix.io/"></head><body><div ui-view="" class="content-ui"></div><script src="http://loader.tabix.io/master.js"></script></body></html>]]></http_server_default_response>
    -->

    <tcp_port>9000</tcp_port>

    <!-- Port for communication between replicas. Used for data exchange. -->
    <interserver_http_port>9009</interserver_http_port>

    <!-- Hostname that is used by other replicas to request this server.
         If not specified, than it is determined analoguous to 'hostname -f' command.
         This setting could be used to switch replication to another network interface.
      -->
    <!--
    <interserver_http_host>example.yandex.ru</interserver_http_host>
    -->
    <networks incl="networks" replace="replace">
        <ip>::/0</ip>
    </networks>

    <!-- Listen specified host. use :: (wildcard IPv6 address), if you want to accept connections both with IPv4 and IPv6 from everywhere. -->
    <listen_host>::</listen_host>
    <!-- <listen_host>::1</listen_host> -->
    <!-- <listen_host>0.0.0.0</listen_host> -->

    <max_connections>4096</max_connections>
    <keep_alive_timeout>3</keep_alive_timeout>

    <!-- Maximum number of concurrent queries. -->
    <max_concurrent_queries>100</max_concurrent_queries>

    <!-- Set limit on number of open files (default: maximum). This setting makes sense on Mac OS X because getrlimit() fails to retrieve
         correct maximum value. -->
    <!-- <max_open_files>262144</max_open_files> -->

    <!-- Size of cache of uncompressed blocks of data, used in tables of MergeTree family.
         In bytes. Cache is single for server. Memory is allocated only on demand.
         Cache is used when 'use_uncompressed_cache' user setting turned on (off by default).
         Uncompressed cache is advantageous only for very short queries and in rare cases.
      -->
    <uncompressed_cache_size>8589934592</uncompressed_cache_size>

    <!-- Approximate size of mark cache, used in tables of MergeTree family.
         In bytes. Cache is single for server. Memory is allocated only on demand.
         You should not lower this value.
      -->
    <mark_cache_size>5368709120</mark_cache_size>


    <!-- Path to data directory, with trailing slash. -->
    <path>/var/lib/clickhouse/</path>

    <!-- Path to temporary data for processing hard queries. -->
    <tmp_path>/var/lib/clickhouse/tmp/</tmp_path>

    <!-- Path to configuration file with users, access rights, profiles of settings, quotas. -->
    <users_config>users.xml</users_config>

    <!-- Default profile of settings.. -->
    <default_profile>default</default_profile>

    <!-- Default database. -->
    <default_database>default</default_database>

    <!-- Server time zone could be set here.
         Time zone is used when converting between String and DateTime types,
          when printing DateTime in text formats and parsing DateTime from text,
          it is used in date and time related functions, if specific time zone was not passed as an argument.
         Time zone is specified as identifier from IANA time zone database, like UTC or Africa/Abidjan.
         If not specified, system time zone at server startup is used.
         Please note, that server could display time zone alias instead of specified name.
         Example: W-SU is an alias for Europe/Moscow and Zulu is an alias for UTC.
    -->
    <!-- <timezone>Europe/Moscow</timezone> -->

    <!-- You can specify umask here (see "man umask"). Server will apply it on startup.
         Number is always parsed as octal. Default umask is 027 (other users cannot read logs, data files, etc; group can only read).
    -->
    <!-- <umask>022</umask> -->

    <!-- Configuration of clusters that could be used in Distributed tables.
         https://clickhouse.yandex/reference_en.html#Distributed
      -->
    <include_from>/etc/clickhouse-server/include_from.xml</include_from>
    <remote_servers incl="clickhouse_remote_servers" />

    <!-- If element has 'incl' attribute, then for it's value will be used corresponding substitution from another file.
         By default, path to file with substitutions is /etc/metrika.xml. It could be changed in config in 'include_from' element.
         Values for substitutions are specified in /yandex/name_of_substitution elements in that file.
      -->

    <!-- ZooKeeper is used to store metadata about replicas, when using Replicated tables.
         Optional. If you don't use replicated tables, you could omit that.
         See https://clickhouse.yandex/reference_en.html#Data%20replication
      -->
    <zookeeper incl="zookeeper-servers" optional="true" />
    <!-- Substitutions for parameters of replicated tables.
          Optional. If you don't use replicated tables, you could omit that.
         See https://clickhouse.yandex/reference_en.html#Creating%20replicated%20tables
      -->
    <!-- <macros incl="macros" optional="true" /> -->
    <macros>
        <shard>01</shard>
        <replica>ch1</replica>
    </macros>


    <!-- Reloading interval for embedded dictionaries, in seconds. Default: 3600. -->
    <builtin_dictionaries_reload_interval>3600</builtin_dictionaries_reload_interval>


    <!-- Maximum session timeout, in seconds. Default: 3600. -->
    <max_session_timeout>3600</max_session_timeout>

    <!-- Default session timeout, in seconds. Default: 60. -->
    <default_session_timeout>60</default_session_timeout>

    <!-- Sending data to Graphite for monitoring. Several sections can be defined. -->
    <!--
        interval - send every X second
        root_path - prefix for keys
        metrics - send data from table system.metrics
        events - send data from table system.events
        asynchronous_metrics - send data from table system.asynchronous_metrics
    -->
    <!--
    <graphite>
        <host>localhost</host>
        <port>42000</port>
        <timeout>0.1</timeout>
        <interval>60</interval>
        <root_path>one_min</root_path>
        <metrics>true</metrics>
        <events>true</events>
        <asynchronous_metrics>true</asynchronous_metrics>
    </graphite>
    <graphite>
        <host>localhost</host>
        <port>42000</port>
        <timeout>0.1</timeout>
        <interval>1</interval>
        <root_path>one_sec</root_path>
        <metrics>true</metrics>
        <events>true</events>
        <asynchronous_metrics>false</asynchronous_metrics>
    </graphite>
    -->


    <!-- Query log. Used only for queries with setting log_queries = 1. -->
    <query_log>
        <!-- What table to insert data. If table is not exist, it will be created.
             When query log structure is changed after system update,
              then old table will be renamed and new table will be created automatically.
        -->
        <database>system</database>
        <table>query_log</table>

        <!-- Interval of flushing data. -->
        <flush_interval_milliseconds>7500</flush_interval_milliseconds>
    </query_log>


    <!-- Uncomment if use part_log
    <part_log>
        <database>system</database>
        <table>part_log</table>
        <flush_interval_milliseconds>7500</flush_interval_milliseconds>
    </part_log>
    -->


    <!-- Parameters for embedded dictionaries, used in Yandex.Metrica.
         See https://clickhouse.yandex/reference_en.html#Internal%20dictionaries
    -->

    <!-- Path to file with region hierarchy. -->
    <!-- <path_to_regions_hierarchy_file>/opt/geo/regions_hierarchy.txt</path_to_regions_hierarchy_file> -->

    <!-- Path to directory with files containing names of regions -->
    <!-- <path_to_regions_names_files>/opt/geo/</path_to_regions_names_files> -->


    <!-- Configuration of external dictionaries. See:
         https://clickhouse.yandex/reference_en.html#External%20Dictionaries
    -->
    <dictionaries_config>*_dictionary.xml</dictionaries_config>

    <!-- Uncomment if you want data to be compressed 30-100% better.
         Don't do that if you just started using ClickHouse.
      -->

    <compression incl="clickhouse_compression">
    <!--
        <!- - Set of variants. Checked in order. Last matching case wins. If nothing matches, lz4 will be used. - ->
        <case>
            <!- - Conditions. All must be satisfied. Some conditions may be omitted. - ->
            <min_part_size>10000000000</min_part_size>        <!- - Min part size in bytes. - ->
            <min_part_size_ratio>0.01</min_part_size_ratio>    <!- - Min size of part relative to whole table size. - ->
            <!- - What compression method to use. - ->
            <method>zstd</method>    <!- - Keep in mind that zstd compression library is highly experimental. - ->
        </case>
    -->
    </compression>

    <resharding>
        <task_queue_path>/clickhouse/task_queue</task_queue_path>
    </resharding>

    <!-- Allow to execute distributed DDL queries (CREATE, DROP, ALTER, RENAME) on cluster.
         Works only if ZooKeeper is enabled. Comment it if such functionality isn't required. -->
    <distributed_ddl>
        <!-- Path in ZooKeeper to queue with DDL queries -->
        <path>/clickhouse/task_queue/ddl</path>
    </distributed_ddl>

    <!-- Settings to fine tune MergeTree tables. See documentation in source code, in MergeTreeSettings.h -->
    <!--
    <merge_tree>
        <max_suspicious_broken_parts>5</max_suspicious_broken_parts>
    </merge_tree>
    -->

    <!-- Protection from accidental DROP.
         If size of a MergeTree table is greater than max_table_size_to_drop (in bytes) than table could not be dropped with any DROP query.
         If you want do delete one table and don't want to restart clickhouse-server, you could create special file <clickhouse-path>/flags/force_drop_table and make DROP once.
         By default max_table_size_to_drop is 50GB, max_table_size_to_drop=0 allows to DROP any tables.
         Uncomment to disable protection.
    -->
    <!-- <max_table_size_to_drop>0</max_table_size_to_drop> -->

    <!-- Example of parameters for GraphiteMergeTree table engine -->
    <graphite_rollup_example>
        <pattern>
            <regexp>click_cost</regexp>
            <function>any</function>
            <retention>
                <age>0</age>
                <precision>3600</precision>
            </retention>
            <retention>
                <age>86400</age>
                <precision>60</precision>
            </retention>
        </pattern>
        <default>
            <function>max</function>
            <retention>
                <age>0</age>
                <precision>60</precision>
            </retention>
            <retention>
                <age>3600</age>
                <precision>300</precision>
            </retention>
            <retention>
                <age>86400</age>
                <precision>3600</precision>
            </retention>
        </default>
    </graphite_rollup_example>
</yandex>
```

#### include_from.xml 設定

```xml
<yandex>
    <clickhouse_remote_servers>
        <ontime_cluster>
            <shard>
                <replica>
                    <default_database>r0</default_database>
                    <host>ch-server-1</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <default_database>r1</default_database>
                    <host>ch-server-3</host>
                    <port>9000</port>
                </replica>
            </shard>
            <shard>
                <replica>
                    <default_database>r0</default_database>
                    <host>ch-server-2</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <default_database>r1</default_database>
                    <host>ch-server-1</host>
                    <port>9000</port>
                </replica>
            </shard>
            <shard>
                <replica>
                    <default_database>r0</default_database>
                    <host>ch-server-3</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <default_database>r1</default_database>
                    <host>ch-server-2</host>
                    <port>9000</port>
                </replica>
            </shard>
        </ontime_cluster>
    </clickhouse_remote_servers>
    <zookeeper>
        <node>
            <host>zookeeper</host>
            <port>2181</port>
        </node>
    </zookeeper>
<networks>
   <ip>::/0</ip>
</networks>
    <clickhouse_compression>
    <!--
        <!- - Set of variants. Checked in order. Last matching case wins. If nothing matches, lz4 will be used. - ->
        <case>

            <!- - Conditions. All must be satisfied. Some conditions may be omitted. - ->
            <min_part_size>10000000000</min_part_size>        <!- - Min part size in bytes. - ->
            <min_part_size_ratio>0.01</min_part_size_ratio>   <!- - Min size of part relative to whole table size. - ->

            <!- - What compression method to use. - ->
            <method>zstd</method>
        </case>
    -->
</clickhouse_compression>
</yandex>

```

#### /etc/hosts 設定

```
127.0.0.1 localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
172.18.0.5 clickhouse-replication-example-master_ch-client_1
172.18.0.2 clickhouse-replication-example-master_ch-server-1_1 ch-server-1
172.18.0.6 clickhouse-replication-example-master_ch-server-2_1 ch-server-2
172.18.0.4 clickhouse-replication-example-master_ch-server-3_1 ch-server-3

```


#### 建立 database r0 與 r1
```
create database r0
create database r1
```

#### 在每一台 clickhouse server 的 r0 database 建立 ontime_local 的 table

```
CREATE TABLE ontime_local (FlightDate Date,Year UInt16) ENGINE = MergeTree(FlightDate, (Year, FlightDate), 8192)
```

#### 在每一台 clickhouse server 的 r0 database 建立 ontime_all 的 Distributed table

```
CREATE TABLE ontime_all AS ontime_local ENGINE = Distributed(ontime_cluster, r0, ontime_local, rand())
```
#### 新增資料做測試

```
insert into ontime_local (FlightDate,Year)values('2001-11-11',2002)
insert into ontime_all (FlightDate,Year)values('2001-10-10',2001)
```

#### 執行結果

```
3cc9fa1f21de :) select * from ontime_all

SELECT *
FROM ontime_all

┌─FlightDate─┬─Year─┐
│ 2001-10-12 │ 2001 │
└────────────┴──────┘
┌─FlightDate─┬─Year─┐
│ 2001-11-12 │ 2002 │
└────────────┴──────┘

2 rows in set. Elapsed: 0.008 sec.

```

#### mac 不支援 ln tmp/data.txt datalink.txt
* 所以當 docker compose 的 volumes 有設定 - ./data/1:/var/lib/clickhouse 時，然後下 insert data 到 Distributed table 時會出現下列錯誤訊息 :  

```
2018.07.27 07:23:03.980974 [ 24 ] <Debug> executeQuery: (from 172.18.0.5:45980, query_id: 812384f4-79c8-4643-8fc7-8603f87eb628) insert into ontime_all (FlightDate,Year)values
2018.07.27 07:23:04.035044 [ 24 ] <Error> executeQuery: Code: 0, e.displayText() = DB::Exception: Could not link /var/lib/clickhouse/data/r0/ontime_all/default@ch%2Dserver%2D2:9000#r0/1.bin to /var/lib/clickhouse/data/r0/ontime_all/default@ch%2Dserver%2D2:9000#r0/tmp/1.bin, errno: 95, strerror: Operation not supported, e.what() = DB::Exception (from 172.18.0.5:45980) (in query: insert into ontime_all (FlightDate,Year)values), Stack trace:

```

#### 可查 system.clusters 看 clusters 的設定

```
804fab967898 :) select * from system.clusters

SELECT *
FROM system.clusters

┌─cluster────────┬─shard_num─┬─shard_weight─┬─replica_num─┬─host_name───┬─host_address─┬─port─┬─is_local─┬─user────┬─default_database─┐
│ ontime_cluster │         1 │            1 │           1 │ ch-server-1 │ 172.18.0.2   │ 9000 │        0 │ default │ r0               │
│ ontime_cluster │         1 │            1 │           2 │ ch-server-3 │ 172.18.0.4   │ 9000 │        0 │ default │ r1               │
│ ontime_cluster │         2 │            1 │           1 │ ch-server-2 │ 172.18.0.6   │ 9000 │        0 │ default │ r0               │
│ ontime_cluster │         2 │            1 │           2 │ ch-server-1 │ 172.18.0.2   │ 9000 │        0 │ default │ r1               │
│ ontime_cluster │         3 │            1 │           1 │ ch-server-3 │ 172.18.0.4   │ 9000 │        0 │ default │ r0               │
│ ontime_cluster │         3 │            1 │           2 │ ch-server-2 │ 172.18.0.6   │ 9000 │        0 │ default │ r1               │
└────────────────┴───────────┴──────────────┴─────────────┴─────────────┴──────────────┴──────┴──────────┴─────────┴──────────────────┘

6 rows in set. Elapsed: 0.003 sec.
```

#### clickhouse server 的 log 會記錄在
```
/var/log/clickhouse-server/clickhouse-server.log
/var/log/clickhouse-server/clickhouse-server.err.log
```

#### 當 container 沒裝 ifconfig 時可先用 hostname -I 查 IP
```
root@42db1eabc7d6:/# hostname -I
172.18.0.5
```

#### docker-compose 的一些指令
```
docker-compose up

docker-compose ps

docker-compose logs

docker-compose rm

docker-compose down
```


#### 使用 java 程式做測試

```java
	@Test
	public void queryDistributedTableTest() {
		try {
			//use clickhouse HTTP interface
			ClickHouseClient client = new ClickHouseClient("http://127.0.0.1:8123/?database=r0", "default","");
			/*
			List<Object[]> rows = new ArrayList<Object[]>();
			rows.add(new Object[] {"2018-07-01","2018"});
			rows.add(new Object[] {"2018-07-02","2018"});
			rows.add(new Object[] {"2018-07-03","2018"});
			client.post("insert into r0.ontime_all", rows);
			Thread.sleep(2000);
			*/
			
			CompletableFuture<ClickHouseResponse<OntimeAll>> comp = client.get("select * from ontime_all", OntimeAll.class);
			ClickHouseResponse<OntimeAll> response = comp.get();
			List<OntimeAll> datas = response.data;
			System.out.println("data count is " + datas.size());
			for(OntimeAll data : datas) {
				System.out.println(data.getFlightDate() + " , " + data.getYear());
			}
			
			Thread.sleep(2000);
			client.close();
		} catch (InterruptedException e) {
			e.printStackTrace();
		} catch (ExecutionException e) {
			e.printStackTrace();
		}
	}
```
#### OntimeAll.java

```java
package com.ckh.example.entity;

public class OntimeAll {
	
	//要定的跟 table 欄位名稱一模一樣 大小寫有差
	private String FlightDate;
	private String Year;
	
	public String getFlightDate() {
		return FlightDate;
	}
	public void setFlightDate(String flightDate) {
		FlightDate = flightDate;
	}
	public String getYear() {
		return Year;
	}
	public void setYear(String year) {
		Year = year;
	}
}

```
#### 執行結果

![clickhouse_day5_1.jpg]({{ "/assets/clickhouse/day5/clickhouse_day5_1.jpg" | absolute_url }})



> 參考資料  
> [table_engines_distributed](http://clickhouse-docs.readthedocs.io/en/latest/table_engines/distributed.html)  
> [github 設定參考-1](https://github.com/sonych/clickhouse-cluster)  
> [github 設定參考-2](https://github.com/abraithwaite/clickhouse-replication-example)  
> [clickhouse cluster 參考](https://www.jianshu.com/p/ae45e0aa2b52)





