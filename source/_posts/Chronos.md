
title: Chronos
date: 2017-10-10 17:32:56
tags: [youdaonote]
---

Chronos 内部很简单：
- 从zk的state store中读取所有job的state
- job都在scheduler里注册过了，会别加载进job graph，以追踪依赖关系
- 根据host时间，job被分成不同的list中，有些现在就要跑，有些不用
- 需要跑的job会被放进队列，只要有充足的资源就马上运行
- 等再有job需要run的时候，就重复1


一个job只有在它的所有parent都运行完成过才能运行。它跑完之后，这个轮次就结束了。

这段代码在mainLoop()方法中。https://github.com/mesos/chronos/blob/be96c4540b331b08d9742442e82c4516b4eaee85/src/main/scala/org/apache/mesos/chronos/scheduler/jobs/JobScheduler.scala#L469-L498


其他特性
- 向cassandra写入job metrics，用来分析、估计等
- 发送不同的通知：email、slack等
- 导出metric到graphite或其他地方


不能做的
- 解决所有分布式问题
- 保证精准调度
- 保证时间同步
- 保证job真正运行了


实例架构
---
![](https://mesos.github.io/chronos/img/emr_use_case.png)



UI
---
- 增删改job
- 运行job
- 展示job依赖关系
- 已经执行完的job的统计量

安装
---
要是已经安装了DC/OS的话
```
dcos package install chronos
```

另外也可以使用docker启动，要保证两个端口资源：HTTP API， libprocess。

```
docker run --net=host -e PORT0=8080 -e PORT1=8081 mesosphere/chronos:v3.0.0 --zk_hosts 192.168.65.90:2181 --master zk://192.168.65.90:2181/mesos
```

源码构建
```
export MESOS_NATIVE_LIBRARY=/usr/local/lib/libmesos.so
git clone https://github.com/mesos/chronos.git
cd chronos
mvn package
java -cp target/chronos*.jar org.apache.mesos.chronos.scheduler.Main --master zk://localhost:2181/mesos --zk_hosts localhost:2181
```

运行
```
java -jar chronos.jar --master zk://127.0.0.1:2181/mesos --zk_hosts 127.0.0.1:2181
```
