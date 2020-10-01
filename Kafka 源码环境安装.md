# Kafka 源码环境安装

## 准备

操作系统： WIN10

**gradle 5.0**

下载 https://services.gradle.org/distributions/gradle-5.0-bin.zip

解压并设置环境变量



**Scala 2.12**

https://downloads.lightbend.com/scala/2.12.12/scala-2.12.12.msi



**IDEA Scala 插件**

 IDEA内插件下载失败  https://www.jetbrains.com/help/idea/managing-plugins.html  

手动下载安装  https://plugins.jetbrains.com/plugin/1347-scala/versions

选择与IDEA版本号对于的安装包下载

放置到IDEA插件目录 C:\Program Files\JetBrains\IntelliJ IDEA 2019.3.3\plugins\scala

从硬盘安装 --> 选择文件 -->  按提示重启 --> 安装成功



**kafka**

https://github.com/apache/kafka

漫长的等待, 万恶的墙...

## QuickStart

**下载和安装gradle wrapper**

```shell
gradle
```

报错：

```shell
Starting a Gradle Daemon (subsequent builds will be faster)

FAILURE: Build failed with an exception.

* What went wrong:
Unable to start the daemon process.
This problem might be caused by incorrect configuration of the daemon.
For example, an unrecognized jvm option is used.
Please refer to the user guide chapter on the daemon at https://docs.gradle.org/5.0/userguide/gradle_daemon.html
Please read the following process output to find out more:
-----------------------
Error occurred during initialization of VM
Could not reserve enough space for 2097152KB object heap


* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output. Run with --scan to get full insights.

* Get more help at https://help.gradle.org
```

解决：在环境变量中添加   `JAVA_OPTIONS=-Xmx512M`

Reference: https://stackoverflow.com/questions/26143740/getting-gradle-error-could-not-reserve-enough-space-for-object-heap-constantly

然后又是漫长的等待...（放着下载，明天再看吧，撤了）

