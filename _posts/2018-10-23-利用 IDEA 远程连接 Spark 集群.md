---
layout:     post
title:      IDEA 远程连接 Linux 下 Spark 集群
subtitle:   远程连接
date:       2018-10-21
author:     GG
header-img: img/post-bg-1.jpg
catalog: true
tags:
    - Docker
    - 换源
    - 大数据生态集群
---



>
>  本文为实现 在Windows 下利用IDEA 连接 Linux 下Spark 集群
>

## 1.1 IDEA新建一个ConnectSparkTest项目 

引入pom文件形式，注意集群版本这里的配置版本相同 

```xml
  <properties>
        <hadoop.version>2.7.4</hadoop.version>
        <scala.version>2.11.11</scala.version>
        <spark.version>2.2.0</spark.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.scala-lang</groupId>
            <artifactId>scala-library</artifactId>
            <version>${scala.version}</version>
        </dependency>

        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_2.11</artifactId>
            <version>${spark.version}</version>
        </dependency>

        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-client</artifactId>
            <version>${hadoop.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-sql_2.11</artifactId>
            <version>${spark.version}</version>
        </dependency>

    </dependencies>
```



## 1.2 配置windows下hosts  

1、 在 C:\Windows\System32\drivers\etc 有个hosts文件配置spark 集群节点。

2、 测试：win+R : ping 通Spark Master 的ip 即可。

## 1.3 IDEA 打jar包  

程序中将jar包位置放进去： 

```scala
val spark = SparkSession.builder().master("spark://ip:7077").
      appName("connect_spark").
      config("spark.jars", "E:\\IDEAworkspace\\connectSparkTest\\out\\artifacts\\connectSparkTest_jar\\connectSparkTest.jar").
      config("spark.cores.max", "10").
      config("spark.executor.memory", "2g").
      config("spark.driver.memory", "1g").
      getOrCreate()
```



## 1.4 配置windows 下的Hadoop 

将linux 下hadoop 文件整体拷贝到Windows 下。

配置 HADOOP_HOME , 并将其添加进 PATH 中。

同时下载 hadoop 对应版本的 winutils.exe 到 HADOOP_HOME\bin 中。

**PS:不进行以上操作**:执行程序时会报错： null\bin\winutils.exe



## 1.5 报错内容

```
WARN TaskSchedulerImpl:Initial job has not accepted any resources; check your cluster UI to register.
```

在Spark UI 的 某个executor 的 State 变为 Exit 时， 查看 stderr ，会发现报错 :

```
Caused by: java.util.concurrent.TimeoutException: Cannot receive any reply from 192.168.159.1:7710 in 120 seconds
```

通过win + r 下 ipconfig 查看 ip ,发现该 ip 为 VMnet8 和VMnet1，是安装的虚拟机的ip ,解决方法就是 禁用 网络即可。 



## 1.6 连接HDFS 

使用此方式：

```scala
val b = spark.sparkContext.textFile("hdfs://d2e469333248:8020/user/lg/a.txt").map(_.split(" "))
```



