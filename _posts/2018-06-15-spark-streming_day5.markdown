---
layout: post
title:  "Spark-Streaming day 5 (spark streaming window)"
date:   2018-06-15 08:44:17 +0800
categories: Spark-Streaming
---
#### 目標
算出每個時間區間的總數

#### StreamingWindowTest.scala
spark-streaming 每 5 秒觸發一次，
使用 reduceByWindow 這 function 會給 2 個參數，第一個是 window length，第二個是 sliding interval．  
這邊的例子就是每 5 秒計算一次 10 秒內出現的數字加總起來．

```scala
package com.streaming.test

import org.apache.spark.streaming.{Seconds, StreamingContext}
import org.apache.spark.{SparkConf, SparkContext}

object StreamingWindowTest {

  def main(args: Array[String]): Unit = {
    val conf = new SparkConf().setMaster("local[2]").setAppName("NetworkWordCount")
    val ssc = new StreamingContext(conf, Seconds(5))
    val lines = ssc.socketTextStream("localhost", 9988)
    val wordspair = lines.map(row => {
      if(row !=null && row != "") {
        val num = row.split(" ")(1);
        num.toInt
      } else {
        0
      }
    })
    val windowsRdds = wordspair.reduceByWindow((x:Int , y:Int) => {
      println(x + " + " + y + " = " + (x+y));
      (x + y)
    }, Seconds(10), Seconds(5)).print()
    ssc.start()
    ssc.awaitTermination()
  }
}

```

#### spark streaming checkpoint
使用 StreamingContext.getOrCreate 這方法，如果沒有 checkpoint file 則重新 new 一個 StreamingContext，
有的話直接讀取 checkpoint file 產生 StreamingContext．

#### StreamingWindowTest.scala 改寫

```scala
package com.streaming.test

import org.apache.spark.streaming.{Seconds, StreamingContext}
import org.apache.spark.SparkConf

object StreamingWindowTest {

  def main(args: Array[String]): Unit = {
    val ssc = StreamingContext.getOrCreate("/Users/daniel/test/checkpointfile" , functionToCreateContext)
    ssc.start()
    ssc.awaitTermination()
  }

  def functionToCreateContext(): StreamingContext = {
    val conf = new SparkConf().setMaster("local[2]").setAppName("NetworkWordCount")
    val ssc = new StreamingContext(conf, Seconds(5))
    ssc.checkpoint("/Users/daniel/test/checkpointfile")
    val lines = ssc.socketTextStream("localhost", 9988)
    val wordspair = lines.map(row => {
      if(row !=null && row != "") {
        val num = row.split(" ")(1);
        num.toInt
      } else {
        0
      }
    })
    val windowsRdds = wordspair.reduceByWindow((x:Int , y:Int) => {
      println(x + " + " + y + " = " + (x+y));
      (x + y)
    }, Seconds(10), Seconds(5)).print()
    ssc
  }
}
```
先 run 起來一次產生幾筆資料後，然後關掉從 run 可以看到下列訊息 :  

![spark-streaming_day5_1.jpg]({{ "/assets/spark-streaming/day5/spark-streaming_day5_1.jpg" | absolute_url }})







> 參考資料  
> [check point](https://blog.csdn.net/qq_36330643/article/details/74451402)

