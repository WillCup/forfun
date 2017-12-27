
title: Roles
date: 2017-07-25 11:37:29
tags: [youdaonote]
---

role
---
好多host-level的OS(例如Linux，BSDs等)都支持多用户。同样的，mesos也是一个多用户集群管理系统，使用一个mesos集群管理一个组织的所有资源，服务于这个组织的用户。

mesos里与资源相关的东西如下：
 - 用户之间对于资源的公平共享
 - 保证用户资源(优先级、隔离、限额等)
 - 精确地资源计算能力
    - 分别有多少资源是 已分配/未分配等等
    - 基于每个用户计算
    

在mesos中，我们为所有用户分配一个或多个role。其实，role就是mesos资源的消费者。这个消费者可以代表一个组织的一个用户，也可能是一个组、一个群、一个service，一个framework等等。

调度器会订阅一个或多个role，监听他们申请到的资源，然后执行调度任务。

下面是一些例子：
 - 保证一个role可以分配到指定的资源（通过quota指定）
 - 保证个别agent上的一些或者所有资源都分配给一个指定的role(通过reservations)
 - 保证资源是在role之间公平共享(通过DRF)
 - 声明某些role获取相对更多的资源(通过weight)
 

role与acl
---
有两种方式来控制一个framework应该订阅哪些role。首先，可以指定ACLs。其次，可以在mesos master启动的时候传递--roles参数，代表role的白名单，逗号隔开。注意在HA环境下，要保证多个master的启动白名单是一致的。

role与framework的关联
---
framework通过制定FrameworkInfo里的roles订阅role。

作为用户，你可以在启动framework的时候指定要订阅哪些role。具体怎样弄需要看你的framework相关的接口是怎样的。例如，一个单用户的scheduler可能会通过--mesos_role参数指定，多用户的scheduerl就通过 --mesos_roles参数指定。或者也可以通过一个LDAP系统动态调整framwork和role之间的订阅关系。

#### 订阅多个role
上面有提到，一个framework可以同时订阅多个role，通过MULTI_ROLE实现。

一个framework对外提供资源的时候，这些资源只能属于一个role。framework可以通过Offer对象的allocation_info.role字段来确认某个offer是提供给哪个role的，或者通过每个提供的Resource对象的allocation_info.role字段。（当前实现中，在一个Offer中的所有资源都是提供给同一个role的）

#### 同一个role的多个framework
多个framework可以订阅同一个role。例如：一个framework可以创建一个持久化的数据卷，然后往里面写数据。一旦这个写数据的任务完成后，这个数据卷就可以提供给订阅同一个role的其他的framewo使用了。第二个framework就可以读取前一个framework写的数据。

然而，配置多个framework使用同一个role的时候需要慎重一些，因为所有的framework都能够访问到这个role申请到的资源。例如，如果一个framework把一些敏感信息存到了数据卷中，其他的framework就可以读取了。类似地，如果一个framework创建了一个持久化的数据卷，其他framework可以直接偷走自己用来启动自己的任务。通常来说，订阅同一个role的framework之间是具有良好的协作关系的。

资源与role的关系
---
资源通过reservation分配给一个role。资源可以永久静态地或者动态地提供给role使用。framework和operator可以指定一定量的资源作为某个role的最小持有量。

默认的role
---
名称为\*的role是一个特殊的role。没有分配的资源是用这个\*代表的。

对于没有提供FrameworkInfo.role的framework注册，会分配给他的就是\*这个role。

role与资源分配
---
默认，mesos master使用基于权重的资源分配方式(weighted Dominant Resource Fairness)。 In particular, this implementation of wDRF first identifies which role is furthest below its fair share of the role’s dominant resource. 然后每个订阅这个role的framework就获取到返回的资源。

资源分配过程可以通过指定role的权重进行自定义控制。


Role与quota
---
为了保证一个role能够分配到指定大小的资源，我们可以通过/quota终端指定quota。

资源分配者在公平分配剩余资源之前，会首先满足quota的需求。


Role与Principal
---
Principal代表一个与mesos互动的实体，更像是用户名。例如，framework在向mesos master注册的时候会提供principal，在使用HTTP终端的时候会提供principal。一个实体需要提供principal来认证自己的身份。

Role，仅仅用来控制资源分配。
