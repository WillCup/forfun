
title: hive2-LLAP
date: 2017-11-27 14:59:23
tags: [youdaonote]
---

Live Long And Process。


#### 概览
最近几年，有了tez和CBO等机制，hive的运行速度与特性已经有了很大提升。这次2.0又把hive带入了一个新的level

- 异步spindle-aware的IO
- 预抽取column chunk，缓存column chunk
- 多线程JIT友好的operator pipeline


上面的特性就是LLAP提供的了，LLAP提供了一个混合引擎模型。它包含了一个长期存在的daemon，替代直接与HDFS的Datanode交互。还有一个与DAG紧密结合的framework。对于缓存、字段pre-fetching、query的执行与ACL都被移动到了daemon中。small/short的query大部分都会被送到这个daemon直接处理，其他稍微重一些的操作都会被YARN的container运行。

跟DataNode类似，LLAP deamon也可以被其他的app使用，尤其是基于文件处理的关系型数据view。这个daemon也提供了可选的API(例如InputFormat)，供其它数据处理框架作为一个构建的block。


最后，细粒度的字段级别的acl在这个model中也得到了完美的呈现。

下图显示了LLAP的一个执行样例。Tez AM负责整体的编排工作。query的初始化stage被push给了LLAP。reduce stage，大的shuffle操作是在几个container中执行的。多个查询和app可以并发访问LLAP。

![](https://cwiki.apache.org/confluence/download/attachments/62689557/LLAP_diagram.png?version=1&modificationDate=1474327021000&api=v2)


#### 持久化daemon
为了进一步优化缓存和JIT，还有能够更好的估算startup耗时等，集群中的worker节点上都会有一个daemon。这个daemon处理IO,缓存，query分片的执行。
- 所有的node都是无状态的。任何发送给LLAP节点的request都包含数据的位置和元信息。可以是本地的位置，也可以是远程的，数据本地化是调用者需要考虑的事情(YARN)
- recovery/resilency。失败和恢复都很简单，因为任何数据家电都可以用来处理输入数据的任何分片。Tez AM只需要简单的重新运行失败的分片即可。
- 节点间沟通。LLAP节点之间可以共享数据(例如抽取partition数据，广播数据分片等)。实现方式与Tez中一样的。


##### 执行引擎
LLAP和已经存在的，基于过程的hive引擎是可以兼容的，保留了hive的可扩展性和广泛性。它并没有取代已经存在的执行模型，而是对其进行增强。
- daemon是可选的。在没有这些daemon的时候，hive仍然是可运行的。
- 外部编排和执行引擎。LLAP并不是一个像Tez或者MR的执行引擎。所有的执行都是被已经存在的hive执行引擎（比如tez）在LLAP节点上进行调度与监控的，跟普通的container一样。显然，LLAP级别的支持是基于每个执行引擎(目前是tez)的。MR的支持并没有计划，但是其他的引擎有可能会在后面也加入进来。其他的框架，例如pig，也可以选择使用LLAP daemon。
- 部分执行。LLAP daemon的执行结果可以说某个hive query的结果的一部分，也可以传递给外部的hive task，这个取决于具体的query
- 资源管理。YARN仍然负责资源的管理与分发。yarn container delegation被用来允许分配资源给LLAP。为了避免jvm内存设置的初始化，缓存的数据是放在off-heap的，大的buffer也是(比如groupby，join等操作)。通过这种方案，daemon可以只是用很小的内存，其他资源(CPU,内存等)会根据负载进行分发。


#### query fragment执行
LLAP节点会执行一些query分片，比如filter，projection，数据转化，部分聚合，排序，分桶，hash join/semi-join等。在LLAP中只接受hive代码和udf。没有任何代码是可以在运行时生成和执行的。这是出于稳定性和安全性的考虑。
- 并发执行。一个LLAP节点允许不同query和session 的多个查询分片的并发执行。
- interface。用户可以通过client API直接访问LLAP节点。他们可以指定关系转化，然后通过面向record的stream进行数据读取。

#### IO

daemon摆脱了压缩格式处理的IO，这些工作交给其他独立的线程。数据在就绪的时候会被传递给execution，就是说前一批数据正在被处理的同时，就可以准备下一批了。数据是以简单的RLE编码的列式格式传递给execution的，主要是为了后面方便进行vectorized processiong【向量化处理？】。这也是缓存的格式，可以减小IO，缓存，execution之间的copy流量。
- 多种文件格式。IO和缓存依赖于已经存在的文件格式(越高效越好)。因此，和vectorization工作类似，不同的文件格式都会通过插件形式被支持(目前是ORC)。另外，一个通用的，不那么搞笑的插件也可以加进来支持任意的hive输入格式。这些插件必须负责持有元信息，并且把原始数据转化到column chunk。
- 预测和bloom filter。SARGs和bloom filter是会被push down到存储层的。


#### 缓存

daemon会缓存输入文件和数据的元数据。即便数据还没有缓存，元数据和索引信息也是可以先缓存起来的。元数据是以java object的形式存在进程的，缓存的数据是根据[IO部分](https://cwiki.apache.org/confluence/display/Hive/LLAP#LLAP-I/O)的指定形式，保存在off-heap的。
- 驱逐策略【eviction policy】。取出策略是为了调优负载，应对频繁的表扫面的。初始阶段，是使用了LRFU的简单策略。这个策略是可插拔的。
- 缓存粒度。column-chunk是缓存数据的基本单位。这是在低开销的处理与高效存储之间的一个权衡。chunk的粒度取决于文件格式和执行引擎了。(Vectorized Row Batch Size, ORC stripe等)

bloomfilter会自动创建，以提供动态运行时过滤。

#### workload管理
YARN用来获取不同workload的资源。某个workload一旦获取到yarn分配的资源之后，这个执行引擎就可以选择把资源代理给LLAP，或者在单独的进程里启动hive executor。通过YARN的资源管理保证了节点不会过载运行，不管是因为LLAP或者其他container。daemon本身也是在YARN的控制之下的。

#### ACID支持
LLAP是事务感知的。在数据放入缓存之前，将delta文件合并，可以生成一个table的特定的state。如果有多版本，可以在请求中指定使用哪个版本。这样的好处是可以异步merge，而且只用cache一次数据，从而避免了operator pipeline的攻击。

#### 安全

LLAP server是一个很自然的就可以进行acl的地方，比per-file的控制粒度还要细。因为LLAP的daemon知道当前要处理的column和record，所有针对这俩东西的策略都可以在这里实施。这并不会替代已有的安全机制，但是会有所增强，也可以提供给其他框架使用。

#### 监控

LLAP监控的配置是存储在resources.json， appConfig.json， metainfo.xml，都会存在于slider的templates.py中。

LLAP monitor daemon也运行在YARN container中，跟LLAP Daemon一样，而且默认监听同样的端口。

LLAP Metric Collection Server从所有的LLAP Daemon周期性地收集JMX metric

LLAP daemon列表是在启动集群的zookeeper中抽取的。

#### web service

- json jmx数据 - /jmx
- jvm stack trace - /stacks
- LLAP daemon的xml配置 - /conf
- LLAP status - /status
- LLAP Peers - /peers

#### 在slider上部署

LLAP可以通过slider进行部署，绕过了节点安装与相关复杂处理

#### LLAP status

[ambari相关](https://issues.apache.org/jira/browse/AMBARI-16149)介绍了LLAP ap的状态，在hiveserver2中可用了。

例子
```
/current/hive-server2-hive2/bin/hive --service llapstatus --name {llap_app_name} [-f] [-w] [-i] [-t]
-f,--findAppTimeout <findAppTimeout>                 Amount of time(s) that the tool will sleep to wait for the YARN application to start. negative values=wait
                                                     forever, 0=Do not wait. default=20s
-H,--help                                            Print help information
   --hiveconf <property=value>                       Use value for given property. Overridden by explicit parameters
-i,--refreshInterval <refreshInterval>               Amount of time in seconds to wait until subsequent status checks in watch mode. Valid only for watch mode.
                                                     (Default 1s)
-n,--name <name>                                     LLAP cluster name
-o,--outputFile <outputFile>                         File to which output should be written (Default stdout)
-r,--runningNodesThreshold <runningNodesThreshold>   When watch mode is enabled (-w), wait until the specified threshold of nodes are running (Default 1.0
                                                     which means 100% nodes are running)
-t,--watchTimeout <watchTimeout>                     Exit watch mode if the desired state is not attained until the specified timeout. (Default 300s)
-w,--watch                                           Watch mode waits until all LLAP daemons are running or subset of the nodes are running (threshold can be
                                                     specified via -r option) (Default wait until all nodes are running)
```

