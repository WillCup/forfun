
title: hadoop-vs-mpp
date: 2017-10-09 11:45:18
tags: [youdaonote]
---



什么是MPP
---
MPP全名是massively parallel processing，是网格计算的一种实现，网格中的所有节点都参与计算过程。MPP DBMS是基于MPP的DBMS。每个查询都需要先切分计算任务，这会比传统的SMP RDBMS更快一些。另外，MPP还具有扩展性，我们可以方便的向网格中添加节点来扩展计算与存储能力。为了处理大量数据，需要对数据进行在节点之间进行shard，这样每个节点都只处理自己本地的数据。这进一步提升了数据处理速度，因为使用共享存储的话会出现一个很大的问题—— 更复杂，更昂贵，不易扩展，更高的网络IO，更小的并行度。这就是大多数MPP DBMS方案share-nothing不共享任何东西，运行在DAS（direct-attached storage）或者小组server共享机架存储的原因。Teradata、Greenplum、Vertica、Netezza等都是使用的这种方式。这些工具都有一个复杂且成熟，专为MPP方案订制的SQL optimizer。而且他们都是可扩展的，围绕这些方案有很多的工具包来支持用户的需求：地理位置分析、全文检索等。他们都是闭源的复杂的企业级解决方案(Greenplum在2015年开源)。

![](https://0x0fff.com/wp-content/uploads/2015/07/MPP_arch.gif)

hadoop
---

那么hadoop呢？不是一个单一的技术，而是一个生态。基友好处也有坏处，最大的优点就是延展性好 - 催生了一些列的新组件（spark），而且这些组件的核心都与hadoop息息相关，这就给集群的扩展性很多可能性。一个缺点是弄一个整套的生态是比较费事儿的，现在没人手工弄，都是借助工具了。


hadoop的存储技术是一个完全不同的方式。并不是根据key去把数据进行shard，而是把数据弄成指定大小的block块进行切分，以block为单位存储在不同节点上。这些数据块一般是比较大且只读的，整个文件系统也是只读的。简单来说，对于加载100行数据表，MPP引擎会基于某个key把数据进行shard，对于较大的集群来说这种方式可能会出现每个node上只有一条记录的现象。而如果使用HDFS的话，这个小的表会被写在一个数据块中，对于datanode的文件系统来说只是一个普通的 文件而已。

资源管理呢？不同于MPP方案，hadoop的资源管理器具有更细粒度。MR任务不需要所有的计算任务都并行执行，如果集群资源满载的时候，我们甚至可以把某个任务的数据和所有计算任务都丢到一个节点上执行。还有一系列的其他延展性，支持长时间运行的container等..但是事实上，它比MPP的资源管理器会慢一些，而且对于并发管理表现也稍逊一筹。

可选
---

对于Hadoop的SQL接口工具，有很多工具可以选择：基于MR/Tez/Spark的hive、sparksql、impala、HAQW或者IBM BigSQL、或者完全不同的Splice Machine。因为有太多工具可选，也很容易迷失。


#### hive
把sql转化为MR/Tez/Spark任务，在集群上执行。所有的这些任务类型其实都是基于MR思想，可以很好的利用集群，也方便与其他的hadoop技术栈进行整合。缺点也很明显 —— 执行查询的延迟较大，对于表之间的join操作性能堪忧，没有查询优化器引擎只按照你说的方式进行执行。下图覆盖了废弃的MR1的设计图：
![](https://0x0fff.com/wp-content/uploads/2015/07/HiveArchitecture.jpg)

#### MPP
类似impala和HAWQ的解决方案则与hive相对，使用了基于HDFS的MPP执行引擎。他们对于查询有更小的延迟、更少的处理时间，但是牺牲了一些扩展性和稳定性。

![](https://0x0fff.com/wp-content/uploads/2015/07/impala.png)


#### SparkSQL

sparkSQL是一个介于MR和MPP-over-hadoop之间的解决方案。与MR类似，它把job切分为一组task进行分别调度，这样来保证比较好的稳定性。又类似于MPP，试图把各个执行的stage之间流式串联起来来提高处理速度。他还使用类似于MPP的固定的executor来降低查询延迟。它继承了优点，同样缺点也跟了过来 —— 不及MPP快，不及MR稳定。



总结
---

指标 | MPP | Hadoop
--- | --- | ---
平台开放性 | 闭塞且专利。对于一些技术甚至连文档都是闭源的 | 完全开源
硬件选择 | 很多都是必须依托服务方的，很少能部署在自己的集群上。所有的解决方案都需要企业级的硬件，比如快速硬盘，大ECC RAM的服务器，无线带宽等 | 任何硬件都可用。最推荐的就是使用廉价的带有DAS的硬件【就是磁盘单挂】
节点水平扩展  | 平均几十台，最多100-200台 | 平均100台，最多几千台
用户数据扩展性 | 几十TB，最多PB级别 | 几百TB, 最多几十PB
查询延迟 | 10-20 milliseconds | 10-20 seconds
平均查询时间 | 5-7 seconds | 	10-15 minutes
最大查询时间 | 1-2 hours | 1-2 weeks
查询优化 | 复杂的企业级查询优化引擎使用全局最优的方式 | 没有optimizer，或者有限制，有时甚至都不支持cost-based
查询debug与profile | 有查询计划与查询静态量，提供错误信息 | OON问题和java堆溢出分析，GC停止信息，分离的任务日志
技术代价 | 每个节点几千或者几万美元 | 每个节点最多几千美元
终端用户的访问 | SQL接口和一些简单的数据库函数 | SQL并不完全兼容。用户需要关注执行逻辑，数据的分布。函数一般需要用java进行编写，放到集群中
目标用户  | 商业分析师 | java dev和经验丰富的DBA
单job冗余度 | 低，MPP节点失败，则job失败 | 高，job失败后有重试
目标系统 | 一般的DWH和分析系统 | 为特定目标构建的数据处理引擎
vendor lock-in | typical case | 	Rare case usually caused by technology misuse
推荐的最小的数据大小 | 任何 | GB
最大并发 | 几十到几百个query | 10-20个job
技术延展性 | 只能使用厂家提供的工具 | 可以随意组合开源工具
需要的DBA技术级别 | 一般水平 | 精通java和RDBMS背景
方案实现服务性 | 中 | 高


有了以上信息，我们知道为什么hadoop不能完全取代传统企业级数据仓库，但是他是可以用作处理海量数据的一种分布式引擎。facebook有一个300PB的hadoop集群，它还额外用了50TB的Vertica集群。Linkedin有一个巨大的hadoop集群，也同样有一个Aster数据中心（Teradata提供的MPP工具）。


参考：
https://0x0fff.com/hadoop-vs-mpp/
