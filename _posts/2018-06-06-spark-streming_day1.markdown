---
layout: post
title:  "Spark-Streaming day 1 (Hello World)"
date:   2018-06-06 08:44:17 +0800
categories: Spark-Streaming
---
#### 目標
寫一個 hello world spark-streaming 程式，然後使用 netcat 工具做測試．

#### netcat 工具
netcat 的功能非常多這邊只先介紹如何測試的方法 :
* 安裝 netcat   
```console
[root@daniel-3-test-master1 ~]# yum install nc
Loaded plugins: fastestmirror
base                                                           | 3.6 kB  00:00:00
cloudera-manager                                               |  951 B  00:00:00
extras                                                         | 3.4 kB  00:00:00
local-cloudera                                                 | 2.9 kB  00:00:00
...
```

* 開一個 console 輸入 nc -lk 9999
```console
daniel@Danielde-MacBook-Pro > nc -lk 9999
```
* 再開另一個 console 輸入 telnet localhost 9999
```console
daniel@Danielde-MacBook-Pro > telnet localhost 9999
Trying ::1...
telnet: connect to address ::1: Connection refused
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
```
![spark-streaming_day1_1.jpg]({{ "/assets/spark-streaming/day1/spark-streaming_day1_1.jpg" | absolute_url }})
不管在哪個 cosole 輸入文字都會同步
![spark-streaming_day1_2.jpg]({{ "/assets/spark-streaming/day1/spark-streaming_day1_2.jpg" | absolute_url }})

* 將 netcat 的內容導到檔案裡 nc -lk 9999 > testNetCat.txt
![spark-streaming_day1_3.jpg]({{ "/assets/spark-streaming/day1/spark-streaming_day1_3.jpg" | absolute_url }})
telnet
![spark-streaming_day1_4.jpg]({{ "/assets/spark-streaming/day1/spark-streaming_day1_4.jpg" | absolute_url }})
看檔案內容的變化，只有從 telnet 的 console 輸入的內容才會導到檔案裡
![spark-streaming_day1_5.jpg]({{ "/assets/spark-streaming/day1/spark-streaming_day1_5.jpg" | absolute_url }})

#### 環境
CDH Spark2.2.0

#### 啟動 spark-shell
![spark-streaming_day1_6.jpg]({{ "/assets/spark-streaming/day1/spark-streaming_day1_6.jpg" | absolute_url }})

#### 在 spark-shell 裡面輸入下面程式  
ssc.socketTextStream("daniel-3-test-master1", 9988)  
這邊要輸入 domain name ，試過輸入 IP 或 localhost 會 Connection refused 原因可能再研究．

```java
import org.apache.spark._
import org.apache.spark.streaming._
val ssc = new StreamingContext(sc, Seconds(1))
val lines = ssc.socketTextStream("daniel-3-test-master1", 9988)
val words = lines.flatMap(_.split(" "))
val pairs = words.map(word => (word, 1))
val wordCounts = pairs.reduceByKey(_ + _)
wordCounts.print()
ssc.start()
```
然後執行 ssc.start()
![spark-streaming_day1_7.jpg]({{ "/assets/spark-streaming/day1/spark-streaming_day1_7.jpg" | absolute_url }})

在使用 netcat 輸入一些訊息
![spark-streaming_day1_8.jpg]({{ "/assets/spark-streaming/day1/spark-streaming_day1_8.jpg" | absolute_url }})

印出的結果如下，系統會每 1 秒執行一次 job
![spark-streaming_day1_9.jpg]({{ "/assets/spark-streaming/day1/spark-streaming_day1_9.jpg" | absolute_url }})

> 參考網址  
> [Spark DAG](https://blog.csdn.net/u011564172/article/details/70172060)  
> [Spark Streaming 架構](https://github.com/lw-lin/CoolplaySpark/blob/master/Spark%20Streaming%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E7%B3%BB%E5%88%97/0.1%20Spark%20Streaming%20%E5%AE%9E%E7%8E%B0%E6%80%9D%E8%B7%AF%E4%B8%8E%E6%A8%A1%E5%9D%97%E6%A6%82%E8%BF%B0.md)  
> [Spark Streaming read file](http://dblab.xmu.edu.cn/blog/1082-2/)







