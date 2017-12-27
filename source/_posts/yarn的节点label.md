
title: yarn的节点label
date: 2017-12-15 10:59:13
tags: [youdaonote]
---

#### Overview

nodel label是用来把一些共同特征的节点进行归组的手段，然后任务可以指定label要求，运行到指定label的节点上。


现在只支持node partition：

一个node只有一个node partition，所以一个集群被分为多个子集，这些子集互不相交。默认情况下，node是属于DEFAULT partition的。用户可以配置每个partition对于queue中可用多少资源。具体需要看下一节。


有两种不同的node partition：

- exclusive： container会分配在node label完全匹配的node上。(例如指定了partition="x"，那么就被分配到partition=“x”的node上，默认的就分配到默认的node上)
- non-exclusive: 如果一个partition是non-exclusive的，它就可以把空闲资源分配给请求DEFAULT partition的。

用户可以为每个queue指定可以访问的多个node label，app只能使用这个queue能使用的node label的node。




#### Features

- Partition cluster - 每个node都可以有一个label，这样cluster就会被划分为包含多个不相交的子集partition.
- ACL of node-labels on queues - 用户可以指定每个queue可以访问的label的node。
- 指定一个queue可以访问某个label的比例 - 例如queue A可以使用label=hbase的30%的node resource。
- 在resource request中指定nodel label的要求, 那么就只会分配这个label属性的节点资源。如果什么都没有指定的话，就默认为DEFAULT partition。
- 运维相关
    - nodel label和node label的映射关系可以通过RM restart进行恢复
    - 更新node label - admin可在RM运行态更新node和queue的label

#### Configuration

##### 设置ResourceManager启用Node Labels:

yarn-site.xml

Property |	Value
---|---
yarn.node-labels.fs-store.root-dir	| hdfs://namenode:port/path/to/store/node-labels/
yarn.node-labels.enabled | 	true
注意:


确保` yarn.node-labels.fs-store.root-dir`已经存在，而且RM有权限访问(一般是yarn用户访问)

如果用户想把node label存储在RM的本地文件系统，就是用`file:///home/yarn/node-label`

###### 添加/修改 node labels list和node-to-labels mapping to YARN

这是两个连续的步骤。

- 添加Node labels池到cluster中，后面给node添加的label只能是这里面的label，并不是给集群所有节点默认label，这个要注意一下:

    - 执行`yarn rmadmin -addToClusterNodeLabels "label_1(exclusive=true/false),label_2(exclusive=true/false)` 添加 node label.
    - 如果用户没有指定 “(exclusive=…)”, execlusive默认为true.
    - 运行`yarn cluster --list-node-labels`，检查新加的node label是否已生效

- 给node添加label

    - 执行`yarn rmadmin -replaceLabelsOnNode “node1[:port]=label1 node2=label2”`. 给node1添加label1,给node2添加label2. 如果用户没有指定port, 就给这个node上运行的所有的NodeManagers都添加.
    
    
    
##### 给node labels配置Schedulers

###### Capacity Scheduler 配置

Property	| Value
---|---
yarn.scheduler.capacity.<queue-path>.capacity |	Set the percentage of the queue can access to nodes belong to DEFAULT partition. The sum of DEFAULT capacities for direct children under each parent, must be equal to 100.
yarn.scheduler.capacity.<queue-path>.accessible-node-labels |	Admin need specify labels can be accessible by each queue, split by comma, like “hbase,storm” means queue can access label hbase and storm. All queues can access to nodes without label, user don’t have to specify that. If user don’t specify this field, it will inherit from its parent. If user want to explicitly specify a queue can only access nodes without labels, just put a space as the value.
yarn.scheduler.capacity.<queue-path>.accessible-node-labels.<label>.capacity |	Set the percentage of the queue can access to nodes belong to <label> partition . The sum of <label> capacities for direct children under each parent, must be equal to 100. By default, it’s 0.
yarn.scheduler.capacity.<queue-path>.accessible-node-labels.<label>.maximum-capacity |	Similar to yarn.scheduler.capacity.<queue-path>.maximum-capacity, it is for maximum-capacity for labels of each queue. By default, it’s 100.
yarn.scheduler.capacity.<queue-path>.default-node-label-expression |	Value like “hbase”, which means: if applications submitted to the queue without specifying node label in their resource requests, it will use “hbase” as default-node-label-expression. By default, this is empty, so application will get containers from nodes without label.


node label配置示例:

假设我们的queue是这样的

                root
            /     |    \
     engineer    sales  marketing
We have 5 nodes (hostname=h1..h5) in the cluster, each of them has 24G memory, 24 vcores. 1 among the 5 nodes has GPU (assume it’s h5). So admin added GPU label to h5.

我们集群中有5个node，每个node有24G内存，24个vcore。其中一个node h5有GPU，那么admin想给很h5添加一个GPU的label。


用户就可以这样配置：
```
yarn.scheduler.capacity.root.queues=engineering,marketing,sales
yarn.scheduler.capacity.root.engineering.capacity=33
yarn.scheduler.capacity.root.marketing.capacity=34
yarn.scheduler.capacity.root.sales.capacity=33

yarn.scheduler.capacity.root.engineering.accessible-node-labels=GPU
yarn.scheduler.capacity.root.marketing.accessible-node-labels=GPU

yarn.scheduler.capacity.root.engineering.accessible-node-labels.GPU.capacity=50
yarn.scheduler.capacity.root.marketing.accessible-node-labels.GPU.capacity=50

yarn.scheduler.capacity.root.engineering.default-node-label-expression=GPU
```

我们设置了`root.engineering/marketing/sales.capacity=33`, 这样每个队列都保证有33%的资源,也就是 24 * 4 * (1/3) = (32G mem, 32 v-cores).


只有engineering/marketing queue 有权限使用GPU partition (因为 root.<queue-name>.accessible-node-labels).

engineering/marketing queue 都可以使用partition=GPU的 1/2资源，也就是h5的1/2的资源，也就是24 * 0.5 = (12G mem, 12 v-cores).

注意:

配置完后，要执行`yarn rmadmin -refreshQueues`来使配置生效, 然后去RM的web UI检查一下。


#### 为APP指定node label

可以通过java API指定node label的要求。

- ApplicationSubmissionContext.setNodeLabelExpression(..) 设置node label表达式
- ResourceRequest.setNodeLabelExpression(..) 设置单独resource request的node label表达式, 这个可以overwrite ApplicationSubmissionContext的表达式设置
- 在ApplicationSubmissionContext中的setAMContainerResourceRequest.setNodeLabelExpressio用来定位app master container要求的node label

#### 监控

##### 通过UI监控

web UI提供的指标:

Nodes page: http://RM-Address:port/cluster/nodes
Node labels page: http://RM-Address:port/cluster/nodelabels,可以获取到类型 (exclusive/non-exclusive), active node managers的数量, 每个partition的总资源
Scheduler page: http://RM-Address:port/cluster/scheduler, 每个queue关于label的配置


##### 通过命令行监控

- yarn cluster --list-node-labels
- yarn node -status <NodeId> 


#### 其他

YARN Capacity Scheduler, if you need more understanding about how to configure Capacity Scheduler
Write YARN application using node labels, you can see following two links as examples: YARN distributed shell, Hadoop MapReduce
