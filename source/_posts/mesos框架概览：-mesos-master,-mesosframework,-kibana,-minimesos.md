
title: mesos框架概览：-mesos-master,-mesosframework,-kibana,-minimesos
date: 2017-10-10 11:03:07
tags: [youdaonote]
---

提纲
---
- 使用[mesos-starter](http://www.github.com/ContainerSolutions/Mesos-Starter)做的自定义framework
- 使用[mesos framework](http://www.github.com/ContainerSolutions/MesosFramework)做的一个通用framework
- 一个为[kibana](https://github.com/mesos/kibana)定制的mesos framework
- 使用[minimesos](http://minimesos.org/)进行测试
- 


mesos-starter
---

这是一个spring工具，[简单参考(http://container-solutions.com/mesos-starter/)。下面是一些特性
- 暴露一个bean来控制实例数量(提供水平扩展能力)
- 管理端口。用户可以请求静态的、或者mesos制定的端口，并且把它们用作环境变量
- 配置docker网络类型
- mesos鉴权
- 用户可以指定任务的sandbox里下载二进制包以及其他附件资源的的URI

这些新特性可以支撑framework更高的灵活性。例如，我们若是在minimesos中使用docker containerizer，就必须使用dcoker bridge模式来保护host上的端口不会冲突(所有的agent都使用同一个docker daemon)。但是在实际情况中，我们需要docker 运行在host模式，这样它们就可以直接使用主机ip了。


mesosFramework
----
MesosFramework是一个分布式的基于spring的二进制包，或者基于mesos-starter的docker容器。用户可以使用一个简单的配置文件快速创建一个可以运行在mesos上的framework，只需要告诉MesosFramework它应该运行什么任务就可以了。MesosFramework很通用，用户可以完全控制deployment。


我们讨论过[为什么会想自己写一个mesos framwork](http://container-solutions.com/reasons-use-apache-mesos-frameworks/)，希望你考虑使用Mesos-starter.许多简单的app都可以通过一个类似marathon的orchestration tool来启动。但是有些用例是介于这两个极端之间的，例如
- 既想利用mesos-starter一些好的特性，又不想编写任何代码(authorisation， 一个orchestraion strategy等)
- 要打包一个特定的framework配置，这样用户就不用关心配置了（还是没有代码）
- 为某framework快速部署一个POC


对我们来说，有很多frameworkd都落入了这个灰色地带。主要的好处就是减少了代码编写，也就减少了维护成本。

MesosFramework还添加了一些Mesos-Starter之外的特性。比如：查看task状态的REST接口、水平扩容等。


Kibana framework
---

新的[Kibana framework](https://github.com/mesos/kibana)就是基于MesosFramework和Mesos-starter构建的。它继承了这两个项目的所有特性。也就是说对于这两个项目的更新与特性发布会自动包含进像kibana这种下游项目中。这就是天堂，不用管理代码就能获得最新的特性。

但是最大的特点其实是：没有代码！！同样的重构也会应用到Logstash framework中。很多mesos framework都有对于代码的过度引用。每个framework都使用同样的方式添加mesos authorisation特性。全球的工程师都在同样的代码。我们都知道：代码越多，bug越多。

移除kibana的一些特定代码之后，我们同时搞定了潜在的bug，添加了新的特性，还降低了维护成本。

minimesos
---

以前，我们使用一个真正的mesos cluster给我们的demo使用。现在只需要使用minimesos就可以了。例如，有一天我想要测试[Elasticsearch framework](https://github.com/mesos/elasticsearch)是不是能够在mesos0.27版本正常工作(它本身是针对0.25版本编译的)。以前我需要写一个0.27版本集群的脚本，现在我只需要修改nimimesos配置文件中的一行就可以了。


参考：http://container-solutions.com/mesos-framework-overview/
