
title: ambari中启用hive-ACID事务
date: 2017-04-20 20:21:45
tags: [youdaonote]
---

场景
 - 数据重新执行
 - 数据流重新执行
 - 逐渐改变的维度
 - 维度历史变更

标准sql通过insert、update、delete、事务还有最近出现的merge方式来提供acid操作。这些已经被证明足够使用的了。

#### 概念
 - 事务表。hive支持单表事务，但是目标表必须需要声明是支持事务的。
 - 分区表。hive支持表分区，把数据分开以提供快速查询。分区与ACID是相互独立的概念。通常大表都是有分区的。
 - ACID操作（insert/update/delete）。
 - 主键
 - 流式数据。数据可以通过storm、flume等流式进入hive的事务表中
 - 优化并发。
 - 压缩。数据必须时段性的被压缩一下，节省空间，优化数据访问。最好让系统自动处理这件事情。，不过也可以设置外部的调度器。
 
#### 启用ACID事务

#### hello
创建一个事务表，并插入一些数据
```sql
drop table if exists hello_acid;
create table hello_acid (key int, value int)
partitioned by (load_date date)
clustered by(key) into 3 buckets
stored as orc tblproperties ('transactional'='true');
```

```sql
insert into hello_acid partition (load_date='2016-03-01') values (1, 1);
insert into hello_acid partition (load_date='2016-03-02') values (2, 2);
insert into hello_acid partition (load_date='2016-03-03') values (3, 3);
select * from hello_acid;
```

删除数据：
```sql
delete from hello_acid where key = 2;
select * from hello_acid;
```

更新数据：
```sql
update hello_acid set value = 10 where key = 3;
select * from hello_acid;
```
这些DML语句意在在大数据中执行少量修改，应该是记录级别的数据管理。如果你有小的多个批量修改，应该使用streaming data ingestion。

#### Streaming Data Ingestion
许多时候我们需要处理连续的实时数据流，想要更方便的操作这些数据。hive可以自动把流式数据加入hive表中，也支持实时的数据抽取与查询。

现在hive提供两种：
- 已经存在是[storm hive bolt](https://github.com/apache/storm/tree/master/external/storm-hive)方案，[flume hive sink](https://flume.apache.org/FlumeUserGuide.html#hive-sink)方案。这些工具对数据有较少的操作，重在转移数据
- 直接使用低级的[Streaming Ingest API](https://cwiki.apache.org/confluence/display/Hive/Streaming+Data+Ingest).

在使用streaming api之前，需要先创建好分区事务表。从查询角度来看，一切应该都是确定好的。

#### 4. 实践
插入几条数据对我们测试起来很简单，但是实际环境中，我们需要一次性处理几千或者几百万的数据。下面我们通过几个通用场景讨论一下怎样处理数据批次。

这些模型需要你建立一个主键。hive并不强制主键唯一，所以你必须在你app中控制一下。虽然hive2.1介绍了non-validating 外键，但是这东西目前还并没有被完整全面的验证过。
##### 4.1 searched updates
hive  ACID支持searched updates，这是一种常见的更新方式。注意，基于hive ACID的架构，更新必须通过批处理的方式执行。一次更新一行这种事情在实际使用中是行不通的。如果想通过某种方式一次性更新大量数据，那么searched updates就可以帮你了。

假设有一个维度表，包含的一个flag，指示当前记录是不是最新的值。这样我们就可以沿时间追踪维度变更情况。当维度表发生更新的时候，我们就把已经存在记录的设置成old。
```sql
drop table if exists mydim;
create table mydim (key int, name string, zip string, is_current boolean)
clustered by(key) into 3 buckets
stored as orc tblproperties ('transactional'='true');
```
```sql
insert into mydim values
  (1, 'bob',   '95136', true),
  (2, 'joe',   '70068', true),
  (3, 'steve', '22150', true);
```
```sql
drop table if exists updates_staging_table;
create table updates_staging_table (key int, newzip string);
insert into updates_staging_table values (1, 87102), (3, 45220);
```
执行更新,执行前后可以看一下维度表的数据变化，其实就是更新了flag而已。
```sql
update mydim set is_current=false
  where mydim.key in (select key from updates_staging_table);
```

##### 4.2 searched deletes
批量删除也可以使用staging table轻松搞定。但是需要你在表之间放置一个公共key，有些类似RDBMS中的主键。
```sql
delete from mydim
where mydim.key in (select key from updates_staging_table);
select * from mydim;
```

#### 5. 批量覆盖更新

有的时候我们需要批量更新一些数据。例如第一种SCD更新或者数据重导。hive目前还不支持merge操作，在支持之前，我们只能考虑使用先删除后插入的方式，但这样可能会造成查询客户端脏读。或者也可以在重新清洗的时候临时停掉查询。

#### 6. ACID工具
ACID事务执行过程中会创建一系列的锁。事务和他们的事务锁可以通过hive的一些工具来查看。
##### 6.1 查看事务
```
show transactions
```
##### 6.1 查看锁
有read, update，X lock。update锁与update操作互斥，但是与read兼容。X锁，与所有锁都互斥，属于独占。
```
show locks
```
##### 6.1 终止事务
注意这并不会马上kill掉所有相关查询。ACID查询是周期性执行的，默认是2.5分钟，如果他们检测到自己的事务被kill，就会执行自动退出。
```
abort transactions T1 T2 T3
```

#### 7. 性能考虑
- 创建分区
- insert快，update和delete都会比较慢一些，因为要扫描整个分区。
- 如果你们的工作里需要大量更新数据，那么请周期行执行compaction操作。不然数据大小会越来越大，查询也会越来越慢。

#### 8. 深入探究
在使用之前一定要深入了解这套系统的工作原理，并且在你的可以容忍丢失的数据上执行测试。过程中也注意备份数据。

ACID表有一个隐藏字段`row__id`。这个系统内置的字段名称有可能会变。你应该构建一个基于这个字段的长远的方案。

这个字段记录的内容有：
- 数据被insert或者update的active的事务id
- 数据所在bucket的bucketid
- 此次事务或者bucket中的rowid

看一下
```sql
hive> select row__id from hello_acid;
OK
{"transactionid":12,"bucketid":0,"rowid":0}
{"transactionid":10,"bucketid":1,"rowid":0}
```

常用场景就是确认所有的数据都load进来了。假设上游数据provider认为hive里的持久化数据丢失了，那么你的provider(storm Bolt比如)就会告诉你插入这些数据的事务ID。然后我们可以使用这个事务id计算一下实际的记录数。查询的时候使用X替换掉你的事务ID
```sql
set hive.optimize.ppd=false;
select count(*) from hello_acid where row__id.transactionid = X;
```

记牢，事务中插入的数据有可能会被后续的update或者delete语句影响到，所以如果count数并不符合，那就可能是这些因素造成的。




参考：

https://hortonworks.com/hadoop-tutorial/using-hive-acid-transactions-insert-update-delete-data/
