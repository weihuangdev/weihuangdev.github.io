---
layout: post
title:  "Spark-Streaming day 4 (spark streaming Custom Receiver)"
date:   2018-06-08 08:44:17 +0800
categories: Spark-Streaming
---
#### 目標
客制 spark-streaming 的 Receiver 到 FTP 讀檔案．

#### 流程圖
![spark-streaming_day4_1.jpg]({{ "/assets/spark-streaming/day4/spark-streaming_day4_1.jpg" | absolute_url }})
1. 從 FTP copy files 到 local 目錄
2. 讀取目錄的檔案轉成 RDD 並對資料做一些處理(在字串加上_encrypt)
3. 把檔案 copy 回 FTP
4. 刪除原來 FTP 的檔案及 local 的檔案

#### 客製一個 Receiver(CustomReceiver.scala)
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
* 使用 StreamingContext 的 receiverStream 將客製的 CustomReceiver 放進去  
* ssc.receiverStream(new CustomReceiver(identity,username,password,host,sourcePath,targetPath))  

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
    
    val conf = new SparkConf().setMaster("local[2]").setAppName("FileWordCount")
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
#### FTPDirTest.scala
由於 spark-sftp_2.11 沒有辦法針對整個目錄做 copy，而底層其實是用 java 的 jsch．有嘗試過改用 scala 實作但會遇到一個問題是 scala 要去引用 java 的內部類別 ChannelSftp.LsEntry 這個物件時，引用不到，所以只好還是用 java 寫好 api 給 scala 的方法呼叫 :  
```scala
package com.streamingjava.test;

import com.jcraft.jsch.*;
import com.springml.sftp.client.ProgressMonitor;
import java.io.File;
import java.util.ArrayList;
import java.util.List;
import java.util.Vector;

public class FTPDirTest {

    public static void main(String[] args) {
        String username = "daniel";
        String password = "123456";
        String pemFileLocation = "/Users/daniel/test/testFolder/";
        String pemPassphrase = "";
        String host = "192.168.61.106";
        int sftpPort = 22;
        String identity = null;
        String target = "/tmp/streaming-test";
        List<String> files = getDirFileList(username,password,host,target);
        for(String file : files) {
            System.out.println(file);
        }
    }


    public static List<String> getDirFileList(String username, String password, String host, String targetDir) {
        List<String> fileList = new ArrayList<String>();
        try {
            ChannelSftp sftpChannel = createSFTPChannel("" , username , password , host);
            sftpChannel.cd(targetDir);

            Vector<ChannelSftp.LsEntry> childFiles = sftpChannel.ls("*.txt");
            for (ChannelSftp.LsEntry lsEntry : childFiles) {
                String entryName = lsEntry.getFilename();
                fileList.add(entryName);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return fileList;
    }

    public static void deleteFile(String username, String password, String host, String deleteFile) {
        try {
            ChannelSftp sftpChannel = createSFTPChannel("" , username , password , host);
            sftpChannel.rm(deleteFile);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private static void copyDir(ChannelSftp sftpChannel, String source, String target) throws Exception {
        System.out.println("Copying files from " + source + " to " + target);
        sftpChannel.cd(source);
        sftpChannel.lcd(target);

        Vector<ChannelSftp.LsEntry> childFiles = sftpChannel.ls(".");
        for (ChannelSftp.LsEntry lsEntry : childFiles) {
            String entryName = lsEntry.getFilename();
            System.out.println("File Entry " + entryName);
        }
    }

    private static ChannelSftp createSFTPChannel(String identity,String username,String password,String host) throws Exception {
        JSch jsch = new JSch();
        boolean useIdentity = identity != null && !identity.isEmpty();
        if (useIdentity) {
            jsch.addIdentity(identity);
        }

        Session session = jsch.getSession(username, host, 22);
        session.setConfig("StrictHostKeyChecking", "no");
        if (!useIdentity) {
            session.setPassword(password);
        }
        session.connect();
        Channel channel = session.openChannel("sftp");
        channel.connect();

        return (ChannelSftp) channel;
    }
}

```
#### FtpFileTest.scala

```scala
package com.streaming.test
import com.springml.sftp.client.SFTPClient
import com.streamingjava.test.FTPDirTest
import scala.collection.JavaConverters._

object FtpFileTest {

  def main(args: Array[String]): Unit = {
    val username = "daniel"
    val password = "123456"
    val host = "192.168.61.106"
    val sftpPort = 22
    val identity = null

    //val sourcePath = "/tmp/streaming-test/testFile.txt"
    //val targetPath = "/Users/daniel/test/testFolder/"
    //copyFile(identity , username, password, host)

    val sourcePath = "/tmp/streaming-test"
    val targetPath = "/Users/daniel/test/testFolder"
    copyDir(identity , username, password, host , sourcePath , targetPath)

  }

  def copyFile(identity:String , username:String , password:String , host:String , sourcePath:String , targetPath:String) : Unit = {
    val sftpClient = new SFTPClient(identity , username, password, host)
    val target = sftpClient.copy(sourcePath,targetPath)
  }

  def copytoFTP(identity:String , username:String , password:String , host:String , sourcePath:String , targetPath:String) = {
    val sftpClient = new SFTPClient(identity , username, password, host)
    val target = sftpClient.copyToFTP(sourcePath , targetPath)
  }

  def copyDir(identity:String , username:String , password:String , host:String , source:String, target:String) : Unit = {
    val jfiles:java.util.List[String] = FTPDirTest.getDirFileList(username , password , host , source)
    //import scala.collection.JavaConverters
    //val sfiles = JavaConverters.asScalaIteratorConverte(jfiles.iterator()).asScala.seq
    //import scala.collection.JavaConverters._
    val sfiles = jfiles.iterator.asScala
    for(sfileName <- sfiles) {
      val sourceFile = source + "/" + sfileName
      copyFile(identity , username , password , host , sourceFile , target)
    }
  }

  def moveFile(): Unit = {

  }

}
```
* java 的 list 轉到 scala 的 list :  
使用 java 的 class 取得檔案清單
```scala
val jfiles:java.util.List[String] = FTPDirTest.getDirFileList(username , password , host , source)
```
* 透過 JavaConverters 轉成 scala 的 seq  
```scala
import scala.collection.JavaConverters
val sfiles = JavaConverters.asScalaIteratorConverte(jfiles.iterator()).asScala.seq
```
* 簡化版
```scala
import scala.collection.JavaConverters._
val sfiles = jfiles.iterator.asScala
```


hdfs dfs -setrep -R 1 /
> 參考資料  
> [sample-1](https://mapr.com/blog/how-integrate-custom-data-sources-apache-spark/)




