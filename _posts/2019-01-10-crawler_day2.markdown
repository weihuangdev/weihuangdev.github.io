---
layout: post
title:  "crawler day 2 (google search rank)"
date:   2019-01-10 10:44:17 +0800
categories: crawler
---

### crawler google search rank
使用 [selenium](https://www.seleniumhq.org/) 來爬查詢關鍵字的前幾名網站

#### build.sbt

```
enablePlugins(ScalaJSPlugin)

name := "scala-starter"
scalaVersion := "2.12.3"

libraryDependencies += "org.jsoup" % "jsoup" % "1.11.2"

libraryDependencies ++= Seq(
  "org.scalatest" %% "scalatest" % "3.0.1" % "test",
  "org.seleniumhq.selenium" % "selenium-java" % "3.10.0",
  "org.scala-js" %%% "scalajs-dom" % "0.9.1",
  "be.doeraene" %%% "scalajs-jquery" % "0.9.1"
)
scalaJSUseMainModuleInitializer := true
scalaSource in Test := baseDirectory.value / "test"

```

#### /crawler/project/plugin.sbt
```
addSbtPlugin("org.scala-js" % "sbt-scalajs" % "0.6.22")
```

#### GoogleKeyWordsCrawler.scala

```
package scalastarter

import java.nio.charset.StandardCharsets
import java.nio.file.{Files, Paths, StandardOpenOption}
import java.util.concurrent.TimeUnit

import org.openqa.selenium.By
import org.openqa.selenium.chrome.{ChromeDriver, ChromeOptions}

import scala.util.Random

object GoogleKeyWordsCrawler {

  def main(args: Array[String]): Unit = {
    val r = new Random
    val crawlerKeywords = "/Users/daniel/1-project/2-ght/crawler/crawler/ref/crawlerKeywords.csv"
    for (line <- scala.io.Source.fromFile(crawlerKeywords).getLines) {
      crawler(line)
      Thread.sleep(40000 + r.nextInt(20000)+1)
    }
  }

  def crawler(line: String): Unit = {
    val outputFile = "/Users/daniel/1-project/2-ght/crawler/crawler/output/webInfo.csv"
    System.setProperty("webdriver.chrome.driver", "/Users/daniel/1-project/2-ght/crawler/crawler/ref/chromedriver")
    val r = new Random
    val options:ChromeOptions = new ChromeOptions()
    // options.addArguments("--headless")// 可以關掉視窗
    val driver = new ChromeDriver(options)
    val lineSplit = line.split(",")
    val no = lineSplit(0)
    val keywords = lineSplit(1)
    var adForSearch = keywords.replaceAll("\\\\|/|:|\\*|\\?|<|>|\\||&|'\"", "+")
    adForSearch = keywords.replaceAll(" ", "+")
    try {
      var url = "NA"
      driver.manage.timeouts.implicitlyWait(r.nextInt(3), TimeUnit.SECONDS)
      //gl等於geolocation,num前100筆
      driver.get("https://www.google.com/search?q=" + adForSearch +"&gl=tw&num=100")
      val results = driver.findElements(By.cssSelector("div.rc > div.r > a"))
      val l = results.toArray.length
      for (i <- 0 until l) {
        url = results.get(i).getAttribute("href")
        val fileInfo = no + ";" + keywords + ";" + url + "\n"
        appendFile(outputFile,fileInfo)
      }
      driver.quit()
    } catch {
      case e: Exception =>
        e.printStackTrace()
        driver.quit()
    }
  }

  def appendFile(outputFile:String , fileInfo:String): Unit = {
    Files.write(Paths.get(outputFile), fileInfo.getBytes(StandardCharsets.UTF_8), StandardOpenOption.APPEND)
  }

}

```

執行結果 :  

```
1;java;https://www.java.com/zh_TW/
1;java;https://zh.wikipedia.org/zh-tw/Java
1;java;https://www.oracle.com/technetwork/java/index.html
1;java;https://translate.google.com/translate?hl=zh-TW&sl=en&u=https://www.oracle.com/technetwork/java/index.html&prev=search
1;java;https://www.oracle.com/technetwork/java/javase/downloads/index.html
1;java;https://translate.google.com/translate?hl=zh-TW&sl=en&u=https://www.oracle.com/technetwork/java/javase/downloads/index.html&prev=search
1;java;https://www.ithome.com.tw/voice/126265
1;java;https://programming.im.ncnu.edu.tw/J_index.html
1;java;http://www.codedata.com.tw/book/java-basic/index.php
...
```


