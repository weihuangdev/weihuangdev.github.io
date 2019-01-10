---
layout: post
title:  "crawler day 1 (TripAdvisor Hotel)"
date:   2019-01-10 10:44:17 +0800
categories: crawler
---

### crawler TripAdvisor
使用 [selenium](https://www.seleniumhq.org/) 來爬 [TripAdvisor Hotel](https://www.tripadvisor.com/?fid=2b2a6035-9fe8-47e0-b8a3-305aee70629c) 的資訊


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

#### AdvHotelCrawler.scala

```
package scalastarter

import java.nio.charset.StandardCharsets
import java.nio.file.{Files, Paths, StandardOpenOption}
import java.util
import java.util.concurrent.TimeUnit

import org.openqa.selenium.By
import org.openqa.selenium.chrome.{ChromeDriver, ChromeOptions}

object AdvHotelCrawler {

  def main(args: Array[String]): Unit = {
    val outpuFile = "/Users/daniel/1-project/2-ght/crawler/crawler/output/hotel.csv"
    for (i <- 0 until 6) {
      //https://www.tripadvisor.com/Hotels-g293935-oa0-Bangladesh-Hotels.html
      val str = scala.io.Source.fromURL("https://www.tripadvisor.com/Hotels-g293935-oa" +(i*30).toString + "-Bangladesh-Hotels.html").mkString.split("data-url=\"/")

      for (j <- 2 to 31) {
        //hotelUrl : https://www.tripadvisor.com/Hotel_Review-g667997-d12288294-Reviews-Hotel_Noorjahan_Grand-Sylhet_City_Sylhet_Division.html
        val hotelUrl = "https://www.tripadvisor.com/" + str(j).split('"')(0)
        //str(j) : <DIV class="prw_rup prw_meta_hsx_responsive_listing ui_section listItem" data-prwidget-name="meta_hsx_responsive_listing" data-prwidget-init="handlers" data-no-pt-mw="true" data-mlv="true"><div class="listing collapsed"data-show-recently-viewed="true"><div class="meta_listing ui_columns  large_thumbnail_mobile "onclick="widgetEvCall('handlers.listingClick', event, this);return false;"data-locationId="7736740"
        //hotelName : Hotel Noorjahan Grand
        val hotelName = str(j).split("Reviews-")(1).split('-')(0).replace('_', ' ')
        val hotelUrlContext = scala.io.Source.fromURL(hotelUrl).mkString
        val hotelInfo = hotelCrawler(hotelUrl)
        println(hotelInfo)
        appendFile(outpuFile,hotelInfo)
      }
    }

    def hotelCrawler(hotelUrl:String):String = {
      System.setProperty("webdriver.chrome.driver", "/Users/daniel/1-project/2-ght/crawler/crawler/ref/chromedriver")
      val options = new ChromeOptions()
      val prefs = new util.HashMap[String, String]()
      prefs.put("intl.accept_languages", "en-US")
      options.setExperimentalOption("prefs", prefs)
      options.addArguments("--headless")//不彈跳視窗
      val driver = new ChromeDriver(options)
      driver.manage.timeouts.implicitlyWait(3, TimeUnit.SECONDS)
      driver.get(hotelUrl)
      driver.findElement(By.cssSelector("h1.header")).click()
      //println("test1...")
      //點選金額單位
      driver.findElement(By.cssSelector("#taplc_global_footer_0 > div > div > div > div.ui_column.top_on_mobile.is-3-tablet.is-12-mobile > div > div > div:nth-child(2)")).click()
      //println("test2...")
      //點選美金
      driver.findElement(By.cssSelector("#BODY_BLOCK_JQUERY_REFLOW > div.ui_overlay.ui_flyout.null.prw_rup.prw_homepage_footer_pickers > div.body_text > div > div > ul > li:nth-child(2)")).click()
      //println("test3...")
      val hotelInfo = parseHotelInfo(driver)
      driver.quit()
      hotelInfo
    }

    def initJquery(driver:ChromeDriver): Unit = {
      //https://code.jquery.com/jquery-3.3.1.min.js
      val jQueryUrl = "https://code.jquery.com/jquery-3.3.1.min.js"
      val jQueryText = scala.io.Source.fromURL(jQueryUrl).mkString
      //println(jQueryText)
      driver.executeScript(jQueryText)
    }

    /*
    * 1. 使用 val roomTypes = driver.findElementByCssSelector("#HEADING").getText 抓不到 ROOM TYPES (原因不明)
    *
    * 2. 使用 val roomTypes = driver.executeScript("return document.
    * querySelector(\"#ABOUT_TAB > div.ui_columns.morefade.is-multiline >
    * div.ui_column.is-3.is-shown-at-desktop > div.section_content >
    * div:nth-child(5) > div\").textContent") 抓得到 ROOM TYPES，但 nth-child(5) 數字都會不一樣很難判斷
    *
    * 3. 最後使用 jquery 的 contains 寫法
    * val roomTypes = driver.executeScript("return $(\"div.sub_title:contains('Room types'):first\").next().text()")
    * */
    def parseHotelInfo(driver:ChromeDriver):String = {
      initJquery(driver)
      val hotelName = driver.findElementByCssSelector("#HEADING").getText
      val roomTypes = driver.executeScript("return $(\"div.sub_title:contains('Room types'):first\").next().text()")
      val priceRange = driver.executeScript("return $(\"div.sub_title:contains('Price range'):first\").next().text()")
      val overRating = driver.findElementByCssSelector("#OVERVIEW > div.ui_columns.is-multiline > div.ui_column.is-4-desktop.is-6-tablet.is-shown-at-tablet > div.hotels-hotel-review-overview-Reviews__rating--1iV2k > span.overallRating").getText

      //println("hotelName : " + hotelName + " , roomTypes : " + roomTypes + " , priceRange : " + priceRange.toString.split("[(]")(0).replaceAll("\\s","") + " , overRating : " + overRating)
      return hotelName + ";" + roomTypes + ";" + priceRange.toString.split("[(]")(0).replaceAll("\\s","") + ";" + overRating + "\n"
    }

    def appendFile(outputFile:String , hotelInfo:String): Unit = {
      Files.write(Paths.get(outputFile), hotelInfo.getBytes(StandardCharsets.UTF_8), StandardOpenOption.APPEND)
    }
  }
}

```

執行結果 :  

```
Hotel 71;Suites, Non-Smoking Rooms, Smoking rooms available, Family Rooms;$38-$65;4.0 
Hotel Noorjahan Grand;Non-Smoking Rooms, Suites, Family Rooms, Smoking rooms available;$38-$75;5.0 
Le Meridien Dhaka;Suites, Non-Smoking Rooms, Family Rooms, Smoking rooms available, Accessible rooms;$234-$452;4.5 
Amari Dhaka;Suites, Kitchenette, Non-Smoking Rooms, Smoking rooms available;$153-$261;4.5 
...
```



