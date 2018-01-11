---
layout: post
title:  "RabbitMQ day 4 (Exchange)"
date:   2018-01-10 10:44:17 +0800
categories: RabbitMQ
---

### Exchange type : 
Exchange 會根據 routing rules 將 message 送到目的．routing rules 有分 : direct (point-to-point), topic (publish-subscribe), fanout (multicast) 和 headers．


### fanout
將同一個 message 送給所有 binding 該 exchange 的 queue，fanount 模式不用給 routing Key．

![rabbitmq_day4_1.jpg]({{ "/assets/rabbitmq/rabbitmq_day4_1.jpg" | absolute_url }})

Sample Code :  
producer : 

```
package com.sample.rabt;

import java.io.IOException;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

public class FanoutProducer {

	public static void main(String[] args) {
		ConnectionFactory factory = new ConnectionFactory();
		factory.setUsername("rbt-dev");
		factory.setPassword("123456");
		factory.setVirtualHost("rbt-dev-vhost");
		factory.setHost("192.168.56.102");
		factory.setPort(5672);
		try {
			Connection connection = factory.newConnection();
			Channel channel = connection.createChannel();

			String exchangeName = "rbt-fanout-inboxes";
			channel.exchangeDeclare(exchangeName, "fanout");
			String message = "Hello World fanout ~~~";
			channel.basicPublish(exchangeName, "", null, message.getBytes());
			System.out.println(" [x] Sent '" + message + "'");

		} catch (IOException e) {
			e.printStackTrace();
		}

	}
}
```
consumer : 

```
package com.sample.rabt;

import java.io.IOException;

import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Consumer;
import com.rabbitmq.client.DefaultConsumer;
import com.rabbitmq.client.Envelope;

public class FanoutConsumer {

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
			String exchangeName = "rbt-fanout-inboxes";
			
			channel.exchangeDeclare(exchangeName, "fanout");
		    String queueName = channel.queueDeclare().getQueue();
		    System.out.println("queueName : " + queueName);
		    channel.queueBind(queueName, exchangeName, "");

		    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

		    Consumer consumer = new DefaultConsumer(channel) {
		      @Override
		      public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
		        String message = new String(body, "UTF-8");
		        System.out.println(" [x] Received '" + message + "'");
		      }
		    };
		    channel.basicConsume(queueName, true, consumer);
		} catch (IOException e) {
			e.printStackTrace();
		}
	}
}
```
先執行 Consumer 後(這邊執行三次)綁定 exchange 後等待 producer 發送訊息，執行 producer 後會看到三個 cunsumer 會看到接受到的訊息．

![rabbitmq_day4_4.jpg]({{ "/assets/rabbitmq/rabbitmq_day4_4.jpg" | absolute_url }})


### direct

根據 binding 的 routing Key 去找到對應到要送 message 的 queue，如果找不到對應的 queue 則 message 會被丟棄．也可以多個 queue 定義同樣的 rouring key 這樣 direct 的模式就會像 fanout 一樣，每個 queue 都會發送 message．

![rabbitmq_day4_2.jpg]({{ "/assets/rabbitmq/rabbitmq_day4_2.jpg" | absolute_url }})

Sample Code :  
producer : 

```
package com.sample.rabt;

import java.io.IOException;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

public class DirectProducer {

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
			String exchangeName = "rbt-direct-inboxes";
			
			channel.exchangeDeclare(exchangeName, "direct");

	        String routingKey = "black";
	        String message = "Hello World Direct ~~~";

	        channel.basicPublish(exchangeName, routingKey, null, message.getBytes());
	        System.out.println(" [x] Sent '" + routingKey + "':'" + message + "'");

	        connection.close();
		} catch (IOException e) {
			e.printStackTrace();
		}
	}
}
```
consumer : 

```
package com.sample.rabt;

import java.io.IOException;

import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Consumer;
import com.rabbitmq.client.DefaultConsumer;
import com.rabbitmq.client.Envelope;

public class DirectConsumer {

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
			String exchangeName = "rbt-direct-inboxes";
			
	        channel.exchangeDeclare(exchangeName, "direct");
		    String queueName1 = channel.queueDeclare().getQueue();
		    String queueName2 = channel.queueDeclare().getQueue();

		    channel.queueBind(queueName1, exchangeName, "black");
		    channel.queueBind(queueName2, exchangeName, "black");
		    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

		    Consumer consumer1 = new DefaultConsumer(channel) {
		      @Override
		      public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
		        String message = new String(body, "UTF-8");
		        System.out.println(" [x] Received 1 '" + envelope.getRoutingKey() + "':'" + message + "'");
		      }
		    };
		    Consumer consumer2 = new DefaultConsumer(channel) {
			      @Override
			      public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
			        String message = new String(body, "UTF-8");
			        System.out.println(" [x] Received 2 '" + envelope.getRoutingKey() + "':'" + message + "'");
			      }
			    };
		    channel.basicConsume(queueName1, true, consumer1);
		    channel.basicConsume(queueName2, true, consumer2);
		} catch (IOException e) {
			e.printStackTrace();
		}		
	}

}

```
先執行 consumer 完後，會在管理畫面看到兩個 queue，然後再執行 producer 就可以看到兩個 queue 都接收到 message．

### topic
在 queue 定義好 routing key 的 pattern，根據 exchange binding 的 routing key，進行 match pattern 後找到對應的 queue．match pattern 的符號 :  
 # : 一個或多個．  
 \* : 一個．  

![rabbitmq_day4_3.jpg]({{ "/assets/rabbitmq/rabbitmq_day4_3.jpg" | absolute_url }})

Sample Code :  
producer : 

```
package com.sample.rabt;

import java.io.IOException;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

public class TopicProducer {
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
			String exchangeName = "rbt-topic-inboxes";
			channel.exchangeDeclare(exchangeName, "topic");
	        String routingKey = "abc.criticalk";
	        String message = "Hello World Topic ~~~";
	        channel.basicPublish(exchangeName, routingKey, null, message.getBytes());
	        System.out.println(" [x] Sent '" + routingKey + "':'" + message + "'");
	        connection.close();
		} catch (IOException e) {
			e.printStackTrace();
		}
	}
}
```
consumer : 

```
package com.sample.rabt;

import java.io.IOException;

import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Consumer;
import com.rabbitmq.client.DefaultConsumer;
import com.rabbitmq.client.Envelope;

public class TopicConsumer {

	public static void main(String[] args) {
		
		//String[] argv = {"#" , "kern.*" , "*.critical"};
		String[] argv = {"abc.*" , "*.def"};
		
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
			String exchangeName = "rbt-topic-inboxes";
			channel.exchangeDeclare(exchangeName, "topic");
		    String queueName = channel.queueDeclare().getQueue();

		    if (argv.length < 1) {
		      System.err.println("Usage: ReceiveLogsTopic [binding_key]...");
		      System.exit(1);
		    }
		    for (String bindingKey : argv) {
		      channel.queueBind(queueName, exchangeName, bindingKey);
		    }
		    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");
		    Consumer consumer = new DefaultConsumer(channel) {
		      @Override
		      public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
		        String message = new String(body, "UTF-8");
		        System.out.println(" [x] Received '" + envelope.getRoutingKey() + "':'" + message + "'");
		      }
		    };
		    channel.basicConsume(queueName, true, consumer);
		} catch (IOException e) {
			e.printStackTrace();
		}
	}
}
```

這邊的範例是定義好一個 queue 然後設定 pattern "abc.*" , "#.def"，只要符合 abc.XXX 或 XXX.def、XXX.OOO.def 的 routing key 都可送 message 到這個 queue 裡．

### headers
類似 direct exchange，但是不是使用 routing Key match，而是使用 headers（message attributes) match 到指定的 queue


### 總結
- - -
可根據 exchange 的不同型態來達到我們如何傳送 message 到對應的 consumer．

