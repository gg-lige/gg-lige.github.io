---
layout:     post
title:      Zeppelin自定义Interpreter插件开发
subtitle:   写一个新的Interpreter解析器
date:       2018-06-08
author:     GG
header-img: img/post-bg-1.jpg
catalog: true
tags:
    - Zeppelin
---



>
>  主要为在 Zeppelin 0.7.3 中自定义 Interpreter插件 的开发过程
>

#	1 前言
​	本文是在   `Zeppelin` 中自定义  `Interpreter`  插件的说明手册。包括  `Zeppelin`  相关内容的简介，自定义 `Interpreter` 的开发过程，及自定义 `Interpreter` 示例的使用。



# 2 系统需求

​	以下说明 Zeppelin 0.7.3 自定义 Interpreter 插件的基本需求。

## 2.1 系统硬件

​	Intel x86_64 服务器。

## 2.2 系统软件
​	Zeppelin 自定义 Interpreter 插件可在 Windows 或Linux 下开发。测试过程中使用 Windows 环境为Win10 企业版；Linux使用虚拟机环境： Ubuntu 16.04 LTS Server x86。



# 3 Zeppelin简介

## 3.1 Zeppelin

​	`Apache Zeppelin` 是一个开源的基于 `web` 的 `notebook` 的实现，通过它可以在 `web` 上与`spark/hive/impala/flink/R/kylin` 等十多种数据处理引擎实现 `SQL` 方式、代码方式交互式的分析查询，并且可以支持图表展现，结果下载和分享等。`zeppelin` 的 `notebook` 功能是通过 `jupyter` 的内核实现的。

## 3.2 Zeppelin Interpreter

​	Zeppelin中最核心的概念是`     Interpreter` 解析器，`Interpreter`是一个插件,它允许用户使用一种指定的语言或数据处理器。

​	以下是 Interpreter的运行原理图：![img](https://zeppelin.apache.org/assets/themes/zeppelin/img/interpreter.png) 

​	和其他类似的系统（HUE）一样，`zeppelin` 对于每一类支持的计算引擎都可以创建多个配置，每一个插件的配置称为一个 `Interpreter`，相同类型的 `Interpreter` 称为一个 `Interpreter group`，一个 `group` 内的所有 `Interpreter` 可以共享一个 `JVM` 进程，在 `zeppelin` 启动时启动 `ZeppelinServer` 进程，当用户选择使用某一个 `interpreter` 的时候会通过 `lazy` 的方式启动一个进程负责该 `interperter` 的请求，`ZeppelinServer` 和 `interpreter` 进程之间的通信通过 `thrift` 【是一种可扩展、跨语言的服务开发框架 ，主要用于各个服务之间的RPC通信，其服务端和客户端可以用不同的语言来开发 】完成的。 

**Attention1: **Zeppelin 可以接入不同的处理引擎供用户使用，当我们每接入一个处理引擎则需要为其配置一个 interpreter,注意：zeppelin 的 interpreter 不仅包含数据处理的引擎（`spark/hive/impala/flink/R/kylin/file/hbase`），也包括编程语言和语法格式(`python/markdown/shell`)。

**Attention2:** 运行模式，`interpreter`有三种运行模式：shared,scoped,isolated。在配置interpreter时可以选择其中之一。在`shared`模式下，每个notebook绑定到一个Interpreter上，他们共享interpreter实例；在`scoped `模式下，在相同的 interpreter进程中各个notebook会创建一个新的interpreter实例；而在`isolated`模式下各个notebook将创建一个新的interpreter实例。

**Attention3:** 每一个`Interpreter`都属于换一个`InterpreterGroup`，同一个`InterpreterGroup`的`Interpreters`可以相互引用，例如`Spark InterpreterGroup`包含5个分组,其中的`SparkSqlInterpreter `可以引用 `SparkInterpreter `以获取 `SparkContext`，因为他们属于同一个`InterpreterGroup`。  

## 3.3 核心功能

​	可以支持多个语言REPL（ Read–Eval–Print Loop ，简单的的交互式的编程环境 ）的解释器。 	

​	**用户角度**：

​	①**在一个Note中混合使用多种语言** 

​	用户可以根据需要完成的任务的类型，选择最合适的语言来实现，不再受限于单个语言的特性 。例如：MarkDown能做出漂亮的文档，python和R有大量的科学计算、机器学习和可视化包，scala与Spark有天然的血缘关系，结合它们可极大提升生产力。

​	②**数据分析和机器学习的本质是一个不断迭代的计算过程**

数据科学家和算法工程师对数据分析平台最看中是通过 `丰富的组件` 来全面的表达出自己的业务需求，通过 `可调试性` 来逐步展开实际业务分析的需求，二者相辅相成

​	**平台角度**：

​	①**统一开发环境** 

​	在数据分析团队中，当需要同时维护多种环境，如R、Python远程桌面开发环境，R和Python都是通过第三方扩展包进行扩展的，成员熟练使用的包，也有所不同，给运维造成了较大压力。实现一个集中式的分析工具，在服务端集中于一处，统一配置R、Python、Spark、Hadoop、Hive等开发环境，可以显著降低运维成本。 

​	②**统一安全控制** 

​	B/S系统方便进行集中式用户权限控制，并且由于该平台”看到”的是各种语言的源码，可以对恶意代码进行监测和过滤，避免有意或者无意地对系统造成的损害。

#	4 自定义 Interpreter 插件开发

​	由于 Zeppelin 目前支持的解析器还不够完善，所以自定义 Interpreter 并关联到我们需要使用的数据处理引擎或编程语言（例如，MongoDB，Mysql，ECharts等） 非常有必要。
####	4.1 下载 Zeppelin 源码


​	从`Zeppelin`官方可以下载到目前` Zeppelin`的最新版本代码：[**http://mirrors.hust.edu.cn/apache/zeppelin/zeppelin-0.7.3/zeppelin-0.7.3-bin-all.tgz**    [此链接包含Zeppelin目前所有内置 Interpreter , 可适用于 Windows 和Linux]

#### 4.2 自定义解析器 NewInterpreter

<u>Step1:</u> Maven 构建工程时 选择与当前所使用的zeppelin同版本的依赖包zeppelin-interpreter 

```java
<dependency>
    <groupId>org.apache.zeppelin</groupId>
    <artifactId>zeppelin-interpreter</artifactId>
    <version>0.7.3</version>
    <scope>provided</scope>
</dependency>
```

​	由于Zeppelin运行环境已经有了该依赖包，所以我们再创建自定义Interpreter插件的时候只需要在代码中对其依赖，打包过程中不需要打包该包。所以使用provided依赖方式.

**Attention1:** 添加内部开发的Interpreter 的相关依赖包. eg: MySql 则需要添加 mysql-connector-java 等相关依赖。

**Attention2:** 一般需要调试打印日志，可添加Log 的相关依赖。



<u>Step2:</u> 创建实现类 并继承 org.apache.zeppelin.interpreter.Interpreter 并实现相应的抽象方法

​	以下是核心的几个抽象方法，一般都需要重写。

```java
public class NewInterpreter extends Interpreter {
    //官网样例给出可按此设置，但我在具体实施时报错,因此我测试时使用的名称是 %NewInterpreter
    //若设置成功，以NI名称注册，那么Zeppelin前端在调用查询时，只需要指定后端引擎名称%NI即可
    static {
  		Interpreter.register("NI", NewInterpreter.class.getName());
	}
    
    //构造器
    public NewInterpreter(Properties property) {
        super(property);
    }
    //打开解析器，设置程序的初始化信息
    @Override
    public void open() {

    }
    //关闭解析器，释放相关资源
    @Override
    public void close() {

    }
    /**
     * 最主要的程序执行方法，负责前后台交互。以同步方式运行代码并返回结果
     * 核心思想：解析命令=>实例化NewIntepreter操作客户端=>操作NewIntepreter客户端进行数据查询
     * => 获取返回结果 封装成InterpreterResult对象
     *
     * @param st Zeppelin 前台交互式界面的用户输入,  注意不包含第一行的  %NewInterpreter  ---->新的解析器的名称
     * @param context 当前Interpreter 插件的上下文，包含插件的配置信息（包括noteId,replName,paragraphId,paragraphText等）
     * @return 状态信息[SUCCESS,INCOMPLETE,ERROR,KEEP_PREVIOUS_RESULT] 、以数据类型[TEXT,HTML,ANGULAR,TABLE,IMG,SVG,NULL]输出结果
     */
    @Override
    public InterpreterResult interpret(String st, InterpreterContext context) {
        if (!StringUtils.isEmpty(st)){
            int firstIndex = st.indexOf('"');
            int lastIndex = st.indexOf('"');
            return new InterpreterResult(InterpreterResult.Code.SUCCESS,st.substring(firstIndex+1,lastIndex));
        }else
            return  new InterpreterResult(InterpreterResult.Code.ERROR,"Please input as: System.out.println(\"***\");");
    }
    //可选；调用cancel 方法中止 interpret 方法
    @Override
    public void cancel(InterpreterContext context) {

    }
    /**
     * 返回形式：动态表单处理
     * @return FormType.SIMPLE enables simple pattern replacement (eg. Hello ${name=world}),
     *          FormType.NATIVE handles form in API
     *          FormType.NULL
     */
    @Override
    public FormType getFormType() {
        return null;
    }
    // interpret 方法运行的百分比，0-100
    @Override
    public int getProgress(InterpreterContext context) {
        return 0;
    }
}
```

​	还有一些其他方法，例如 getScheduler()等方法可自行实现，具体可查看`org.apache.zeppelin.interpreter.Interpreter` 类。



Step3: 创建 interpreter-setting.json 并配置相应信息

​	将interpreter-setting.json 文件推荐放置在 {ZEPPELIN_INTERPRETER_DIR}/{YOUR_OWN_INTERPRETER_DIR}/interpreter-setting.json}

文件示例如下：

```java
[
  {
    "group": "your-group",
    "name": "your-name",
    "className": "your.own.interpreter.class",
    "properties": {
      "properties1": {
        "envName": null,
        "propertyName": "property.1.name",
        "defaultValue": "propertyDefaultValue",
        "description": "Property description"
      },
      "properties2": {
        "envName": PROPERTIES_2,
        "propertyName": null,
        "defaultValue": "property2DefaultValue",
        "description": "Property 2 description"
      }, ...
    }
  },
  {
    ...
  } 
]
```



Step4: Maven打包

```
mvn clean package
```

#### 4.3 插件部署

<u>Step1</u>: 拷贝NewInterpreter 插件包

​	在 {ZEPPELIN_HOME}/interpreter/下创建 NewInterpreter 文件夹，并将 4.2  Step4 中打好的所有依赖包拷贝至此文件夹下。



<u>Step2</u>: 实现类的配置 

​	修改{ZEPPELIN_HOME}/conf/zeppelin-site.xml  ，主要添加 注册解析器的相关类

```java
<property>
  <name>zeppelin.interpreters</name>  				    		   <value>org.apache.zeppelin.spark.SparkInterpreter,org.apache.zeppelin.spark.PySparkInterpreter,org.apache.zeppelin.rinterpreter.RRepl,org.apache.zeppelin.rinterpreter.KnitR,org.apache.zeppelin.spark.SparkRInterpreter,org.apache.zeppelin.spark.SparkSqlInterpreter,org.apache.zeppelin.spark.DepInterpreter,org.apache.zeppelin.markdown.Markdown,org.apache.zeppelin.angular.AngularInterpreter,org.apache.zeppelin.shell.ShellInterpreter,org.apache.zeppelin.file.HDFSFileInterpreter,org.apache.zeppelin.flink.FlinkInterpreter,,org.apache.zeppelin.python.PythonInterpreter,org.apache.zeppelin.python.PythonInterpreterPandasSql,org.apache.zeppelin.python.PythonCondaInterpreter,org.apache.zeppelin.python.PythonDockerInterpreter,org.apache.zeppelin.lens.LensInterpreter,org.apache.zeppelin.ignite.IgniteInterpreter,org.apache.zeppelin.ignite.IgniteSqlInterpreter,org.apache.zeppelin.cassandra.CassandraInterpreter,org.apache.zeppelin.geode.GeodeOqlInterpreter,org.apache.zeppelin.postgresql.PostgreSqlInterpreter,org.apache.zeppelin.jdbc.JDBCInterpreter,org.apache.zeppelin.kylin.KylinInterpreter,org.apache.zeppelin.elasticsearch.ElasticsearchInterpreter,org.apache.zeppelin.scalding.ScaldingInterpreter,org.apache.zeppelin.alluxio.AlluxioInterpreter,org.apache.zeppelin.hbase.HbaseInterpreter,org.apache.zeppelin.livy.LivySparkInterpreter,org.apache.zeppelin.livy.LivyPySparkInterpreter,org.apache.zeppelin.livy.LivyPySpark3Interpreter,org.apache.zeppelin.livy.LivySparkRInterpreter,org.apache.zeppelin.livy.LivySparkSQLInterpreter,org.apache.zeppelin.bigquery.BigQueryInterpreter,org.apache.zeppelin.beam.BeamInterpreter,org.apache.zeppelin.pig.PigInterpreter,org.apache.zeppelin.pig.PigQueryInterpreter,org.apache.zeppelin.scio.ScioInterpreter,org.apache.zeppelin.mongodb.MongoDbInterpreter,com.lg.MysqlInterpreter,
```

 **com.lg.NewInterpreter**

```java
 </value> 
 <description>Comma separated interpreter configurations. First interpreter become a default</description>
</property>
```



<u>Step3</u>：启动Zeppelin

​	Windows 下启动命令为： 

```java

```

​	Linux 下启动命令为：

```
{ZEPPELIN_HOME}/bin/zeppelin-daemon.sh start
```



<u>Step4:</u> Zeppelin 管理界面添加 NewInterpreter 解析器的相关配置

​	用户anonymous—>Interpreter

![1528548172021](/img/post-2018-06-09-1.PNG)

​	添加 自定义的 Interpreter—>Create—>此时在Interpreter group 选项中会出现我们自定义的 NewInterpreter—>Save。

![1528548500283](/img/post-2018-06-09-2.PNG)



**Attention1**:这里的Properties 中的参数与 4.2 中 Step3 中配置 相通，具体可根据我们要添加的解析器来进行配置，eg: Mongo 可能需要配置mongo.server.host、mongo.server.port、mongo.server.database、mongo.server.username、mongo.server.password等。

#### 4.4 运行

​	新建一个Note,绑定 NewInterpreter 解析器。蓝色表示对当前Note 绑定此解析器，白色未绑定。

![1528549992400](/img/post-2018-06-09-3.PNG)

​	执行命令,即可看到返回结果

![1528547950876](/img/post-2018-06-09-4.PNG)