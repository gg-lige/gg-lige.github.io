---
layout:     post
title:      SparkStreaming + Kafka 程序示例
subtitle:   SparkStreaming  Kafka应用程序 
date:       2018-05-29
author:     GG
header-img: img/post-bg-1.jpg
catalog: true
tags:
    - Spark Streaming 
    - Kafka 
    - Zookeeper
---



>  
>  本文是 Spark Streaming 学习过程中的总结。
>  同时包含 与 Kafka 连接的环境搭建
> 

#   使用自带Zookeeper搭建kafka集群
1. 下载
   - kafka_2.11-1.1.0.tgz软件下载并解压 tar -zxvf  kafka_2.11-1.1.0.tgz
2. 配置Zookeeper

   - /opt/lg/packages/kafka_2.11-1.1.0/config下的 zookeeper.properties；注意自行在三台cluster1、cluster2、cluster3上修改此文件

   ```java
   dataDir=/opt/lg/kafkaZooData   //注意自己创建具体目录位置
   # the port at which the clients will connect
   clientPort=2181
   # disable the per-ip limit on the number of connections since this is a non-production config
   
   maxClientCnxns=100
   tickTime=2000
   initLimit=10
   syncLimit=5
   
   server.1=cluster1:2888:3888
   server.2=cluster2:2888:3888
   server.3=cluster3:2888:3888
   ```

   -  创建myid文件，进入/opt/lg/kafkaZooData，创建myid文件，将三个服务器上的myid文件分别写入1，2，3, --myid是zk集群用来发现彼此的标识，必须创建，且不能相同 

   - 验证配置：进入kafka目录 执行启动zookeeper命令   

     ```java
      ./bin/zookeeper-server-start.sh config/zookeeper.properties &  
     ```

     三台机器都执行启动命令，查看zookeeper的日志文件，没有报错就说明zookeeper集群启动成功了。 注意等三台都启动了，控制台就不报错了

3. 配置Kafka集群

   - /opt/lg/packages/kafka_2.11-1.1.0/config下的server.properties ；注意自行在三台cluster1、cluster2、luster3上修改此文件

   ```java
   # The id of the broker. This must be set to a unique integer for each broker.
   broker.id=1  // cluster1=1  cluster2=2  cluster3=3
   
   # A comma separated list of directories under which to store log files
   log.dirs=/opt/lg/kafka-logs   //自行创建
   
   # root directory for all kafka znodes.
   zookeeper.connect=cluster1:2181,cluster2:2181,cluster3:2181
   ```

   验证配置：启动kafka集群，进入kafka目录，执行如下命令 

   ```java
   ./bin/kafka-server-start.sh config/server.properties & 
   ```

4. 开启与关闭

   开启时先开zookeeper , 再开 kafka

   关闭时先关 kafka , 再关 zookeeper 

#   Spark Streaming 程序搭建
> 主要总结了 开发过程中遇到的问题

###	问题
1.jar包版本问题：Spark版本为2.1.0版本，然而在spark 1.5.2版本之后 org/apache/spark/Logging 已经被移除了。 由于spark-streaming-kafka 1.6.3版本中使用到了logging。解决方式：在pom.xml文件中增加配置 

```java
<dependency>
	<groupId>org.apache.spark</groupId>
	<artifactId>spark-streaming-kafka-0-8_2.11</artifactId>
	<version>2.0.0</version>
</dependency>
```
报错：Exception in thread "main" java.lang.NoClassDefFoundError: org/apache/spark/Logging

**在非maven 构建的工程中，解决方法未知**

2.IDEA打jar包时注意添加class path

```java
/opt/wwd/scala/lib/scala-swing_2.11-1.0.2.jar
/opt/wwd/scala/lib/scala-library.jar
/opt/wwd/scala/lib/scala-actors-2.11.0.jar
/opt/lg/packages/kafka_2.11-1.1.0/libs/zookeeper-3.4.10.jar
/opt/lg/packages/kafka_2.11-1.1.0/libs/zkclient-0.10.jar
/opt/lg/packages/kafka_2.11-1.1.0/libs/kafka-clients-1.1.0.jar
/opt/lg/packages/spark-streaming-kafka_2.11-1.6.2.jar      //注意：版本问题
/opt/wwd/packages/spark-2.2.0-bin-hadoop2.7/jars/log4j-1.2.17.jar
/opt/wwd/packages/spark-2.2.0-bin-hadoop2.7/jars/slf4j-api-1.7.16.jar
/opt/lg/packages/kafka_2.11-1.1.0/libs/kafka_2.11-1.1.0.jar
```

