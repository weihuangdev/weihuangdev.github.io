---
layout: post
title:  "RabbitMQ day 1 (install)"
date:   2018-01-04 10:44:17 +0800
categories: RabbitMQ
---

### RabbitMQ 安裝
這邊使用的是 CentOS-6.8 安裝 rabbitmq． 
安裝 rabbitmq 前要先安裝 erlang，安裝指令步驟 : 

1. yum install epel-release (安裝epel擴充資源庫)
2. yum install erlang
3. 確認 erlang 版本指令 : erl -version
4. yum install rabbitmq-server

預設開機要啟動 : 

```console
chkconfig rabbitmq-server on
```
啟動 rabbitmq server : 

```console
service rabbitmq-server start
```
查看 rabbitmq server 狀態 : 

```console
service rabbitmq-server status
```
停止 rabbitmq server : 

```console
service rabbitmq-server stop
```
### 安裝 web 管理介面
到 /usr/lib/rabbitmq/lib/rabbitmq_server-3.1.5/sbin 底下執行 : 
```console
./rabbitmq-plugins enable rabbitmq_management
```
重啟 rabbitmq server 後，輸入網址 http://localhost:15672，帳密都使用 guest 登入．

![rabbitmq_day1_1.jpg]({{ "/assets/rabbitmq_day1_1.jpg" | absolute_url }})

要連到 centos 先關閉防火牆，下 setup :  

![rabbitmq_day1_2.jpg]({{ "/assets/rabbitmq_day1_2.jpg" | absolute_url }})

不要勾選，接著 OK 確認  

![rabbitmq_day1_3.jpg]({{ "/assets/rabbitmq_day1_3.jpg" | absolute_url }})

### Run Sample Code : 
Maven project pom.xml :  

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.simple</groupId>
  <artifactId>rabt</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <packaging>jar</packaging>
  <name>rabt</name>
  <url>http://maven.apache.org</url>
  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>
  <dependencies>
  <dependency>
      <groupId>com.rabbitmq</groupId>
      <artifactId>amqp-client</artifactId>
      <version>3.1.4</version>
  </dependency>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>3.8.1</version>
      <scope>test</scope>
    </dependency>
  </dependencies>
</project>
```

Producer.java :  

```java
package com.simple.rabt;

import java.io.IOException;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

public class Producer {
  
  private final static String QUEUE_NAME = "hello";
  
  public static void main(String[] args) {
    try {
      ConnectionFactory factory = new ConnectionFactory();
          factory.setHost("192.168.56.102");
      Connection connection = factory.newConnection();
      Channel channel = connection.createChannel();
      channel.queueDeclare(QUEUE_NAME, false, false, false, null);  
          String message = "Hello World!";  
          channel.basicPublish("", QUEUE_NAME, null, message.getBytes("UTF-8"));  
          System.out.println("Produce [x] Send '" + message + "'");  
          channel.close();
          connection.close();
    } catch (IOException e) {
      e.printStackTrace();
    }   
  }
}

```
Consumer.java :  

```java
package com.simple.rabt;

import java.io.IOException;

import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.DefaultConsumer;
import com.rabbitmq.client.Envelope;

public class Consumer {
  
  private final static String QUEUE_NAME = "hello";
  
  public static void main(String[] args) {
    
    try {
      ConnectionFactory factory = new ConnectionFactory();  
      factory.setHost("192.168.56.102");  
      Connection connection = factory.newConnection();
      Channel channel = connection.createChannel();  
          channel.queueDeclare(QUEUE_NAME, false, false, false, null);  
          System.out.println("Consume [*] Waiting for messages. To exit press CTRL+C");  
          DefaultConsumer consumer = new DefaultConsumer(channel) {
              public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {  
                  String message = new String(body, "UTF-8");  
                  System.out.println("Consume [x] Received '" + message + "'");  
              }  
          };  
          channel.basicConsume(QUEUE_NAME, true, consumer);  
    } catch (IOException e) {
      e.printStackTrace();
    }  
  }
}

```
接著先執行 Consumer :  

![rabbitmq_day1_4.jpg]({{ "/assets/rabbitmq_day1_4.jpg" | absolute_url }})

在 web 管理介面上，會看到有 Connections 在上面 :  

![rabbitmq_day1_5.jpg]({{ "/assets/rabbitmq_day1_5.jpg" | absolute_url }})

上面看到的 IP 是 192.168.56.1 這 IP 是本機透過這個 Gateway 連到 Virtual Box 的 IP 192.168.56.102．
附上我 Virtual Box 網卡的設定，設了兩張網卡(NAT是為了連外網、僅限主機介面卡是與本機溝通，在本機下 ifconfig 可看到192.168.56.1) :  

![rabbitmq_day1_8.jpg]({{ "/assets/rabbitmq_day1_8.jpg" | absolute_url }})

![rabbitmq_day1_9.jpg]({{ "/assets/rabbitmq_day1_9.jpg" | absolute_url }})

![rabbitmq_day1_10.jpg]({{ "/assets/rabbitmq_day1_10.jpg" | absolute_url }})

接著再執行 Producer :  

![rabbitmq_day1_6.jpg]({{ "/assets/rabbitmq_day1_6.jpg" | absolute_url }})

Consumer 會接到 Producer 送出的 message "Hello World" :  

![rabbitmq_day1_7.jpg]({{ "/assets/rabbitmq_day1_7.jpg" | absolute_url }})


### 總結
- - -
今天安裝好了 RabbitMQ 也 run 了 Hello World Sample Code，之後再詳細介紹 RabbitMQ 的工作及原理．


### 補充
- - -
* 查看 yum 目前 repositories
yum repolist

* 查詢 Centos 版本
cat /etc/redhat-release //查詢 Centos 版本

* netstat 顯示與IP、TCP、UDP和ICMP協定相關的統計資料，一般用於檢驗本機各埠的網路連接情況
netstat -ntpl | grep 5672

* centos 6.8 設定防火牆或網路時使用
setup

