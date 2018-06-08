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

#### 最後附上這次測試 vsftpd.conf 的設定值

```
#### 
# Example config file /etc/vsftpd/vsftpd.conf
#
# The default compiled in settings are fairly paranoid. This sample file
# loosens things up a bit, to make the ftp daemon more usable.
# Please see vsftpd.conf.5 for all compiled in defaults.
#
# READ THIS: This example file is NOT an exhaustive list of vsftpd options.
# Please read the vsftpd.conf.5 manual page to get a full idea of vsftpd's
# capabilities.
#
# Allow anonymous FTP? (Beware - allowed by default if you comment this out).
anonymous_enable=YES
#
# Uncomment this to allow local users to log in.
# When SELinux is enforcing check for SE bool ftp_home_dir
local_enable=YES
#
# Uncomment this to enable any form of FTP write command.
write_enable=YES
#
# Default umask for local users is 077. You may wish to change this to 022,
# if your users expect that (022 is used by most other ftpd's)
local_umask=022
#
# Uncomment this to allow the anonymous FTP user to upload files. This only
# has an effect if the above global write enable is activated. Also, you will
# obviously need to create a directory writable by the FTP user.
# When SELinux is enforcing check for SE bool allow_ftpd_anon_write, allow_ftpd_full_access
anon_upload_enable=YES
#
# Uncomment this if you want the anonymous FTP user to be able to create
# new directories.
anon_mkdir_write_enable=YES
#
# Activate directory messages - messages given to remote users when they
# go into a certain directory.
dirmessage_enable=YES
#
# Activate logging of uploads/downloads.
xferlog_enable=YES
#
# Make sure PORT transfer connections originate from port 20 (ftp-data).
connect_from_port_20=YES
#
# If you want, you can arrange for uploaded anonymous files to be owned by
# a different user. Note! Using "root" for uploaded files is not
# recommended!
#chown_uploads=YES
#chown_username=whoever
#
# You may override where the log file goes if you like. The default is shown
# below.
#xferlog_file=/var/log/xferlog
#
# If you want, you can have your log file in standard ftpd xferlog format.
# Note that the default log file location is /var/log/xferlog in this case.
xferlog_std_format=YES
#
# You may change the default value for timing out an idle session.
#idle_session_timeout=600
#
# You may change the default value for timing out a data connection.
#data_connection_timeout=120
#
# It is recommended that you define on your system a unique user which the
# ftp server can use as a totally isolated and unprivileged user.
#nopriv_user=ftpsecure
#
# Enable this and the server will recognise asynchronous ABOR requests. Not
# recommended for security (the code is non-trivial). Not enabling it,
# however, may confuse older FTP clients.
#async_abor_enable=YES
#
# By default the server will pretend to allow ASCII mode but in fact ignore
# the request. Turn on the below options to have the server actually do ASCII
# mangling on files when in ASCII mode.
# Beware that on some FTP servers, ASCII support allows a denial of service
# attack (DoS) via the command "SIZE /big/file" in ASCII mode. vsftpd
# predicted this attack and has always been safe, reporting the size of the
# raw file.
# ASCII mangling is a horrible feature of the protocol.
#ascii_upload_enable=YES
#ascii_download_enable=YES
#
# You may fully customise the login banner string:
#ftpd_banner=Welcome to blah FTP service.
#
# You may specify a file of disallowed anonymous e-mail addresses. Apparently
# useful for combatting certain DoS attacks.
#deny_email_enable=YES
# (default follows)
#banned_email_file=/etc/vsftpd/banned_emails
#
# You may specify an explicit list of local users to chroot() to their home
# directory. If chroot_local_user is YES, then this list becomes a list of
# users to NOT chroot().
# (Warning! chroot'ing can be very dangerous. If using chroot, make sure that
# the user does not have write access to the top level directory within the
# chroot)
chroot_local_user=YES
chroot_list_enable=YES
# (default follows)
chroot_list_file=/etc/vsftpd/chroot_list
#
# You may activate the "-R" option to the builtin ls. This is disabled by
# default to avoid remote users being able to cause excessive I/O on large
# sites. However, some broken FTP clients such as "ncftp" and "mirror" assume
# the presence of the "-R" option, so there is a strong case for enabling it.
#ls_recurse_enable=YES
#
# When "listen" directive is enabled, vsftpd runs in standalone mode and
# listens on IPv4 sockets. This directive cannot be used in conjunction
# with the listen_ipv6 directive.
listen=NO
#
# This directive enables listening on IPv6 sockets. By default, listening
# on the IPv6 "any" address (::) will accept connections from both IPv6
# and IPv4 clients. It is not necessary to listen on *both* IPv4 and IPv6
# sockets. If you want that (perhaps because you want to listen on specific
# addresses) then you must run two copies of vsftpd with two configuration
# files.
# Make sure, that one of the listen options is commented !!
listen_ipv6=YES

pam_service_name=vsftpd
userlist_enable=YES
tcp_wrappers=YES
no_anon_password=YES
```






