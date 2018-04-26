---
layout:     post
title:      HBase- 架构&常用Shell
subtitle:   架构 & 常用 Shell 总结
date:       2018-04-26
author:     GG
header-img: img/post-bg-1.jpg
catalog: true
tags:
    - HBase
	- Shell
    - 架构
---



>  
>  本文是HBase 相关 Shell 的总结。
>  同时包含其主要的架构图
> 

#   架构图
![image](E:/git/gg-lige.github.io/img/post-2018-04-26.jpg)

HBase有三个主要组成部分：客户端库，主服务器和区域服务器。区域服务器可以按要求添加或删除。

1. 主服务器

- 分配区域给区域服务器并在Apache ZooKeeper的帮助下完成这个任务。
- 处理跨区域的服务器区域的负载均衡。它卸载繁忙的服务器和转移区域较少占用的服务器。
- 通过判定负载均衡以维护集群的状态。
- 负责模式变化和其他元数据操作，如创建表和列。


2. 区域服务器

   区域：区域只不过是表被拆分，并分布在区域服务器。
- 与客户端进行通信并处理数据相关的操作。
- 句柄读写的所有地区的请求。
- 由以下的区域大小的阈值决定的区域的大小。

3. Zookeeper

- Zookeeper管理是一个开源项目，提供服务，如维护配置信息，命名，提供分布式同步等
- Zookeeper代表不同区域的服务器短暂节点。主服务器使用这些节点来发现可用的服务器。
- 除了可用性，该节点也用于追踪服务器故障或网络分区。
- 客户端通过与zookeeper区域服务器进行通信。
- 在模拟和独立模式，HBase由zookeeper来管理

#	启动
命令行启动HBase
```
./hbase shell
```

#   常用命令
主要总结了 查看状态的命令 和 增删改查 的命令。

###	查看状态
1. status：命令返回包括在系统上运行的服务器的细节和系统的状态
```
hbase(main):003:0> status
1 active master, 0 backup masters, 4 servers, 0 dead, 0.5000 average load
```
2. version: 该命令返回HBase系统使用的版本
```
hbase(main):004:0> version
1.2.6, rUnknown, Mon May 29 02:25:32 CDT 2017
```
3. whoami: 该命令返回HBase用户详细信息
```
hbase(main):007:0> whoami
root (auth:SIMPLE)
    groups: root
```
###	增删改查
1. 创建表
```
create 'emp','personal data','professional data'
0 row(s) in 2.4700 seconds
=> Hbase::Table - emp
```
2. 验证创建
```
hbase(main):009:0> list
TABLE                  
emp                            
1 row(s) in 0.0130 seconds
=> ["emp"]
```
3. 禁用表
```
hbase(main):010:0> disable 'emp'
0 row(s) in 4.5280 seconds
hbase(main):011:0> list
TABLE                           
emp                           
1 row(s) in 0.0100 seconds
=> ["emp"]
```
4. 是否被禁用
```
hbase(main):014:0> is_disabled 'emp'
true                             
0 row(s) in 0.0100 seconds
```
5. 使其可用
```
hbase(main):015:0> enable 'emp'
0 row(s) in 2.3610 seconds
hbase(main):016:0> scan 'emp'
ROW                                              COLUMN+CELL
0 row(s) in 0.0210 seconds
```
6. 描述表
```
hbase(main):017:0> describe 'emp'
Table emp is ENABLED                            
emp                             
COLUMN FAMILIES DESCRIPTION     

{NAME => 'personal data', BLOOMFILTER => 'ROW', VERSIONS => '1', IN_MEMORY => 'false', KEEP_DELETED_CELLS => 'FALSE', DATA_BLOCK_ENCODING => 'NONE', TTL => 'FOREVER', COMPRESSION => 'NONE',
 MIN_VERSIONS => '0', BLOCKCACHE => 'true', BLOCKSIZE => '65536', REPLICATION_SCOPE => '0'}    

{NAME => 'professional data', BLOOMFILTER => 'ROW', VERSIONS => '1', IN_MEMORY => 'false', KEEP_DELETED_CELLS => 'FALSE', DATA_BLOCK_ENCODING => 'NONE', TTL => 'FOREVER', COMPRESSION => 'NO
NE', MIN_VERSIONS => '0', BLOCKCACHE => 'true', BLOCKSIZE => '65536', REPLICATION_SCOPE => '0'}   

2 row(s) in 0.0350 seconds
```
7. 修改表的列族单元格的最大数目 & 修改为只读的
```
hbase(main):018:0> alter 'emp',NAME=>'personal data',VERSIONS=>5
Updating all regions with the new schema...
1/1 regions updated.
Done.
0 row(s) in 2.6180 seconds

hbase(main):019:0> alter 'emp',READONLY
Updating all regions with the new schema...
1/1 regions updated.
Done.
0 row(s) in 2.5650 seconds
```
8. 表是否存在
```
hbase(main):020:0> exists 'emp'
Table emp does exist 

0 row(s) in 0.0130 seconds
```
9. 添加元素
```
hbase(main):021:0> put 'emp' ,'1','personal data:name','raju'
0 row(s) in 0.1480 seconds

hbase(main):022:0> put 'emp' ,'1','personal data:city','hyderhad'
0 row(s) in 0.0120 seconds

hbase(main):023:0> put 'emp' ,'1','professional data:designation','manager'
0 row(s) in 0.0110 seconds

hbase(main):024:0> put 'emp' ,'1','professional data:salary','50000'
0 row(s) in 0.0100 seconds

hbase(main):026:0> put 'emp' ,'2','personal data:name','ravi'
0 row(s) in 0.0130 seconds

hbase(main):027:0> put 'emp' ,'2','personal data:city','chennai'
0 row(s) in 0.0090 seconds

hbase(main):028:0> put 'emp' ,'2','professional data:designation','srengineer'
0 row(s) in 0.0090 seconds

hbase(main):002:0> put 'emp' ,'2','professional data:salary','50000'
0 row(s) in 0.1060 seconds
```
10. 获取某行元素
```
hbase(main):006:0> get 'emp' ,'1'
COLUMN                                           CELL                                                     
 personal data:city                              timestamp=1524726237598, value=Dali                                                                                                         
 personal data:name                              timestamp=1524725507906, value=raju                                                                                                         
 professional data:designation                   timestamp=1524725575919, value=manager                                                                                                      
 professional data:salary                        timestamp=1524725592180, value=50000                                                                                                        
4 row(s) in 0.0410 seconds
```

11. 获取某个单元格的元素
```
hbase(main):008:0> get 'emp','1',{COLUMN=>'personal data:name'}
COLUMN                                           CELL                            
 personal data:name                              timestamp=1524725507906, value=raju                                     
1 row(s) in 0.0100 seconds
```

12. 统计表中元素
```
hbase(main):012:0> count 'emp'
2 row(s) in 0.0330 seconds

=> 2
```

13. 截断表，不删除定义，只是清空表
```
hbase(main):013:0> truncate 'emp'
Truncating 'emp' table (it may take a while):
 - Disabling table...
 - Truncating table...
0 row(s) in 7.5970 seconds
```

14. 授予权限或者 收回 权限
```
hbase(main):016:0> grant 'lg','RWXCA'
hbase(main):018:0> revoke 'lg'
```


#	终止

可通过 exit 或者 ctrl+v 退出。
