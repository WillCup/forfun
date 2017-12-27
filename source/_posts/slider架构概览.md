
title: slider架构概览
date: 2017-12-05 16:03:33
tags: [youdaonote]
---

#### 概述
slider是一个YARN application，用来部署非YARN的app到YARN集群中。

slider包含一个YARN application master，Slider AM，然后client application需要通过RPC或者HTTP与YARN和Slider AM进行通信。client application提供了命令行与low-level API，以供测试。

被部署的app应该是可以运行在YARN管理的server池中的程序，还要能动态定位到它自己的peer。slider并不负责配置这些peer server，只负责一些app指定的实例的配置初始化工作。

每个app实例都是一个或多个component，每个component可以有不同的程序或命令，不同的配置项或参数。


AM负责启动哪些具体的role，为每个component请求一个YARN container。然后监控app实例的状态，当一个远程执行的进程完成之后，YARN会通知到AM。然后AM就去部署这个component的下一个实例去了。

#### slider打包
slider一个重要的目标就是支持已经存在的APP部署到yarn上去

#### AM架构

AM包含：
 - AM engine，负责处理所有的外部服务的整合，尤其是YARN和其他的slider client
 - provider，指定部署app的class
 - app的状态
 
app状态是app实例的模型，包含：
 - app实例一些期望的状态，比如每种component的实例数量，他们的YARN container内存等
 - 当前实例在yarn集群中每种component的数据，包括每个node上的可用资源
 - role history。记录每个node都被部署过哪种component，后面可以还这样部署。这是为了在需要读写本地磁盘的时候不出问题。
 - 追踪消息队列：请求、发布、启动节点
 
app engine整合了所有外部的东西：YARN RM， 指定node的NM，接受来自RM的service，request，释放container的event，在分配的container上启动app。

集群中发生任何变化，都会发送通知，然后app engine把通知传递给App的Status类，Status类更新它的状态，返回一系列集群操作(请求不同类型的container， 可能会指定节点，或者请求释放container)，供提交。

有了这些之后，再加上分配消息，app engine就可以把app State分配给指定的组件了，然后出发provider构建app的启动context。

provider有带有用于启动provider支持的程序的文件关联、环境变量、命令，用以发布container。

core provider在目标container上部署一个minimal 的agent，然后这个agent会计入agent provider的REST API， 执行它的命令。

这个agnet要执行的命令主要是从HDFS下载归档文件，解压开，运行python脚本来执行真正的配置，最后就是目标的执行了。这里面会用到很多模板。

总结一下：slider不是一个典型的YARN analysis app，YARN analysis app是指在短期、中期生命的container中分配和调度工作，带有一个query或者analysis session的一个生命周期，slider的生命周期是长达几天或者几个月的。slider要是app集群保持某个状态，app也应该能在node失败后进行恢复，还有就是定位到自己的peer node，再有就是与HDFS文件系统的数据进行交互。

Samza是第一个被设计运行在yarn上的app，它作为一个平台或者long-live 服务存在。这些app对yarn的需求是不一样的，他们的application master的设计主要集中在维护分布式app在一个稳定的状态，而不是提交什么作业。

MVC的切分实现了mode与aid mock测试的隔离性，增强了slider能够在YARN上部署更大规模的信息。


#### 失败模型
app master被设计为一个[ crash-only application](https://www.usenix.org/legacy/events/hotos03/tech/full_papers/candea/candea.pdf), client是随时可以直接请求YARN来终止app实例的。

有一个RPC方法可以停掉app实例。这个是挺好的，会记录一条message在日志里，以后可能还会给provider警告一下这个app实例要被关掉了。这也有一定潜在的危险，以你为provider实现里可能开始期望这个方法被可靠调用。slider的设计是失败不会告警，而是基于配置进行重新构建，可以被人工停止。

#### RPC 接口

RPC接口允许client查询当前app的状态，也能通过json请求进行更新。

主要操作有：
- getJSONClusterStatus(): 获取app实例的json状态输出
- flexCluster(): 更新调整不同component的数量
- stopCluster 

还有一些其他更low-level的操作供我们进行诊断、测试，但是比较有限。




