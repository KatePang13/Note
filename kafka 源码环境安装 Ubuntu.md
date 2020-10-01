## kafka 源码环境安装

操作系统： Ubuntu



**java**

```shell
pang@ubuntu:~$ sudo apt install openjdk-14-jre-headless
pang@ubuntu:~$ java -version
openjdk version "14.0.2" 2020-07-14
```

**scala**

```shell
sudo apt install scala
pang@ubuntu:~$ scala -version
Scala code runner version 2.11.12 -- Copyright 2002-2017, LAMP/EPFL
```

**gradle**

Reference :  https://gradle.org/install/

```shell
#
sudo unzip -d /opt/gradle /mnt/hgfs/GitHub/gradle-6.0-bin.zip
# 解压后添加 ../bin/ 到PATH
```

```shell
# 到kafka工程目录下  gradle 初始化编译环境
pang@ubuntu:~$ gradle
# 报错
FAILURE: Build failed with an exception.

* What went wrong:
Could not initialize class org.codehaus.groovy.reflection.ReflectionCache
# 解决: 安装 gradle 6.6
```

```shell
# gradlew 编译
pang@ubuntu:~$ ./gradlew jar
```



