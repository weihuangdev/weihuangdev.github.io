---
layout: post
title:  "clickhouse day 4 (java call clickhouse)"
date:   2018-07-25 10:44:17 +0800
categories: clickhouse
---

參考 clickhouse 提供的 [third-party_client_libraries](https://clickhouse.yandex/docs/en/interfaces/third-party_client_libraries/)  

#### pom.xml

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.ckh.example</groupId>
  <artifactId>ckhClient</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <packaging>jar</packaging>

  <name>ckhClient</name>
  <url>http://maven.apache.org</url>

  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>

  <dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>3.8.1</version>
      <scope>test</scope>
    </dependency>
    <dependency>
		<groupId>com.github.virtusai</groupId>
		<artifactId>clickhouse-client-java</artifactId>
		<version>-SNAPSHOT</version>
	</dependency>
  </dependencies>
  
  
  <repositories>
	<repository>
		<id>jitpack.io</id>
		<url>https://jitpack.io</url>
	</repository>
</repositories>
  
</project>

```


#### java 測試程式

```java
package com.ckh.example.ckhClient;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutionException;

import com.ckh.example.entity.TestTables;
import com.virtusai.clickhouseclient.ClickHouseClient;
import com.virtusai.clickhouseclient.models.http.ClickHouseResponse;

public class ClickhouseOperater {

	public static void main(String[] args) {
		try {
			ClickHouseClient client = new ClickHouseClient("http://127.0.0.1:8123", "default", "");
			List<Object[]> rows = new ArrayList<Object[]>();
			rows.add(new Object[] {"2018-07-01","message_2018-07-01"});
			rows.add(new Object[] {"2018-07-02","message_2018-07-02"});
			rows.add(new Object[] {"2018-07-03","message_2018-07-03"});
			// 新增資料
			client.post("insert into testTables", rows);
			Thread.sleep(2000);

			//查詢資料
			CompletableFuture<ClickHouseResponse<TestTables>> comp = client.get("select * from testTables", TestTables.class);
			ClickHouseResponse<TestTables> response = comp.get();
			List<TestTables> datas = response.data;
			for(TestTables data : datas) {
				System.out.println(data.getDate() + " , " + data.getValue());
			}

			Thread.sleep(2000);
			client.close();
		} catch (InterruptedException e) {
			e.printStackTrace();
		} catch (ExecutionException e) {
			e.printStackTrace();
		}
		
	}

}
```
#### TestTables.java
```java
package com.ckh.example.entity;

public class TestTables {
	
	private String date;
	private String value;
	
	public String getDate() {
		return date;
	}
	public void setDate(String date) {
		this.date = date;
	}
	public String getValue() {
		return value;
	}
	public void setValue(String value) {
		this.value = value;
	}
	
	
}

```


#### 在 run clickhouse-server 的 images 時，要加上 -p 127.0.0.1:8123:8123 才可以讓本機的 8123 port 連到 container 的 8123 port
```console
docker run -d --name daniel-clickhouse-server -p 127.0.0.1:8123:8123 --ulimit nofile=262144:262144 yandex/clickhouse-server
```

#### 啟動 yandex/clickhouse-client 時是透過 --link daniel-clickhouse-server:clickhouse-server，讓兩台 continer 彼此能夠溝通
```
docker run -it --rm --link daniel-clickhouse-server:clickhouse-server yandex/clickhouse-client --host clickhouse-server
```

#### 執行結果
![clickhouse_day4_1.jpg]({{ "/assets/clickhouse/day4/clickhouse_day4_1.jpg" | absolute_url }})




