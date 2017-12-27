
title: 关于hive-update
date: 2017-04-19 21:49:54
tags: [youdaonote]
---

ACID
---
- A: atomicity。原子性，一个操作要么成功要么失败，不能存在中间状态。
- C：consistency。一致性。一旦某个操作完成之后，后续所有的操作都能访问到上个操作完成之后的状态
- I：isolation。隔离性。一个未完成的操作只对当前操作用户有影响，不能影响到其他用户
- D：durability。持久性。一个操作完成之后，就会把成功的状态持久化，即便系统挂掉也要持久化。

0.13版本之前，只在分区层次上支持原子性、一致性、持久性。隔离性通过调整可用的锁机制来达成(通过zk或者内存)。0.13之后完全支持了row级别的ACID，这样就可以在别人读取数据的时候添加行，也不会有影响。

#### 用例
一般具有ACID特性的事务适用于以下场景
 - 流式数据接受。许多用户使用flume、storm、kafka等将数据放到hadoop集群中。这些工具可能会以每秒几百条甚至更多的速度写入。hive只能每十五分钟或者1个小时创建一个分区来接受他们。添加分区的策略会导致表里的分区暴增。也可以把流式数据放到既有的分区中，但是这会导致用户脏读（他们可能会看到一些他们开始查询时还没有的数据），还有就是会有很多的小文件产生，这对namenode也是一个很大的压力。有了ACID之后，就可以解决这个问题了
 - 慢慢改变的维度表。在一个典型的星型数据仓库架构中，维度表可能会随着时间推移慢慢修改。例如，零售商可能会慢慢增开新店，那么新店就应该被添加到store表里，或者已经存在的店面可能会修改店面大小或者其他的因素。这些改变都会产生数据的insert或者update。从0.14开始，支持了
 - 数据重导。有时手机来的数据发现不对，需要纠正。或者，开始有90%的数据，后面又有全量数据了。或者业务规则改变需要修改对应规则的一些数据。0.14之后用户可以insert、update、delete。
 - 使用SQL MERGE语句批量更新。

#### 限制
 - BEGIN,COMMIT,ROLLBACK目前还不支持。所有的语言都是auto-commit。后面版本会支持
 - 只支持ORC的文件格式。
 - 默认这种事务是关闭的。
 - 数据表必须是bucketed的。外部表不支持ACID，因为它不受compactor的控制[HIVE-13175](https://issues.apache.org/jira/browse/HIVE-13175)
 - 在一个非ACID的session里执行ACID操作是不可以的。hive的事务manager必须设置成  ` org.apache.hadoop.hive.ql.lockmgr.DbTxnManager `才能使用ACID表。
 - 目前只支持快照级别的隔离性。当一个查询开始后，会给它提供一个一致的数据快照。不支持dirty read, read committed, repeatable read, seializable。
 - 已经存在的zk或者内存锁manager与此类事务并不兼容。这里不详说了，参考[这里](https://cwiki.apache.org/confluence/display/Hive/Hive+Transactions#HiveTransactions-BasicDesign)
 - 使用oracle作为元数据库可能会有些问题，设置`datanucleus.connectionPoolingType=DBCP`

 #### 语法改变
 
 - 添加了 `SHOW TRANSACTIONS`、`SHOW COMPACTIONS`、`ABORT TRANSACTIONS`
 - `SHOW LOCKS`命令改成了提供事务相关的锁的信息。如果在使用ak或者内存的lock manager的话，你会发现其实输出的东西没什么区别。
 - alter table的时候增加了一个是否执行表或者分区的压缩。通常用户不用管，系统会自动做这些事情。然而，如果compaction被关闭了，那么系统就不会自动处理了。alter table用来初始化压缩。具体参考[这里](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL#LanguageManualDDL-AlterTable/PartitionCompact)。这会把压缩请求放进一个队列里。要看压缩进度的话，就使用`SHOW COMPACTIONS`语句。
 
#### 基本设计
HDFS是不支持直接修改文件的。当有人写数据到文件的时候也不提供读的一致性。为了提供这些特性，我们使用了其他的数据仓库工具。表和分区的数据存在一系列的base文件里。新的insert、update、delete存在delta文件中，读取的时候把这些更新都执行上。

##### base和delta文件夹
以前一个分区的所有文件都放在一个单独的目录下。ACID的writer会有一个base文件的目录，还有一堆delta文件的目录。下面是一个没有分区的数据表“t”
```
hive> dfs -ls -R /user/hive/warehouse/t;
drwxr-xr-x   - ekoifman staff          0 2016-06-09 17:03 /user/hive/warehouse/t/base_0000022
-rw-r--r--   1 ekoifman staff        602 2016-06-09 17:03 /user/hive/warehouse/t/base_0000022/bucket_00000
drwxr-xr-x   - ekoifman staff          0 2016-06-09 17:06 /user/hive/warehouse/t/delta_0000023_0000023_0000
-rw-r--r--   1 ekoifman staff        611 2016-06-09 17:06 /user/hive/warehouse/t/delta_0000023_0000023_0000/bucket_00000
drwxr-xr-x   - ekoifman staff          0 2016-06-09 17:07 /user/hive/warehouse/t/delta_0000024_0000024_0000
-rw-r--r--   1 ekoifman staff        610 2016-06-09 17:07 /user/hive/warehouse/t/delta_0000024_0000024_0000/bucket_00000
```

##### compactor
compactor是在metastore中运行的一系列的后台进程，用来支持ACID系统。包含Initiator, Worker, Cleaner,AcidHouseKeeperService等。

###### delta文件压缩

随着修改表数据的操作越来越多，delta文件也越来越多，需要压缩提高后面的处理效率。类似hbase，也是有minor compaction和major compaction
 - Minor compaction把已经存在的同一个bucket的多个delta文件合并成一个
 - Majro compaction把bucket中的base文件和delta文件合并到一个新的base文件。这个会耗时多一些，不过也更高效。
 
压缩工作是后台自动运行的，不会阻止并发的读写操作。执行压缩之后，系统会等用户读完所有的旧文件之后移除这些旧文件。

###### Initiator
这个模块是负责发现哪些table或者分区需要压缩。使用`hive.compactor.initiator.on`参数启用。每个compaction task负责处理一个分区(没分区的就是一个表)。如果连续压缩失败，超过`hive.compactor.initiator.failed.compacts.threshold`设置的阈值,就停止compaction了。

###### Worker
一个Woker处理一个compaction task。通常是MR任务，名字格式是：主机名-compactor-数据库.表名.分区。使用`hive.compactor.worker.threads `设置每个metastore启动的compactor线程数。

###### Cleaner
用来删除压缩之后，之前的一些旧文件

###### AcidHouseKeeperService
找到心跳超时`hive.txn.timeout`的事务，终止它。释放资源。

#### SHOW COMPACTIONS
这个命令用来显示目前正在运行的compaction信息，还有最近的压缩历史。

#### 事务/锁 manager
添加了新的逻辑实体`事务管理器`，包含原有的`database/table/partition lock manager`的概念在里面。原来默认的`hive.lock.manager`是` org.apache.hadoop.hive.ql.lockmgr.zookeeper.ZooKeeperHiveLockManager`。新的事务管理器现在也负责管理事务锁了。默认的`DummyTxnManager `跟原有版本的hive一样：并没有事务，使用hive.lock.manager来为表、分区、数据库创建lock manager。新的`DbTxnManager `使用hive metastore中的`DbLockManager`管理所有的锁与事务（系统挂掉也不会碍事儿）。这就是说启用事务特性之后，并没有必要弄什么zookeeper来搞这些了。为了避免客户端挂掉、离开当前事务、或者锁悬停的情况，我们需要锁的拥有者和事务发起者以一定频率发送心跳给metastore。如果心跳超时几次，那么这个锁或者食物就会被终止。

在1.3.0版本中，DbLockManager会持续重试去获取锁。每次获取不到，都会double下一次获取锁的时间间隔，再重试，以支持短时间的查询，也降低metastore压力。假设初始等待时间是100ms，如果重试10次的话，总时间会是91m:42s:300ms。

注意DbTxnManager的lock manager会获取所有表的锁。默认情况下，对于非事务表的insert操作会获取一个排它锁，不允许其他的读写操作。虽然从技术角度来看是正确的，但是违背hive的传统工作方式的。为了向后兼容，提供了`hive.txn.strict.locking.mode `参数，可以让lock manager在对于非事务表insert操作的时候获取共享锁。这样就保证在读取的时候，不会出现数据被drop的情况。注意，对于事务表，insert操作是总会获取共享锁的，因为这些表在存储层支持MVCC架构，即便有并发写的时候也能够提供对于读的强一致性(快照隔离).

###### 配置
客户端
 - hive.support.concurrency - true
 - hive.enforce.bucketing - true(2.0版本后不需要)
 - hive.exec.dynamic.partition.mode - nonstrict
 - hive.txn.manager - org.apache.hadoop.hive.ql.lockmgr.DbTxnManager

服务端(metastore)
 - hive.compactor.initiator.on - true
 - hive.compactor.worker.threads - 正数就行，看情况


#### 事务相关的新配置参数

#### 表属性
如果一个表要使用ACID写操作，那么必须在表上设置`transactional=true`。从0.14.0开始，一旦一个表通过·TBLPROPERTIES ("transactional"="true")·定义为一个ACID表之后，就不能在改成非ACID表了，就是说不能执行TBLPROPERTIES ("transactional"="false") 这种修改。还有，hive.txn.manager必须设置为org.apache.hadoop.hive.ql.lockmgr.DbTxnManager, 可以再hive-site.xml里设置，也可以在session开始的时候设置。不然insert就会按照老的方式运行，delete操作会被禁止。

如果一个表的owner不想让系统自动决定什么时候compact，那就在表上设置`NO_AUTO_COMPACTION`。

在表创建之后表属性通过TBLPROPERTIES语句设置。

更多关于压缩的选项可以通过TBLPROPERTIES设置。可以在创建表时设置，也可以执行修改表的时候设置。例如，要覆盖一个压缩任务的MR参数，就可在创建表或者修改表的时候加上`compactor.<mr property>=xx`。

表级别设置
```sql

CREATE TABLE table_name (
  id                int,
  name              string
)
CLUSTERED BY (id) INTO 2 BUCKETS STORED AS ORC
TBLPROPERTIES ("transactional"="true",
  "compactor.mapreduce.map.memory.mb"="2048",     -- specify compaction map job properties
  "compactorthreshold.hive.compactor.delta.num.threshold"="4",  -- trigger minor compaction if there are more than 4 delta directories
  "compactorthreshold.hive.compactor.delta.pct.threshold"="0.5" -- trigger major compaction if the ratio of size of delta files to
                                                                   -- size of base files is greater than 50%
);
```
单次请求时候设置
```sql
ALTER TABLE table_name COMPACT 'minor' 
   WITH OVERWRITE TBLPROPERTIES ("compactor.mapreduce.map.memory.mb"="3072");  -- specify compaction map job properties
ALTER TABLE table_name COMPACT 'major'
   WITH OVERWRITE TBLPROPERTIES ("tblprops.orc.compress.size"="8192");         -- change any other Hive table properties
```
参考：
https://cwiki.apache.org/confluence/display/Hive/Hive+Transactions

https://hortonworks.com/hadoop-tutorial/using-hive-acid-transactions-insert-update-delete-data/
