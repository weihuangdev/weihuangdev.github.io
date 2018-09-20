---
layout: post
title:  "cassandra day 2 (spark cassandra)"
date:   2018-09-20 11:44:17 +0800
categories: cassandra
---

#### build.sbt
```
name := "Enricher-ext"

version := "0.1"

scalaVersion := "2.11.12"

dependencyOverrides += "com.fasterxml.jackson.core" % "jackson-core" % "2.8.7"
dependencyOverrides += "com.fasterxml.jackson.core" % "jackson-databind" % "2.8.7"
dependencyOverrides += "com.fasterxml.jackson.module" % "jackson-module-scala_2.11" % "2.8.7"

libraryDependencies ++= Seq(
  "io.grpc"               %   "grpc-netty"            % "1.14.0",
  "io.grpc"               %   "grpc-okhttp"           % "1.14.0",
  "com.google.guava"      %   "guava"                 % "21.0",
  "org.apache.spark"      %%  "spark-core"            % "2.3.1" ,
  "org.apache.spark"      %%  "spark-sql"             % "2.3.1" ,
  "redis.clients"         %   "jedis"                 % "2.9.0",
  "com.crealytics"        %%  "spark-excel"           % "0.9.8",
  "com.datastax.spark" %% "spark-cassandra-connector" % "2.3.2",
  "org.scalatest"         %%  "scalatest"             % "3.0.5" % "test"
)

lazy val copyJar = taskKey[Unit]("Test copy jar file")
copyJar := {
  val folder = new File("target/lib")
  (managedClasspath in Compile).value.files.foreach { f =>
    IO.copyFile(f, folder / f.getName)
  }
  val jarFile = sbt.Keys.`package`.in(Compile, packageBin).value
  IO.copyFile(jarFile, new File("target") / jarFile.getName)
}

```
#### 需要 import connector
```
import org.apache.spark.sql.SparkSession
import com.datastax.spark.connector._
```

#### Query cassandra
```
val rdd = spark.sparkContext.cassandraTable("castest","prod")
println(rdd.count)
println(rdd.first)
rdd.foreach(raw => {
  val prodId = raw.get[String]("prodid")
  val prodCount = raw.get[Int]("count")
  println(prodId + " , " + prodCount)
})
```

#### query cassandra with where
```
spark.sparkContext.cassandraTable("castest","prod").select("count").where("count > ? " , 50).foreach(raw => {
	val prodCount = raw.get[Int]("count")
	println(prodCount)
})
```

#### insert cassandra 
```
val listElements = spark.sparkContext.parallelize(
  Seq(
  (5,100),
  (6,200),
  (7,300))
)
listElements.saveToCassandra("castest","prod",SomeColumns("prodid","count"))
```
#### update cassandra
```
val listElements = spark.sparkContext.parallelize(
  Seq(
    (6,100),
    (8,400)
  )
)
listElements.saveToCassandra("castest","prod",SomeColumns("prodid","count"))
```

#### delete cassandra 
```
val listElements = spark.sparkContext.parallelize(
  Seq(
    (6,100)
  )
)
listElements.deleteFromCassandra("castest","prod")
```
#### delete cassandra with where
```
spark.sparkContext.cassandraTable("castest","prod").where("count < 50").deleteFromCassandra("castest","prod")
```

> 參考資料  
> [spark-cassandra](https://github.com/datastax/spark-cassandra-connector)





