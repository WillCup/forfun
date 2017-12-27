
title: marathon-应用部署
date: 2017-08-31 18:23:25
tags: [youdaonote]
---

每次app的配置文件修改生效就出发一次deployment。一次deployment是一系列动作的集合：
 - 启动或者停止一个或多个app
 - 升级一个或多个app
 - 扩展一个或多个app

Deloyment需要花费时间，并不能马上使app恢复可用。

只要每个app只被一次deployment修改，那么你就可以同时执行多个deployment。在我们尝试执行一个已经被别人修改过的deployment 的时候，marathon会驳回此次deployment。


#### 依赖
没有依赖的app可以以任意顺序部署。如果app间有依赖，那么deployment就应该严格按照app的依赖顺序执行。

![](https://mesosphere.github.io/marathon/img/dependency.png)

如上图，app是以来db的。

启动： 先启动db，再启动app
停止： 先停止app，再停止db
升级： 参考 Rolling Restart 部分
扩容： db先扩容，然后app

#### Rolling Restart
我们可以通过 Rolling Restart来部署新版本的app。通常，有两个阶段：启动一些新版本的进程，然后停掉旧版本的进程。

在marathon中，可以在app级别使用minimumHealthCapacity 定义一个upgrade strategy。


minimumHealthCapacity是指达升级过程中，处于正常服务状态的实例不能少于整体的百分之多少。如果是0， 在新版本部署前，就把所有的旧版本杀掉。如果是1，就先启动所有的新版本实例，然后再杀掉所有的旧版本实例。


这个动作在有依赖关系的时候会复杂一些。上面的例子来说：
1. 先升级db到所有的实例都完成；
2. 升级app到所有的实例都完成。
上面两个app的具体过程，还是按照upgradeStrategy来处理。假设我们设置的db和app对应的参数分别是0.6和0.8，那么过程中就会出现db有12个实例（6个新的，6个旧的），app出现32个实例（16个新的，16个旧的）

#### 强制deployment

一个app只能同时被一个deployment修改。其他的修改必须等待当前的完成才行。如果运行deployement的时候加上force标记就可以打破这个限制。

但是**注意**： force标记只适用于失败的deployment

如果指定了force，那么所有其他的修改都会被取消。这个操作可能让系统进入一个不一致状态，尤其是在一个app执行rolling upgrade的时候，会导致一部分新版本app与一些旧版本app并行的秦广。如果新的deployment没有更新这些app，就会一直处于这种不一致状态了。

唯一可以进行force-update的app是那些只有一个app实例的。**要强制部署多个app的唯一理由，是修正一个失败的deployment。**


#### 失败的deployment
deployment是包含很多严谨的、有次序的步骤的。

下面是一些永远不能成功的情景：
- 新app没启动成功
- 新app不能进入healthy状态
- 新app的依赖没有声明、或者不可用
- 集群资源耗尽
- app使用docker container，docker相关配置没有启用

这些情况会导致app一直循环重启，又不能成功，需要重新部署正确的app配置才行。

#### /v2/deployment终端API

参考 [marathon的REST API](https://mesosphere.github.io/marathon/docs/rest-api.html)
