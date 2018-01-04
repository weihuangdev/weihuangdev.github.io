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


### 補充
- - -
* 查看 yum 目前 repositories
yum repolist

* 查詢 Centos 版本
cat /etc/redhat-release //查詢 Centos 版本

