---
layout: post
title:  "Spark-Streaming day 2 (intelliJ Hello World)"
date:   2018-06-06 08:44:17 +0800
categories: Spark-Streaming
---
#### 目標
使用 intelliJ 開發一個 Hello World 程式

#### install scala plugin
![spark-streaming_day2_1.jpg]({{ "/assets/spark-streaming/day2/spark-streaming_day2_1.jpg" | absolute_url }})
![spark-streaming_day2_2.jpg]({{ "/assets/spark-streaming/day2/spark-streaming_day2_2.jpg" | absolute_url }})
![spark-streaming_day2_3.jpg]({{ "/assets/spark-streaming/day2/spark-streaming_day2_3.jpg" | absolute_url }})
![spark-streaming_day2_4.jpg]({{ "/assets/spark-streaming/day2/spark-streaming_day2_4.jpg" | absolute_url }})
#### 建立一個新的 project
![spark-streaming_day2_5.jpg]({{ "/assets/spark-streaming/day2/spark-streaming_day2_5.jpg" | absolute_url }})
![spark-streaming_day2_6.jpg]({{ "/assets/spark-streaming/day2/spark-streaming_day2_6.jpg" | absolute_url }})
![spark-streaming_day2_7.jpg]({{ "/assets/spark-streaming/day2/spark-streaming_day2_7.jpg" | absolute_url }})
#### 寫一個測試程式 StreamingTest
![spark-streaming_day2_8.jpg]({{ "/assets/spark-streaming/day2/spark-streaming_day2_8.jpg" | absolute_url }})
![spark-streaming_day2_9.jpg]({{ "/assets/spark-streaming/day2/spark-streaming_day2_9.jpg" | absolute_url }})
![spark-streaming_day2_10.jpg]({{ "/assets/spark-streaming/day2/spark-streaming_day2_10.jpg" | absolute_url }})
#### 使用 intelliJ build jar
![spark-streaming_day2_11.jpg]({{ "/assets/spark-streaming/day2/spark-streaming_day2_11.jpg" | absolute_url }})
![spark-streaming_day2_12.jpg]({{ "/assets/spark-streaming/day2/spark-streaming_day2_12.jpg" | absolute_url }})
![spark-streaming_day2_13.jpg]({{ "/assets/spark-streaming/day2/spark-streaming_day2_13.jpg" | absolute_url }})
![spark-streaming_day2_14.jpg]({{ "/assets/spark-streaming/day2/spark-streaming_day2_14.jpg" | absolute_url }})
![spark-streaming_day2_15.jpg]({{ "/assets/spark-streaming/day2/spark-streaming_day2_15.jpg" | absolute_url }})
![spark-streaming_day2_16.jpg]({{ "/assets/spark-streaming/day2/spark-streaming_day2_16.jpg" | absolute_url }})
![spark-streaming_day2_17.jpg]({{ "/assets/spark-streaming/day2/spark-streaming_day2_17.jpg" | absolute_url }})
#### 啟動一個 netcat
![spark-streaming_day2_18.jpg]({{ "/assets/spark-streaming/day2/spark-streaming_day2_18.jpg" | absolute_url }})
#### 先直接 run main function 測試看看
![spark-streaming_day2_19.jpg]({{ "/assets/spark-streaming/day2/spark-streaming_day2_19.jpg" | absolute_url }})
#### 出現下列錯誤訊息
![spark-streaming_day2_20.jpg]({{ "/assets/spark-streaming/day2/spark-streaming_day2_20.jpg" | absolute_url }})
#### 修改 sbt 檔，從 2.10 版變成 2.11
![spark-streaming_day2_21.jpg]({{ "/assets/spark-streaming/day2/spark-streaming_day2_21.jpg" | absolute_url }})
#### 測試執行成功
![spark-streaming_day2_22.jpg]({{ "/assets/spark-streaming/day2/spark-streaming_day2_22.jpg" | absolute_url }})

#### 接著 demo 讓 spark-streaming 從某個目錄抓取檔案內容處理
在本機建立一個目錄路徑為 /Users/daniel/test/streamFile  

#### 修改程式
將原來的 ssc.socketTextStream 改成 ssc.textFileStream
```scala
import org.apache.spark._
import org.apache.spark.streaming._

object StreamingTest {

  def main(args: Array[String]): Unit = {
    val conf = new SparkConf().setMaster("local[2]").setAppName("NetworkWordCount")
    val ssc = new StreamingContext(conf, Seconds(5))
    //val lines = ssc.socketTextStream("localhost", 9988)
    val lines = ssc.textFileStream("file:///Users/daniel/test/streamFile")
    val words = lines.flatMap(_.split(" "))
    val pairs = words.map(word => (word, 1))
    val wordCounts = pairs.reduceByKey(_ + _)
    wordCounts.print()
    ssc.start()
    ssc.awaitTermination()
  }
}
```
#### 測試執行成功
![spark-streaming_day2_23.jpg]({{ "/assets/spark-streaming/day2/spark-streaming_day2_23.jpg" | absolute_url }})


#### 使用 spark2-submit 測試
* 流程圖
![spark-streaming_day2_24.jpg]({{ "/assets/spark-streaming/day2/spark-streaming_day2_24.jpg" | absolute_url }})

#### 1.先把 compiler 好的 jar 丟到 master1 上面
```console
scp streaming-test.jar root@192.168.61.105:/tmp/streaming-test
```
#### 2.在使用 spark2-submit 把 spark job 送到 yarn 上執行
```console
spark2-submit --class com.streaming.test.StreamingTest --master yarn /tmp/streaming-test/streaming-test.jar
```
* 成功後會再 yarn 上看到執行的 job
![spark-streaming_day2_25.jpg]({{ "/assets/spark-streaming/day2/spark-streaming_day2_25.jpg" | absolute_url }})
#### 3.再把檔案(testFile.txt)丟到該目錄底下(/tmp/streaming-test/testfiles) 
* 但 master1 及 slave 1 都要有該檔案及目錄，因為是分散式處理的關係
![spark-streaming_day2_26.jpg]({{ "/assets/spark-streaming/day2/spark-streaming_day2_26.jpg" | absolute_url }})
![spark-streaming_day2_27.jpg]({{ "/assets/spark-streaming/day2/spark-streaming_day2_27.jpg" | absolute_url }})
#### 會在 master1 上看到執行結果
![spark-streaming_day2_28.jpg]({{ "/assets/spark-streaming/day2/spark-streaming_day2_28.jpg" | absolute_url }})







