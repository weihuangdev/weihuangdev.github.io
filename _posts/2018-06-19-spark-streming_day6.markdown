---
layout: post
title:  "Spark-Streaming day 6 (spark streaming csv files)"
date:   2018-06-19 08:44:17 +0800
categories: Spark-Streaming
---

#### spark 讀取 csv 檔

```scala
package com.streaming.test

import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.types._

object ReadCsvFile {

  def main(args: Array[String]): Unit = {
    val spark = SparkSession
      .builder
      .appName("Spark-csv")
      .master("local[2]")
      .getOrCreate()

    val mySchema = StructType(Array(
      StructField("TagName", StringType),
      StructField("TimeStamp", StringType),
      StructField("Min", IntegerType),
      StructField("Max", IntegerType),
      StructField("Avg", DoubleType)))

    val csvDf = spark.read.schema(mySchema).option("header","true").csv("/Volumes/Transcend/1-program-workspace/2-intellij-workspace/streaming-test/csvfile/C_2018-05-09_101732_L41-OPU-012.csv")
    csvDf.foreach(println(_))
  }
}

```

#### 讀取目錄底下所有的 csv 檔
```scala
package com.streaming.test

import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.types._

object ReadCsvFile {

  def main(args: Array[String]): Unit = {

    val userSchema = new StructType()
      .add("TagName", "string")
      .add("TimeStamp", "string")
      .add("Min", "integer")
      .add("Max", "integer")
      .add("Avg", "double")

    val spark = SparkSession
      .builder
      .appName("Spark-csv")
      .master("local[2]")
      .getOrCreate()

    val csvDF = spark.read
      .option("header","true")
      .schema(userSchema)
      .csv("/Volumes/Transcend/1-program-workspace/2-intellij-workspace/streaming-test/csvfile")
    csvDF.foreach(row => println(row))
  }
}

```


#### 使用 com.databricks.spark.cvs 格式讀取  
```scala
val userSchema = new StructType()
      .add("TagName", "string")
      .add("TimeStamp", "string")
      .add("Min", "integer")
      .add("Max", "integer")
      .add("Avg", "double")
val spark = SparkSession
  .builder
  .appName("Spark-csv")
  .master("local[2]")
  .getOrCreate()
val csvDir = "/Volumes/Transcend/1-program-workspace/2-intellij-workspace/streaming-test/csvfile"
spark.read.option("header","true").format("com.databricks.spark.cvs").csv(csvDir).foreach(println(_))
```






