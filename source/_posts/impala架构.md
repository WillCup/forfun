
title: impala架构
date: 2017-10-09 15:52:50
tags: [youdaonote]
---

impala Daemon
---
核心的impala组件是impala daemon，它需要在每个datanode上部署一个，进程名是impalad。它负责读写数据文件、接收来自impala-shell命令行、hue、JDBC或者ODBC的查询；并行执行查询、为集群分配分布式任务、再把中间查询结果发送回coordinator node。

可以通过任何一个datanode上的impala daemon提交查询，然后这个node上的impala daemon就作为这个查询的coordinator node。其他node把部分结果回传给coordinator node，coordinator node负责把这些结果进行组织，并产生最终结果。当通过impala-shell命令运行查询的时候，我们可能需要一直连接着同一个impala daemon。对于集群生产环境，我们需要考虑负载均衡，将查询放到不同的node上去提交。

impala daemon需要与statestore持续联系，以确认哪些node是健康的，可以接受任务。

impala daemon还要接收catalogd的广播信息，获取集群中哪些impala node新建、修改、drop一些类型的对象、或insert或者load data语句的执行。这个动作最小化了REFRESH和INVALIDATE METADATA语句的执行次数，这两个动作在1.2之前必须要协调所有node的元数据。


在2.9或更高版本中，我们可以控制哪些node可以用作query coordinator，那些是query executor，以此提升扩展性。



impala statestore
---
statestore负责检查所有 datanode上的impala daemon的健康状态，同时也通知每个impala daemon它的新发现。他是一个独立的进程，statestored，需要集群中的某个机器上有一个就可以。如果一个impala daemon掉线的话，statestore会通知所有其他的impala daemon，这要后面的查询任务就不会分配到这个掉线的node上。

因为statestore的职责是处理问题情况，所以它并不是impala 集群执行正常任务的核心所在。如果statestore挂掉，impala daemon会像往常一样继续运行、分配任务、执行任务。只是集群会在某些impala daemon挂掉的时候变得不那么强壮。当statestore恢复之后，它会重建与impala daemon的联系。


多数既有的HA和LB方案都可以用于impala daemon。而statestored和catalogd进程就不需要，因为它们挂掉并不会造成数据丢失。如果这些进程变为不可用的话，可以直接停掉impala服务，删掉基友的impala statestore和impala catalog server，然后把它们分配到其他的node上，重启impala service就可以了。



impala catalog服务
---
catalog服务负责将impala SQL语句导致的源数据变化传递给集群中的所有的Datanode节点，进程为catalogd，整个集群只需要一个。因为查询请求会通过statestore daemon，最好将statestored和catalogd两个服务运行在同一个host上。


catalog服务避免了由元数据变化引起的REFRESH和INVALIDATE METADATA操作。当我们通过hive创建表、加载数据的时候，我们必须要执行REFRESH和INVALIDATE METADATA操作才能执行数据查询。

这个特性有关于impala的以下几个方面：
- 见安装impala、升级impala
- 通过impala执行CREATE TABLE, INSERT或者其他修改表、修改数据的操作并不需要REFRESH和INVALIDATE METADATA操作。但是如果是通过hive或者手动加载HDFS文件方式的话，就需要。

默认地，加载元数据和加载缓存是异步的，所以impala可以立刻接收请求。可以通过参数配置load_catalog_in_background=false.。

impala对于hadoop生态的兼容与适配
---
#### hive
impala维护了table定义，也就是metastore，其实就是hive 的metastore的数据库。impala还要追踪对应的数据文件：HDFS中数据的block的具体位置。

对于数据量特别大、分区数特别多的表来说，单抽取元数据就可能花掉数分钟的时间。因此，每个impala  node都会把这些信息缓存起来以供后面重用。

如果某个表的定义改变或者数据被更新，那么其他所有的impala daemon都要在对这个表执行查询之前同步最新的metadata，替换掉老的那份缓存。1.2版本之后，这个过程是自动的通过catalogd进程完成。

对于未通过impala操作的修改，则需要执行REFRESH（已经存在的表加入新的数据文件或更新）或者INVALIDATE METADATA(创建新标、删除表、执行了HDFS rebalance、 删除了数据文件等)。INVALIDATE METATDATA会重新抽取所有的表的metadata。所有如果要是知道哪个表有变化的话，可以使用REFRESH table_name来执行只拉取这个表的相关元数据。

#### HDFS
impala使用HDFS作为主要的存储介质，利用HDFS的冗余来保证数据不丢失。impala表数据物理上就是HDFS的数据文件，可以是HDFS支持的任何格式或压缩格式。当数据文件是放在一个目录作为一个表存在是，impala会把这个目录下所有的文件都作为表的内容进行读取。通过impala新增的数据文件，则会由impala进行命名。

#### HBase
HDFS的替代方案。需要在impala中额外定义对应表。

并发控制
---
通过准入控制来闲置资源的方式之一是限制一个最大并发查询数量。在我们没有内存使用情况的足够信息的时候这是一个比较初步的方式。这个参数可以分别给每个动态资源池进行设置。

这个配置也可以结合下面章节中内存控制的配置一起使用。不管是达到了内存最大使用量，或者超过了最大并发查询数，后续的查询都会被放到队列里，等待资源。


内存控制
---
每个动态资源池都可以配置一个最大内存使用量。最好是在已经明确知道工作量与使用大小的时候使用这个配置。

每个host上都要指定一个默认的最大内存Default Query
Memory Limit，相当于在这个资源池中为每个query指定一个MEM_LIMIT参数。

另外，也可以根据每个node的最大内存计算出一个集群级别的最大内存限制，来控制最多并行查询数。

例如下面场景：
-  五个datanode上运行了impalad
-  设置一个动态资源池的最大内存为100G
-  host级别的最大内存Default Query
Memory Limit设置为10G，那么每个查询最多使用 5 * 10 = 50G
-  在这个动态资源池中可以并行运行的查询数为2，100 / 50 = 2.
-  如果查询并没有用满host或者集群级别的内存上限额度，并不会有什么惩罚措施。这些上限值只是用来估算资源池里可以并行多少查询。


注意： 如果你为某个impala动态资源池指定了集群级别的最大内存，就必须指定host级别的最大内存Default Query
Memory Limit。


与其他资源管理工具的关系
---
admission control是轻量级、去中心化的。他设置的是一个soft limit，并不是像yarn把资源进行all-or-nothing方式处理。

因为admission control并不与其他hadoop组件(MR)交互，我们可以使用yarn的部分静态资源池，与其他hadoop应用共享yarn。要使用impala多租户的时候推荐使用此种配置。admission control控制查询并发和内存上限，yarn来管理其它的组件。这种情况中，impala的资源并不是由yarn管理的。

impala的admission control可以使用yarn的user与pool的认证关系。


虽然impala admission control使用了fair-scheduelr.xml配置文件，但是这个文件并不关心YARN实际使用哪个scheduler，也就是说它仍可以使用capacity scheduler。



