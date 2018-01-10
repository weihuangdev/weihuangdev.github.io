---
layout: post
title:  "RabbitMQ day 3 (Implement)"
date:   2018-01-09 10:44:17 +0800
categories: RabbitMQ
---

* Establishing a solid connection to RabbitMQ
建立 Connection 程式 : 

```
package com.sample.rabt;

import java.io.IOException;

import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

public class GetConnection {
	public static void main(String[] args) {
		ConnectionFactory factory = new ConnectionFactory();
		factory.setUsername("rbt-dev");
		factory.setPassword("123456");
		factory.setVirtualHost("rbt-dev-vhost");
		factory.setHost("192.168.56.102");
		factory.setPort(5672);
		try {
			Connection connection = factory.newConnection();
			System.out.println(connection.isOpen());
		} catch (IOException e) {
			e.printStackTrace();
		}
	}
}
```
執行完上述程式後會在管理介面上看到建立的 connection 資訊 : 

![rabbitmq_day3_1.jpg]({{ "/assets/rabbitmq/rabbitmq_day3_1.jpg" | absolute_url }})

* Working with channels
用上面建立的 connection 建立 channel : 

```
Channel channel = connection.createChannel();
System.out.println(channel.isOpen());
```
* 定義 exchange

```
String type = "direct";
boolean durable = true;//server restart 之後還會在
boolean autoDelete = false;//keep it even if nobody is using it
Map<String, Object> arguments = null;
String exchangeInBox = "rbt-inboxes";
channel.exchangeDeclare(exchangeInBox, type, durable, autoDelete, arguments);
```
* 定義 queue

```
String queue = "rbt-queue";
durable = true;
autoDelete = false;
boolean exclusive = false;// this queue to be consumable by other connections
arguments = null;
channel.queueDeclare(queue, durable, exclusive, autoDelete, arguments);
```
* 將 routingKey 與 queue、exchange bind 起來

```
String routingKey = queue;
channel.queueBind(queue, exchangeInBox, routingKey);
```
* Sending user messages

```
String messageId = UUID.randomUUID().toString();//讓每個 message 有一個唯一識別值，在分散式系統可追查
BasicProperties props = new BasicProperties.Builder().contentEncoding("UTF-8").messageId(messageId).deliveryMode(2).build();

String message = "Hello World~~~";
channel.basicPublish(exchangeInBox, routingKey, props, message.getBytes("UTF-8"));
```
* deliveryMode : The AMQP speci cation de nes the value for this property as follows: for Non-persistent it is set to 1 and for Persistent it is set to 2. 
* messageID : important aspect of traceability in messaging and distributed applications.

使用 rbt-admin (admin才可登入)登入管理介面 : 

![rabbitmq_day3_2.jpg]({{ "/assets/rabbitmq/rabbitmq_day3_2.jpg" | absolute_url }})

![rabbitmq_day3_3.jpg]({{ "/assets/rabbitmq/rabbitmq_day3_3.jpg" | absolute_url }})

![rabbitmq_day3_4.jpg]({{ "/assets/rabbitmq/rabbitmq_day3_4.jpg" | absolute_url }})

* AMQP message structure  

![rabbitmq_day3_5.jpg]({{ "/assets/rabbitmq/rabbitmq_day3_5.jpg" | absolute_url }})

* 接收 queue 的訊息 

![rabbitmq_day3_6.jpg]({{ "/assets/rabbitmq/rabbitmq_day3_6.jpg" | absolute_url }})

可以看到 4 筆 message 都被 consumer 取走了 : 

![rabbitmq_day3_7.jpg]({{ "/assets/rabbitmq/rabbitmq_day3_7.jpg" | absolute_url }})


### 總結
- - -
在前一天介紹完架構後，再來看今天介紹的程式會比較有感覺．  
Sample Code :  

```
package com.sample.rabt;

import java.io.IOException;
import java.util.Map;
import java.util.UUID;

import com.rabbitmq.client.AMQP.BasicProperties;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

public class GetConnection {
	public static void main(String[] args) {
		ConnectionFactory factory = new ConnectionFactory();
		factory.setUsername("rbt-dev");
		factory.setPassword("123456");
		factory.setVirtualHost("rbt-dev-vhost");
		factory.setHost("192.168.56.102");
		factory.setPort(5672);
		try {
			Connection connection = factory.newConnection();
			System.out.println(connection.isOpen());
			Channel channel = connection.createChannel();
			System.out.println(channel.isOpen());
			
			
			String type = "direct";
			boolean durable = true;//server restart 之後還會在
			boolean autoDelete = false;//keep it even if nobody is using it
			Map<String, Object> arguments = null;
			String exchangeInBox = "rbt-inboxes";
			channel.exchangeDeclare(exchangeInBox, type, durable, autoDelete, arguments);
			
			String queue = "rbt-queue";
			durable = true;
			autoDelete = false;
			boolean exclusive = false;// this queue to be consumable by other connections
			arguments = null;
			channel.queueDeclare(queue, durable, exclusive, autoDelete, arguments);
			
			String routingKey = queue;
			channel.queueBind(queue, exchangeInBox, routingKey);
			
			String messageId = UUID.randomUUID().toString();
            BasicProperties props = new BasicProperties.Builder().contentEncoding("UTF-8").messageId(messageId).deliveryMode(2).build();
			
			String message = "Hello World~~~";
			channel.basicPublish(exchangeInBox, routingKey, props, message.getBytes("UTF-8"));
		} catch (IOException e) {
			e.printStackTrace();
		}
	}
}
```

```
package com.sample.rabt;

import java.io.IOException;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.GetResponse;

public class GetMessages {
	public static void main(String[] args) {
		ConnectionFactory factory = new ConnectionFactory();
		factory.setUsername("rbt-dev");
		factory.setPassword("123456");
		factory.setVirtualHost("rbt-dev-vhost");
		factory.setHost("192.168.56.102");
		factory.setPort(5672);
		try {
			Connection connection = factory.newConnection();
			System.out.println(connection.isOpen());
			Channel channel = connection.createChannel();
			String queue = "rbt-queue";
			GetResponse getResponse = null;
			String contentEncoding = "UTF-8";
			while((getResponse = channel.basicGet(queue, false)) != null) {
				String retMessage = new String(getResponse.getBody() , contentEncoding);
				System.out.println(retMessage);
			}
		} catch (IOException e) {
			e.printStackTrace();
		}
	}
}

```







