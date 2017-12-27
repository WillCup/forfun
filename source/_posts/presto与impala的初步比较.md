
title: presto与impala的初步比较
date: 2017-12-26 15:38:34
tags: [youdaonote]
---

特性 | presto | impala
--- | --- | ---
sql兼容 | 兼容mysql标准函数与类型，分析师只需掌握mysql语法函数即可 |  高度兼容hive
部署难度 | 较低 | 高
规模 | 动态调整 | 每个DataNode上都要一个impala实例
容错 | 多个worker挂掉不影响 | 多个impalad实例挂掉会有较大影响
性能 | 高，秒级  | 很高，秒级
语言 | java  | c++
容量 | 易于扩缩容，不影响既有应用  | 完全取决于datanode
问题解决 | 基于java,，自主patch或者熟悉原理的成本低 | 基于c++，自主patch或者熟悉原理的成本较高
社区 | 国内各种技术会议议题点击率较高【美团、头条、滴滴、链家等等】 | 国内使用的目前耳闻几乎没有
hive兼容性 | 对于基于异构存储的hive表不能兼容 | 自己维护额外的元数据表，所有的hive表变更都需要额外的命令操作，极不方便
数据源兼容性 | 高，自定义connector | 低，只兼容HDFS之上


我们的现状
- 人力资源少，可投入专项维护的人员几乎没有
- 需求方接受查询在分钟级别的体验，期望更快的体验
- 需尽快响应使用中出现的问题

基于以上，两者的性能差距对我们的影响较小，执行时间差别感知较小。而其他维护性的工作【扩缩容、问题排查等】对于当前数据组的有限精力来说更为看重。所以选型**暂定为persto**。

其实最主要的就是： **出问题的话，是几度懵逼！！**


参考 :

- https://www.quora.com/How-does-Cloudera-Impala-compare-to-Facebooks-Presto
- http://blog.cloudera.com/blog/2014/09/new-benchmarks-for-sql-on-hadoop-impala-1-4-widens-the-performance-gap/ 【这个是impala的爸爸CDH的文章，准确性有待探测】
- https://groups.google.com/forum/#!topic/presto-users/NA42x4B0H-o

