
title: kudu简介
date: 2017-11-02 15:54:09
tags: [youdaonote]
---

为apache hadoop平台开发的列式存储管理器。kudo拥有hadoop生态的很多特性：廉价硬件、水平扩展、高可用。

##### 背景

- HDFS可以快速追加和查询，尤其是顺序扫描，如果能和parquet结合就可以有更好的表现。
- hbase擅长处理易变的数据，快速查询和写单条数据。


kudu介于两者之间。


kudu结合了parquet和hbase的特性，创建了一个本地列式存储的系统。


kudu特性:
- 动态管理分区，动态创建明天的分区，删除100天前的分区
- 删除指定range的数据
- 支持kerberos认证，内部的task也使用kerberos给的token进行内部验证
- server之间、server和client之间TLS加密处理，不影响框架处理速度。如果都在本机，就不用走这个
- 三种访问角色：client、admin、service[给daemon使用的]。也有白名单之类
- ksck工具链接到master查看整体状态
- rack aware


- 严格结构化数据，不支持blob
- 多语言API
- 与spark和impala整合好的API
- 所有table都被分区，每个分区叫做tablet，每个tablet有多个备份，使用raft达成最终一致，**都存在本地磁盘**，并不基于HDFS。HDFS的机制不能支持分布式存储，不能快速处理failover，还有内存部分快速访问的设计，这里有些像kafka的思想。
- metadata，把所有的metadata放在内存中，以便快速访问或处理
- spark、impala、MR、flume、drill整合


给spark带来啥
---
- 像parquet的性能，而且insert和update都无延迟
- pushdown predicate filter，更快更高效的扫描
- 相比parquet还有主键索引
- 在master中metadata的查找，精准找到目标partition

##### 针对spark的优化

- 只读取有关的字段
- 自动将where转化为kudu predicate
- kudu predicate自动转化为主键扫描等

##### 最佳实践
可以在spark中注册临时表，然后针对临时表执行kudu查询的操作。


##### 其他
如果查询的数据在内存，那么kudu比parquet快，否则差不多会比parquet慢一倍。单条数据处理，虽然比较快了，但还是会比phenix慢很多。

##### 场景
- 顺序或随机的读写
- 最小化数据延迟

- 时间序列数据
- 实时报表/实时数据仓库(ODS)



参考：
- https://www.youtube.com/watch?v=NEg1nn9UyBY
- https://www.youtube.com/watch?v=CLP6jHs2z14
