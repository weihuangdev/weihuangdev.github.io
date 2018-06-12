---
layout: post
title:  "Spark-Streaming day 4 (spark streaming Custom Receiver)"
date:   2018-06-08 08:44:17 +0800
categories: Spark-Streaming
---
#### 目標
客制 spark-streaming 的 Receiver 到 FTP 讀檔案

#### CustomReceiver.scala
```scala
package com.streaming.receiver

import java.io.{BufferedReader, File, FileInputStream, InputStreamReader}
import java.nio.charset.StandardCharsets
import com.streaming.test.FtpFileTest
import com.streamingjava.test.FTPDirTest
import org.apache.spark.storage.StorageLevel
import org.apache.spark.streaming.receiver.Receiver

class CustomReceiver(identity:String , username:String , password:String , host:String ,sourcePath:String , targetPath:String) extends Receiver[String](StorageLevel.MEMORY_AND_DISK_2) {

  def onStart(): Unit = {
    FtpFileTest.copyDir(identity , username , password , host ,sourcePath , targetPath)
    new Thread("Custom FTP Receiver") {
      override def run() { receive() }
    }.start()
  }

  override def onStop(): Unit = {
  }

  private def receive(){
    var userInput: String = null
    try {
      var i = 0
      val localFiles = getListOfFiles(new File(targetPath))
      for(localFile <- localFiles) {
        val reader = new BufferedReader(new InputStreamReader(new FileInputStream(localFile), StandardCharsets.UTF_8))
        userInput = reader.readLine()
        while(!isStopped && userInput != null) {
          if(i > 0) {
            userInput = userInput + "_encrypt"
            store(userInput)
            userInput = reader.readLine()
          }
          i = 1
        }
        reader.close()
        FtpFileTest.copytoFTP(identity , username , password , host ,targetPath + "/" + localFile.getName , "/tmp/streaming-test/complete")
        FTPDirTest.deleteFile(username , password , host , sourcePath + "/" + localFile.getName )
        localFile.delete()
      }
      restart("Trying to connect again")
    } catch {
      case e: java.net.ConnectException =>
        restart("Error connecting to " + host + ":" , e)
      case t: Throwable =>
        restart("Error receiving data", t)
    }
  }

  def getListOfFiles(dir: File):List[File] = dir.listFiles.filter(_.isFile).toList
}

```
#### CustomReceiverTest.scala 
```scala
package com.streaming.test

import java.io.{BufferedWriter, File, FileWriter}

import com.streaming.receiver.CustomReceiver
import org.apache.spark.SparkConf
import org.apache.spark.streaming.{Seconds, StreamingContext}

object CustomReceiverTest {

  def main(args: Array[String]): Unit = {
    val username = "daniel"
    val password = "123456"
    val host = "192.168.61.106"
    val sftpPort = 22
    val identity = null
    val sourcePath = "/tmp/streaming-test/sourcefile"
    val targetPath = "/Users/daniel/test/streamFile"
    
    val conf = new SparkConf().setMaster("local[1]").setAppName("FileWordCount")
    val ssc = new StreamingContext(conf, Seconds(1))
    val lines = ssc.receiverStream(new CustomReceiver(identity,username,password,host,sourcePath,targetPath))

    lines.foreachRDD(myRdd => {
      if(myRdd != null) {
        val test = myRdd.map(x => x)
        test.saveAsTextFile("/Users/daniel/test/streamFile_result")
      } else {
      }

    })
    ssc.start()
    ssc.awaitTermination()
  }
}

```





