---
layout: post
title:  "hortonworks day 2 (Apache NiFi)"
date:   2018-06-01 08:44:17 +0800
categories: hortonworks
---

### Apache NiFi

#### 準備兩個行程(Processor) : GenerateFlowFile(input)和 PutFile(output)  
GenerateFlowFlies will generate random texts as flowfiles.  
PutiFile will write the content from GenerateFlowFiles into the disk as many individual files.  

#### 新增一個 Processor GenerateFlowFile
![kafka_day2_1.jpg]({{ "/assets/hortonworks/day2/htwks_day2_1.jpg" | absolute_url }})
#### 將 Processor 拖拉一個圖示到下面空白處後會有選擇畫面 : 
![kafka_day2_2.jpg]({{ "/assets/hortonworks/day2/htwks_day2_2.jpg" | absolute_url }})
#### 接著在 filter 裡找 GenerateFlowFlies
![kafka_day2_3.jpg]({{ "/assets/hortonworks/day2/htwks_day2_3.jpg" | absolute_url }})
#### ADD 後會出現
![kafka_day2_4.jpg]({{ "/assets/hortonworks/day2/htwks_day2_4.jpg" | absolute_url }})
#### 按右鍵可以 config 
![kafka_day2_5.jpg]({{ "/assets/hortonworks/day2/htwks_day2_5.jpg" | absolute_url }})
#### 新增一個 Processor PutFile
![kafka_day2_6.jpg]({{ "/assets/hortonworks/day2/htwks_day2_6.jpg" | absolute_url }})
#### ADD 後會出現
![kafka_day2_7.jpg]({{ "/assets/hortonworks/day2/htwks_day2_7.jpg" | absolute_url }})
#### 修改 config 裡的值
![kafka_day2_8.jpg]({{ "/assets/hortonworks/day2/htwks_day2_8.jpg" | absolute_url }})
#### 給產出目錄位置 /tmp/nifi
![kafka_day2_9.jpg]({{ "/assets/hortonworks/day2/htwks_day2_9.jpg" | absolute_url }})
#### 建立processor的連結
![kafka_day2_10.jpg]({{ "/assets/hortonworks/day2/htwks_day2_10.jpg" | absolute_url }})
![kafka_day2_11.jpg]({{ "/assets/hortonworks/day2/htwks_day2_11.jpg" | absolute_url }})
#### 建立成功
![kafka_day2_12.jpg]({{ "/assets/hortonworks/day2/htwks_day2_12.jpg" | absolute_url }})
#### 讓 processor 都開始跑
![kafka_day2_13.jpg]({{ "/assets/hortonworks/day2/htwks_day2_13.jpg" | absolute_url }})
#### 在/tmp/nifi 底下產生跑的檔案
![kafka_day2_14.jpg]({{ "/assets/hortonworks/day2/htwks_day2_14.jpg" | absolute_url }})
#### 測試完後要關閉不然會產生很多檔案

> 參考網址
[hello-world-nifi](https://medium.com/@suci/hello-world-nifi-dcafcba0fdb0)  


