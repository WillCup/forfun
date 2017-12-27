
title: presto架构
date: 2017-10-09 17:18:53
tags: [youdaonote]
---

server 类型
---

#### Coordinator

#### Worker


数据源
---
#### Connector

用来适配presto到类似hive的RDBMS。可以把connector当成database的driver。是Presto的SPI的一个实现，允许presto使用标准API与资源进行交互。

Presto包含几个内置的connectors：JMX, HIVE, System connector用来提供访问内部系统表的访问方式， TPCH connector用来提供TPC-H benchmark数据。还有一些第三方的connector。


每个catalog是与一个指定的connector进行协作的。如果你看过catalog的配置文件，你会发现每个都必须包含一个connector.name属性，catalog manager用这个来为指定catalog创建对应的connector。可以多个catalog共用同一个connector来连接两个相同数据库的不同实例。例如，假设我们有两个hive集群，我们就配置两个catalog实例，都用hive connector，这样甚至能在同一个SQL查询中查询到两个hive集群中的数据。



#### catalog

一个catalog包含多个数据库schema和通过一个connector关联的数据源。在运行sql语句的时候，可以在一个或多个catalog中进行数据查询。

在presto中，数据表的全名是catalog的根部。例如hive.test.data.test就是hive catalog中的test_data数据库中的test表。

catalog是通过presto配置目录中的properties文件定义的。

#### schema
schema是用来组织table的。catalog和schema用来定义一组可供查询的数据表。在prestor查询hive或者mysql等关系型数据库的时候，schema就对应于数据库名。其他类型的connector也可以根据具体场景进行对应

#### Table
未排序的数据行的组合，每个列都有自己的类型。与关系型数据库定义一样。

查询模型
---

#### Statement
Presto执行ANSI兼容的SQL语句。当一个statement被执行的时候，presto创建一个query和一个query plan，然后这个plan被分布到presto worker中执行。

#### Query
Presto解析statement的时候，把statement转化成一个query，创建一个分布式的query plan，然后形成一系列相互关联的stage，运行到不同的presto worker上。当我们抽取一个查询的信息的时候，我们会获取到对应于这个satement的每个组件的状态快照。

statement和query的区别很简单，statement可以看作传递给presto的SQL语句，一个query则对应于执行这个statement的配置与实例化的组件。一个query包含了stage、task、split、connector、数据源以及其他组件。


#### Stage

presto执行query时，会把query plan打成一系列有层级的stage。例如，如果presto需要聚合hive里的一亿行数据，他需要创建一个root stage来对其他stage的output执行聚合操作，这些其他stage负责query plan中的不同部分。

这些stage可以使用tree来表示。每个query都有一个root stage来执行其他stage的聚合操作。coordinator 是coordinator用来模型化分布式query plan的，但是stage本身并不运行在presto worker上。


#### Task

前面提到，stage只是模型化分布式查询计划的一个特定部分，而不在presto worker上执行。一个stage其实又被一系列task实现，分布式运行在presto worker上。

presto task是有多个input和output的，可以与一系列driver并行执行。

#### Split

stage是通过connector从数据源以split为单位抽取数据的，然后中间的stage或高层stage又从其他stage中抽取数据。

prestor执行query时，coordinator会从一个connector查到指定table的所有的split。coordinator追踪哪些机器在运行哪些task，哪些split在被哪些task处理。


#### Driver
task包含一个或多个并行的driver，一个driver只有一个input和output。dirver整合operator来生成output，这个output会被task聚合，再传递给其他stage的一个task。一个driver是一个operator实例的序列，可以把他看成内存中的一系列operator集合。它是presto并发执行的最低层单元了。


#### Operator
operator消费、转换、生产数据。例如，一个table scan从connector中抽取数据，然后生产可以被其他operator消费的数据。一个filter operator消费数据，生产已经过滤掉的一个数据子集。

#### Exchange

Exchange负责在Presto节点间传递同一个query的不同stage的数据。task把数据生成进一个output buffer，并使用exchange client消费其他task生成的数据。
