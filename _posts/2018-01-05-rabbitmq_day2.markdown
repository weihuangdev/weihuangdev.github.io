---
layout: post
title:  "RabbitMQ day 2 (Message System)"
date:   2018-01-05 10:44:17 +0800
categories: RabbitMQ
---

### Message
系統之間彼此交換訊息(message)，根據可靠行及同步可分為四個區塊，這邊列出四個代表 :
* SMTP(Simple Mail Transfer Protocol)
* AMQP(Advanced Message Queuing Protocol)
* HTTP(HyperText Transfer Protocol)
* IIOP(Internet Inter-ORB Protocol)

![rabbitmq_day2_0.jpg]({{ "/assets/rabbitmq/rabbitmq_day2_0.jpg" | absolute_url }})

根據系統不一樣的需求，就會有不同交換訊息的設計．

* request-response 型態 : 系統傳 request 給另一個系統等待回傳 response，一種 point-to-point 的方式

![rabbitmq_day2_1.jpg]({{ "/assets/rabbitmq/rabbitmq_day2_1.jpg" | absolute_url }})

* 使用 Message Queue 非同步的設計，會有一個 publishers (producers) 發送 message 給 broker，broker 在將 message 傳給需要的 consumer．

![rabbitmq_day2_2.jpg]({{ "/assets/rabbitmq/rabbitmq_day2_2.jpg" | absolute_url }})

* loosely coupled 的架構，publisher 跟 consumer 彼此之間不用知道對方在哪 : 

![rabbitmq_day2_3.jpg]({{ "/assets/rabbitmq/rabbitmq_day2_3.jpg" | absolute_url }})

這樣的設計有幾項優點 :  
1. publisher 跟 consumer 的系統如果有損壞的，彼此間不影響．
2. 系統之間的效能不會彼此影響．
3. publisher 跟 consumer 彼此不知道對方的位址及技術．

### Simple Mail Transfer Protocol (SMTP).
 e-mails 會發送(published)到一個 SMTP server，然後該 server 會再傳到另一個 SMTP server，直到傳到接收者的 email server．則 message 會被 queue 等待 consumer(POP3“Post Office Protocol 3”或IMAP) 來將 message 取走．在 SMTP publisher 不會知道 e-mail 最後送去哪，只會知道已成功送給 broker(SMTP server) 並且已初始化送出．

![rabbitmq_day2_4.jpg]({{ "/assets/rabbitmq/rabbitmq_day2_4.jpg" | absolute_url }})

### AMQP
RabbitMQ 是用 Erlang 這語言來實作 AMQP．AMQP (Advanced Message Queuing Protocol)，是一種協定，它保証可以傳達(reliable)且可非同步(async)．AMQP 有一些核心的概念 :  

* Broker : 是一個中介的應用程式，接收 publisher 的 message 將它傳給 consumer 或下一個 broker
* Virtual host : 是在 broker 裡的一個虛擬區塊，用來隔離 publisher、consumer 及 AMQP 的結構，不同 Virtual host 不可共享彼此的資源(Exchange、queue)，為了安全的議題，預設是(/)．
* Connection : 指的是 TCP connection，連接於 publisher、consumer 和 broker．client disconnection 或網路、broker 掛掉時 connection 才會關閉．
* Channel : channel 會隔離特定的 client 和 broker，以至於它們不會互相影響．多個 Channel 可以成為一個 connection．
* Exchange : 會將 publisher 傳送的 message 做初始化找到對應的 routing rules 將 message 送到目的地．routing rules 包括 : direct (point-to-point), topic (publish-subscribe) 和 fanout (multicast) 
* Queue : 這是 message 準備送到 consumer 的最後階段，單一訊息可被複製到多個 queue 裡，但由 Exchange 決定．
* Binding : 是一個介於 Exchange 及 Queue 的虛擬連線， routing key 怎麼做 binding 會根據 Exchange routing rules 決定．

![rabbitmq_day2_5.jpg]({{ "/assets/rabbitmq/rabbitmq_day2_5.jpg" | absolute_url }})


### 總結
- - -
今天稍微了解了一下 Message System 的設計以及 AMQP 協定，這樣的架構感覺很彈性．


