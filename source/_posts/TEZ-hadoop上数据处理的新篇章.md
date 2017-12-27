
title: TEZ-hadoop上数据处理的新篇章
date: 2017-04-26 20:15:47
tags: [youdaonote]
---

TEZ
--
生成一个复杂的DAG task图

目前基本所有的hadoop任务都是MapReduce程序，面向批数据的特性使得它并不能够满足一些特定查询。TEZ是传统MR程序的另一个选择，可以更快地返回结果，而且减少过程中的吞吐。

动机
---
分布式处理是hadoop的核心。存储与分析各种不同的大量的数据使得hadoop之上产生了很多工具。随着hadoop进入yarn时代，它把MR解耦了出来，这样可以使用其他的数据处理方式来迎接新的不同的挑战。

设计
---
hive,pig这些高层次的数据处理工具需要一个引擎来解析他们的语句，然后执行。TEZ就是这样一个引擎。

#### 解析，建模，执行
TEZ把数据处理看作一个数据流图，节点代表数据处理逻辑，边代表着数据的流向。TEZ有一个很好的数据流相关API,能够让用户清晰表达出自己的复杂查询逻辑，也能够支持hive、pig等高级应用生成的查询计划。

如下图，使用range partitioning进行分布式sort的模型。Preprocessor stage中发送samples给Sampler，计算每个数据分区的range，以保证平均分配。然后这些range再被发送给Partition stage。然后partition stage和aggregate stage就读取分配的stage，执行数据的扫描与后续聚合工作。

![](https://2xbbhjxc6wk3v21p62t8n4d4-wpengine.netdna-ssl.com/wp-content/uploads/2013/09/tez1-450x257.png)
#### 灵活的input-processor-output任务模型
TEZ中对于数据流中每个处理逻辑都是由三部分组成：input, processor, output。input和output决定数据接口。processor则是数据处理逻辑。TEZ只要求在同一个vertex task中这三部分之间是相互兼容的即可。

![](https://2xbbhjxc6wk3v21p62t8n4d4-wpengine.netdna-ssl.com/wp-content/uploads/2013/09/tez2-450x154.png)

#### 动态图配置的性能

分布式数据处理天生就是动态的，所以很难提前知道最优并发和数据走向。更多的信息只能在运行时才获知，比如数据的内容和大小，这其实是可以帮助我们优化执行计划的。我们还注意到TEZ自身并不能总是自己执行这些动态优化。

![](https://2xbbhjxc6wk3v21p62t8n4d4-wpengine.netdna-ssl.com/wp-content/uploads/2013/09/tez3-450x208.png)

- https://hortonworks.com/blog/apache-tez-a-new-chapter-in-hadoop-data-processing/
