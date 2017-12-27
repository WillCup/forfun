
title: Flink特性
date: 2017-11-22 14:13:34
tags: [youdaonote]
---

stateful operator可以追溯到前面的状态，例如count操作


stateful
- 聚合操作
- complex event processing
- ML pattern


stateless
- 数据抽取
- 数据清洗
- stateless的转化


#### queryable state
如果设定了一个小时的窗口，在到达一小时之前这个数据一般是不可查询的。但是有了 queryable 是state，我们可以在未到时间窗口的时候，就进行查询计算。


#### 不想中断数据处理，但又想
- 修改worker的数量
- 迁移到另一个集群
- 修复代码bug
- 升级flink
- 测试不同的算法
- ......

以上这些对于stateless任务其实是很简单的，直接操作就可以。停掉，重启。对于statefule的任务，就比较棘手。对于添加worker的需求，state需要reload或者redistribute。

Flink方案：savepoint。
- 创建savepoint
- 修改
- 从savepoint重启

对于session window的state也是很棘手的，Flink1.1会支持这个场景。
