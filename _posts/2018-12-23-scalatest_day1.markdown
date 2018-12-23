---
layout: post
title:  "scalatest day 1 (sbt)"
date:   2018-12-23 10:45:17 +0800
categories: scalatest
---

建立 sbt project : 

```
/testingscala
 -src/main/scala
 -src/test/scala
 build.sbt
```

build.sbt : 

```
name := "Testing Scala"
version := "1.0"
scalaVersion := "2.9.2"
libraryDependencies += "org.scalatest" %% "scalatest" % "1.8" % "test"
```

由於 org.scalatest 後面接的是 %% 所以會在 meven 和 scala 的 repositories 去找 dependency scalatest_2.9.2．
%% 會在底線(_)後面接 scalaVersion 代表要找的 library 的名稱．如果不想要 sbt 幫忙控制 scala 版本的話，可以改用 % ．
但就要自己加上版本 :  

```
libraryDependencies += "org.scalatest" % "scalatest_2.9.2" % "1.8" % "test"
```

最後的 "test" 是代表該 dependency 的 scope．可以確保所有該 dependency 的 class 檔只會 load for test．

執行 sbt，當在 sbt 的 command prompt 時，有修改 build.sbt 的話可以下 reload 跟 update 指令，更新 dependency．

```
> ~/testingscala > sbt
[info] Updated file /Users/daniel/testingscala/project/build.properties: set sbt.version to 1.1.6
[info] Loading settings from idea.sbt ...
[info] Loading global plugins from /Users/daniel/.sbt/1.0/plugins
[info] Loading project definition from /Users/daniel/testingscala/project
[info] Updating ProjectRef(uri("file:/Users/daniel/testingscala/project/"), "testingscala-build")...
[info] Done updating.
[info] Loading settings from build.sbt ...
[info] Set current project to Testing Scala (in build file:/Users/daniel/testingscala/)
[info] sbt server started at local:///Users/daniel/.sbt/1.0/server/dfd2749eebeef02db62d/sock
sbt:Testing Scala> reload
[info] Loading settings from idea.sbt ...
[info] Loading global plugins from /Users/daniel/.sbt/1.0/plugins
[info] Loading project definition from /Users/daniel/testingscala/project
[info] Loading settings from build.sbt ...
[info] Set current project to Testing Scala (in build file:/Users/daniel/testingscala/)
sbt:Testing Scala> update
[info] Updating ...
[info] downloading https://repo1.maven.org/maven2/org/scala-lang/scala-library/2.9.2/scala-library-2.9.2.jar ...
[info] downloading https://repo1.maven.org/maven2/org/scala-lang/scala-compiler/2.9.2/scala-compiler-2.9.2.jar ...
[info] downloading https://repo1.maven.org/maven2/org/scala-lang/jline/2.9.2/jline-2.9.2.jar ...
[info] downloading https://repo1.maven.org/maven2/org/scalatest/scalatest_2.9.2/1.8/scalatest_2.9.2-1.8.jar ...
[info]  [SUCCESSFUL ] org.scala-lang#jline;2.9.2!jline.jar (2741ms)
[info]  [SUCCESSFUL ] org.scalatest#scalatest_2.9.2;1.8!scalatest_2.9.2.jar (5157ms)
[info]  [SUCCESSFUL ] org.scala-lang#scala-library;2.9.2!scala-library.jar (10999ms)
[info]  [SUCCESSFUL ] org.scala-lang#scala-compiler;2.9.2!scala-compiler.jar (15499ms)
[info] Done updating.
[success] Total time: 21 s, completed 2018/12/23 下午 12:11:59
```

如果想要的 library 不在預設的 maven repositories，可以新增 repositories 可以在 resolvers 增加(name at location) :  

```
resolvers += "Codehaus stable repository" at "http://repository.codehaus.org/"
```

如果只想測試某些 package，或指定的 class 可以使用 testOnly :  

```
~/testingscala > sbt "testOnly com.simple.*"
[info] Loading settings from idea.sbt ...
[info] Loading global plugins from /Users/daniel/.sbt/1.0/plugins
[info] Loading project definition from /Users/daniel/testingscala/project
[info] Loading settings from build.sbt ...
[info] Set current project to Testing Scala (in build file:/Users/daniel/testingscala/)
Hello
[info] ShowMessageTest:
[info] - showHelloTest
[info] Run completed in 323 milliseconds.
[info] Total number of tests run: 1
[info] Suites: completed 1, aborted 0
[info] Tests: succeeded 1, failed 0, canceled 0, ignored 0, pending 0
[info] All tests passed.
[success] Total time: 2 s, completed 2018/12/23 下午 03:20:50
```














