---
layout: post
title:  "Spark-Streaming day 3 (spark read ftp file)"
date:   2018-06-08 08:44:17 +0800
categories: Spark-Streaming
---
#### 目標
用 vsftpd 建立 ftp server ，並使用 spark-sftp_2.11 套件讀取 ftp file  

#### 安裝 vsftpd(linux 可用來當作 ftp server)
```
yum install vsftpd
```
* 安裝好設定檔位置 :  
```
/etc/vsftpd/vsftpd.conf
```
* 啟動 vsftpd :  
```
service vsftpd start
```
* 預設匿名者(anonymous)登入的根目錄 :  
```
/var/ftp/
```
* 設定下列參數值(vsftpd.conf) :  
```
write_enable=YES
anon_mkdir_write_enable=YES
anon_upload_enable=YES
chroot_local_user=YES
```
#### 測試 anonymous 登入  
![spark-streaming_day3_1.jpg]({{ "/assets/spark-streaming/day3/spark-streaming_day3_1.jpg" | absolute_url }})

#### 建立新的帳號登入
![spark-streaming_day3_2.jpg]({{ "/assets/spark-streaming/day3/spark-streaming_day3_2.jpg" | absolute_url }})

#### 設定下列參數值(vsftpd.conf) :  
chroot_local_user=YES
chroot_list_enable=YES
chroot_list_file=/etc/vsftpd/chroot_list

#### 新增 chroot_list 檔案
```
touch /etc/vsftpd/chroot_list
```
#### 在 chroot_list 加上 user name
![spark-streaming_day3_3.jpg]({{ "/assets/spark-streaming/day3/spark-streaming_day3_3.jpg" | absolute_url }})
#### 測試用建立的帳號 allen 登入，並上傳測試檔案  
![spark-streaming_day3_4.jpg]({{ "/assets/spark-streaming/day3/spark-streaming_day3_4.jpg" | absolute_url }})
![spark-streaming_day3_5.jpg]({{ "/assets/spark-streaming/day3/spark-streaming_day3_5.jpg" | absolute_url }})

#### 修改 build.sbt

```
name := "streaming-test"
version := "0.1"
scalaVersion := "2.11.0"

libraryDependencies += "org.apache.spark" %% "spark-streaming" % "2.2.0"
libraryDependencies += "org.apache.spark" %% "spark-sql" % "2.2.0"
libraryDependencies += "com.springml" % "spark-sftp_2.11" % "1.1.1"
```

#### 寫測試程式 : 
```
package com.streaming.test
import org.apache.spark.sql.SparkSession

object FtpStreamingTest {

  def main(args: Array[String]): Unit = {
    val sparkSession = SparkSession.builder.master("local").
      appName("spark session example").getOrCreate()

    val df = sparkSession.read.format("com.springml.spark.sftp").
      option("username", "allen").
      option("password", "123456").
      option("host", "192.168.61.106").
      option("fileType", "json").
      load("/home/allen/testfile/file_1.txt")

    df.collect.foreach(println)
  }
}
```
#### 測試結果  
![spark-streaming_day3_6.jpg]({{ "/assets/spark-streaming/day3/spark-streaming_day3_6.jpg" | absolute_url }})


> 參考網址  
> [spark-sftp](https://github.com/springml/spark-sftp)  
> [vsftpd-1](http://linux.vbird.org/linux_server/0410vsftpd/0410vsftpd.php)  
> [vsftpd-2](http://www.wkb.idv.tw/study/centos/c07.htm)  






