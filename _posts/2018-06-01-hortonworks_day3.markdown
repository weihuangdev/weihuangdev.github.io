---
layout: post
title:  "hortonworks day 3 (Apache NiFi custom processor)"
date:   2018-06-01 08:44:17 +0800
categories: hortonworks
---

### custom processor

#### 建立一個目錄
* 建立 ChakraProcessor  
/Volumes/Transcend/1-eclipse-workspace/test-workspace/ChakraProcessor

#### 進入目錄內，下 maven 指令
mvn archetype:generate

#### 會問說要找那個 archetype，輸入 nifi
```console
Choose a number or apply filter (format: [groupId:]artifactId, case sensitive contains): 1198: nifi
Choose archetype:
1: remote -> org.apache.nifi:nifi-processor-bundle-archetype (-)
2: remote -> org.apache.nifi:nifi-service-bundle-archetype (-)
```
![kafka_day3_1.jpg]({{ "/assets/hortonworks/day3/htwks_day3_1.jpg" | absolute_url }})

#### 選 1 後，然後選版本 1.4.0，輸入 25 (1.5 及 1.6 mvn install 會失敗需再研究)

```console
Choose a number or apply filter (format: [groupId:]artifactId, case sensitive contains): : 1
Choose org.apache.nifi:nifi-processor-bundle-archetype version:
1: 0.0.2-incubating
2: 0.1.0-incubating
3: 0.2.0-incubating
4: 0.2.1
5: 0.3.0
6: 0.4.0
7: 0.4.1
8: 0.5.0
9: 0.5.1
10: 0.6.0
11: 0.6.1
12: 0.7.0
13: 0.7.1
14: 0.7.2
15: 0.7.3
16: 0.7.4
17: 1.0.0-BETA
18: 1.0.0
19: 1.0.1
20: 1.1.0
21: 1.1.1
22: 1.1.2
23: 1.2.0
24: 1.3.0
25: 1.4.0
26: 1.5.0
27: 1.6.0


Choose a number: 27: 25
```
![kafka_day3_2.jpg]({{ "/assets/hortonworks/day3/htwks_day3_2.jpg" | absolute_url }})

#### 輸入 groupId 及 artifactId 等資訊

```console
Define value for property 'groupId': mynifi
Define value for property 'artifactId': myprocessors
Define value for property 'version' 1.0-SNAPSHOT: : 1.0
Define value for property 'artifactBaseName': demo
Define value for property 'package' mynifi.processors.demo: : 
[INFO] Using property: nifiVersion = 1.6.0
Confirm properties configuration:
groupId: mynifi
artifactId: myprocessors
version: 1.0
artifactBaseName: demo
package: nifiVersion = 1.6.0
nifiVersion: 1.6.0
 Y
```
![kafka_day3_3.jpg]({{ "/assets/hortonworks/day3/htwks_day3_3.jpg" | absolute_url }})

#### 執行結果
![kafka_day3_4.jpg]({{ "/assets/hortonworks/day3/htwks_day3_4.jpg" | absolute_url }})

#### 抓下來的目錄結構
![kafka_day3_5.jpg]({{ "/assets/hortonworks/day3/htwks_day3_5.jpg" | absolute_url }})

#### 修改 MyProcessor.java 加上一些 Log
```java
@Override
public void onTrigger(final ProcessContext context, final ProcessSession session) throws ProcessException {
    FlowFile flowFile = session.get();
    if ( flowFile == null ) {
        return;
    }
    // TODO implement
    System.out.println("****** MyProcessor_Received a flow file ......");
    getLogger().info("####### INFO MyProcessor_Received a flow file ......");
    getLogger().debug("####### DEBUG MyProcessor_Received a flow file ......");
    getLogger().error("####### ERROR MyProcessor_Received a flow file ......");
    session.transfer(flowFile , MY_RELATIONSHIP);
}
```

#### mvn install
* 到 /Volumes/Transcend/1-eclipse-workspace/test-workspace/ChakraProcessor/myprocess 這層下 install
```console
mvn install
```
![kafka_day3_6.jpg]({{ "/assets/hortonworks/day3/htwks_day3_6.jpg" | absolute_url }})

#### 執行成功的目錄結構
![kafka_day3_7.jpg]({{ "/assets/hortonworks/day3/htwks_day3_7.jpg" | absolute_url }})

#### 將 compiler 成功的 nar 檔丟到 nifi 的 lib 底下
* compiler 的 nar :  
/Volumes/Transcend/1-eclipse-workspace/test-workspace/cust-demo/myprocess/nifi-demo-nar/target/nifi-demo-nar-1.0.nar
* sandbox nifi 的 lib 路徑 :  
/usr/hdf/3.1.0.0-564/nifi/lib 
* 連進 sandbox 指令 :  
ssh root@localhost -p 2222
* copy file 到 sandbox 指令  
```console
daniel@Danielde-MBP > /Volumes/Transcend/1-eclipse-workspace/test-workspace/cust-demo/myprocess/nifi-demo-nar/target > scp -P 2222 nifi-demo-nar-1.0.nar root@localhost:/usr/hdf/3.1.0.0-564/nifi/lib
root@localhost's password:
nifi-demo-nar-1.0.nar                             100%  143KB   3.0MB/s   00:00
```

#### 重啟 nifi
![kafka_day3_8.jpg]({{ "/assets/hortonworks/day3/htwks_day3_8.jpg" | absolute_url }})

#### 新增 MyProcessor
![kafka_day3_9.jpg]({{ "/assets/hortonworks/day3/htwks_day3_9.jpg" | absolute_url }})

#### 設定 MyProcessor 的 propety
* 程式有給定 propety 並設定 require 為 true，所以要給值
![kafka_day3_10.jpg]({{ "/assets/hortonworks/day3/htwks_day3_10.jpg" | absolute_url }})

#### 新增 LogAttribute Processor，並串連起來
![kafka_day3_11.jpg]({{ "/assets/hortonworks/day3/htwks_day3_11.jpg" | absolute_url }})

#### 執行全部的 Processor 後可以去看 nifi 的 log
* log 位置
![kafka_day3_12.jpg]({{ "/assets/hortonworks/day3/htwks_day3_12.jpg" | absolute_url }})
* MyProcessor 的 log 看來只有 INFO 和 ERROR 會印出來
![kafka_day3_13.jpg]({{ "/assets/hortonworks/day3/htwks_day3_13.jpg" | absolute_url }})

#### LogAttribute Processor 的 log 資訊
* LogAttribute Processor 會將 FlowFile 的 Attribute 列印出來
![kafka_day3_14.jpg]({{ "/assets/hortonworks/day3/htwks_day3_14.jpg" | absolute_url }})
* random 產生的 filename 也會在 FlowFile 的 Attribute 裡
![kafka_day3_15.jpg]({{ "/assets/hortonworks/day3/htwks_day3_15.jpg" | absolute_url }})

> 參考網址  
> [官網教學](https://community.hortonworks.com/articles/4318/build-custom-nifi-processor.html)  
> [custom processor 教學-1](https://www.youtube.com/watch?v=3ldmNFlelhw)  
> [custom processor 教學-2](https://www.youtube.com/watch?v=QRzVr82V_Is)  
> [nifi flowfile attributes](https://kisstechdocs.wordpress.com/2015/01/20/nifi-flowfile-attributes-an-analogy/)  












