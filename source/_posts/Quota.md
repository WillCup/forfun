
title: Quota
date: 2017-07-28 14:06:28
tags: [youdaonote]
---

在mesos上运行多个framework的时候有个问题，就是如果当前没有其他framework接受剩余资源，默认的WDRF分配器可能会把剩余资源分配给已经满足公平共享的framework。并没有把剩余资源留下备用的机制。尽管dynamic reservations允许operator和framework可在指定的mesos agent上动态保留资源，但是并不能从根本上解决问题，因为每个agent都可能挂掉。

Mesos 0.27提出了quota，用来保证一个role会获取指定大小的最小资源保留。当然，这个role也可以获得比这个quota配置的更多的资源。


quota有些类似reserved resource，稍有不同。quota资源不能绑定到指定的agent上，就是说一个quota保证一个role会有指定大小的资源，但是可以是在集群的任意位置。然而 reservations是用来指定特定的资源给指定role的特定的agent使用。quota只能让operator通过HTTP终端进行配置，而dynamic reservations则可以通过framework的principal验证后进行控制。

注意：reserved reservations是用来满足一个role的quota配置的。例如，如果一个role指定定了一个4CPU和一个agent上的2个CPU的reserved reservations。那么这个role有一个4CPU的最小保证，而不是6个。

Terminology【术语】
---

operator是一个管理mesos集群的东西，可以是人，工具，或者脚本。

在计算机领域，quota通常指下面的事情：
- 一个最小保证
- 一个最大范围
- 以上两者

在mesos中，quota是一个role即将游泳的资源的最小保证。


动力和限制
---

通过下面场景，进一步了解quota。

#### 场景1： 贪婪framework
在一个集群中有两个framework，每个都订阅了自己特有的role，每个role都有相同的权重。frameworkA定于了role RA，frameworkB订阅了role RB。集群中只有一种单一资源：100CPU. fA只需要10个CPU，而FB需要全部的CPU。如果没有Quota的话，FA就会根据默认的公平调度，每个framework都50个CPU，那么就会有40个是浪费的。


#### 场景2： 新framework的资源

贪婪framework FB订阅role RB，是当前集群唯一的framework，那么它就都占了100CPU.如果一个fa新加入进来，需要等待FB的任务完成后才能获取属于自己的50CPU资源。

对于场景2来说，quota自己是不足以解决这个问题的，然而可以一直保持一个资源池作为保留或者或者进行资源抢占。

Operator HTTP终端
---
master的/quota HTTP终端使operator可以操控quota。目前是提供一种类REST接口，支持下面的操作：
- POST设置一个新的quota
- DELETE 移除一个已经存在的quota
- GET 查询当前的quota

目前并不支持更新原有的quota，需要先删除，后添加。

工作原理
---
首先mesos master会处理quota相关的请求，然后allocator来实施quota。

要注意quota具体怎样被实施是依赖于你所使用的allocator的。还有就是实施过程中，可能会对正在运行的framework造成一些影响，因为要把资源重新分配一下。还有资源是只可扩展的资源，端口号这种不行....

quota请求处理过程
---
添加的过程
1. 认证
2. 解析与验证request
3. 如果开启了授权，就授权认证
4. 运行capacity heuristic
5. 存储quota
6. 解除outstanding offer

移除的过程

1. 认证
2. 确认request有效性
3. 鉴权
4. 移除quota


#### capacity heuristic检查
配置错误的quota可能会导致所有的framework都分配不到资源。例如假设一个opetaror给不存在的role 设置了100CPU的quota，而当前集群只用100CPU.

为了防止这种极端情形的出现，mesos master会用一个capacity heuristic来检查quota是不是对于当前集群资源合理。这个heuristic会测试总的quota是不是超过了集群非静态保留的总资源。

> 总资源 - 静态保留资源 》= 总资源 + quota request

注意即便当前是有足够的资源的，但是agent随时可能挂掉，导致集群不能满足配置的quota的需求。

使用force标志可以让check通过，通常是因为operator知道后期会有新的资源加进来。


#### 接触outstanding offer

设置新quota的过程中，master会接触outstanding offer。这避免了当前剩余的没有offer出去，但是很多分配出去给其他framework的资源还没有被使用，造成的资源不足情况。因此，按照下面规则解除outstanding offer：
- 解除至少跟quota请求中一样多的资源
- 如果一个agent有一个offer需要被解除，那么这个agent的所有offer都会被解除。This is done in order to make the potential offer bigger, which increases the chances that a quota'ed framework will be able to use the offer.
- 尽可能多的解除当前quota的role的framework的offer。这会让每个订阅这个role的framework收到一个offer。

#### 通过wDRF Allocator实施
wDRF Allocator首先会根据quota设置的分配资源给role。如果所有的quota都满足了，就为剩余的role执行标准的wDRF。

注意： 一个已经quota的role不会会获取到多一些的未被预定的不可撤销的资源。如果这个role中framework没有选择可撤销的资源，那么他们可能会在role满足之后停止获取offer。这时设置任何比这个role的fair share还低的quota值都很可能导致给这个role的总资源降低。

注意：当前auota保证也是一个quota限制。就是说，一旦一个role的quota满足了，这个role就不会获得更多的资源了。这是为了减轻缺乏quota限制。

如果有多个framework订阅了一个role，这个role有一个quota，标准的wDRF算法来决定这些framework的offer比例。

默认的wDRF allocator认为只有不可撤销的资源才是quota可用的。


容灾
---
如果一个role里有quota，master容灾恢复就很重要。在恢复过程中会有一段时间并不是所有的agent都向master注册了自己。因此不可能所有的quota都被满足。

为了定位这个问题，如果恢复的时候任何前面的quota被探测到后，allocator就进入recovery模式，在这个过程中，allocator并不保证offer。只有满足下面任何一个条件的时候才退出这个模式
- 指定数量的agent重新注册自己(默认80%)
- 一个超时时间(默认10分钟)

当前不足
---
- quota不允许指定详细资源粒度(例如每个节点上10个CPU)
- 不允许指定约束条件(比如，在互斥节点上分配2 * 5个CPU，形成HA)
- 不能为默认的role \*设置quota
- 目前不支持更新

参考： http://mesos.apache.org/documentation/latest/quota/
