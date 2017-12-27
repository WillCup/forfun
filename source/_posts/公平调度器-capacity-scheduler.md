
title: 公平调度器-capacity-scheduler
date: 2016-12-05 17:11:05
tags: [youdaonote]
---

## 概览


公平调度器是Hadoop上的一个可配置的调度器，支持多租户共享集群资源，最大化利用集群资源。传统来说，每个组织都有自己的私有资源，这些资源能够在一些极端峰值状态足够满足自己部门的SLA。这样其实是有问题的，所有的集群都需要管理，而且每个部门的资源平均使用率并不高。集群共享其实是一个比较划算的事情，每个部门既能完成工作，又不用去单独管理各自的集群。但是，各部门又有些担心如果共享集群资源的话，其他部门的东西会不会对自己的SLA造成影响。CapacityScheduler就是解决这个问题而生的，它可以让多个部门共享一个大的集群资源，而且各个部门的资源不会被其他部门使用，这保证了部门之间资源共享的灵活性，而且很划算。

多个组织之间共享集群资源，很重要的一点是保证每个组织有一个自保的机制，保证自己的资源不受其他部门资源调度的影响。CapacityScheduler提供了一套严格的保证机制来限制应用、用户、queue等不能无限制地访问集群资源。另外，CapacityScheduler也限制了一个用户或者queue的应用的初始化与pending资源来保证集群的稳定性和公平性。

CapacityScheduler提供的主要概念是queue的概念。一般queue都是管理员创建的，反映了共享集群的各个经济单体。

为了进一步的控制资源共享情况，CapacityScheduler支持纵向队列，也就是资源还可以在sub-queue子队列里进行分配。
To provide further control and predictability on sharing of resources, the CapacityScheduler supports hierarchical queues to ensure resources are shared among the sub-queues of an organization before other queues are allowed to use free resources, there-by providing affinity for sharing free resources among applications of a given organization.

## 特性

如下:
 - Hierarchical Queues 纵向队列 - 
在其他queue使用空闲资源之前，支持资源可被当前组织的sub-queue优先抢占，以此来提供更高的可控性和可预测性。
- 保证容量 - 保证总资源的某个百分比的容量随时听候某queue的调遣。所有提交到这个queue的应用都可以访问这部分资源。 管理员可以灵活配置soft limit和hard limit限制每个queue的容量占比。
- 安全 - 每个queue都有严格的ACLs策略控制谁可以提交应用到哪个queue。另外，也不能让某些普通用户看到或者修改运行在其他queue里的应用。 管理员角色粒度可以对应一个queue或整个scheduler系统。
- 弹性 - 任何队列资源不够的话，都可以使用空闲资源区的资源。 When there is demand for these resources from queues running below capacity at a future point in time, as tasks scheduled on these resources complete, they will be assigned to applications on queues running below the capacity (pre-emption is not supported). This ensures that resources are available in a predictable and elastic manner to queues, thus preventing artifical silos of resources in the cluster which helps utilization.
- 多租户 - Comprehensive set of limits are provided to prevent a single application, user and queue from monopolizing resources of the queue or the cluster as a whole to ensure that the cluster isn’t overwhelmed.
Operability
- 运行时配置优先 - The queue definitions and properties such as capacity, ACLs can be changed, at runtime, by administrators in a secure manner to minimize disruption to users. Also, a console is provided for users and administrators to view current allocation of resources to various queues in the system. Administrators can add additional queues at runtime, but queues cannot be deleted at runtime.
- 清空应用 - .如果一个queue是STOPPED状态，就不能向它或者它的任何sub-queue提交应用了。 正在运行的应用会继续运行到完成，因此queue可以被优雅的清空。管理员也可以启动或者停掉某个queue。
- 基于资源的调度 - 可以多分配一些配置，主要是为了支持一些资源敏感的应用。
Queue Mapping based on User or Group - This feature allows users to map a job to a specific queue based on the user or group.
#### 配置

配置ResourceManager使用CapacityScheduler

conf/yarn-site.xml:
|  属性 | 值  |
|---|---|
| yarn.resourcemanager.scheduler.class  |org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler | 
|   |   |
|   |   |

配置queues

etc/hadoop/capacity-scheduler.xml是CapacityScheduler的主配置文件.
CapacityScheduler又一个预定义的queue: root, 所有的队列都是这个队列的子队列sub-queue.其他的队列通过设置 **yarn.scheduler.capacity.root.queues**属性，弄一个逗号隔开的字符串就行.
CapacityScheduler使用queue path配置queue的层次. queue path是queue层级的全路径, 以root开始, 使用. (点)作为分隔符.
指定queue的所有sub-queue可以这样指定: **yarn.scheduler.capacity.<queue-path>.queues**. 子队列并不自动继承queue的属性。
下面是一个例子，a\b\c为root的子队列，然后a\b还有自己的子队列:
```xml
<property>
  <name>yarn.scheduler.capacity.root.queues</name>
  <value>a,b,c</value>
  <description>The queues at the this level (root is the root queue).
  </description>
</property>

<property>
  <name>yarn.scheduler.capacity.root.a.queues</name>
  <value>a1,a2</value>
  <description>The queues at the this level (root is the root queue).
  </description>
</property>

<property>
  <name>yarn.scheduler.capacity.root.b.queues</name>
  <value>b1,b2,b3</value>
  <description>The queues at the this level (root is the root queue).
  </description>
</property>
```

#### queue属性

Resource Allocation
Property	Description
yarn.scheduler.capacity.<queue-path>.capacity	Queue capacity in percentage (%) as a float (e.g. 12.5). The sum of capacities for all queues, at each level, must be equal to 100. Applications in the queue may consume more resources than the queue’s capacity if there are free resources, providing elasticity.
yarn.scheduler.capacity.<queue-path>.maximum-capacity	Maximum queue capacity in percentage (%) as a float. This limits the elasticity for applications in the queue. Defaults to -1 which disables it.
yarn.scheduler.capacity.<queue-path>.minimum-user-limit-percent	Each queue enforces a limit on the percentage of resources allocated to a user at any given time, if there is demand for resources. The user limit can vary between a minimum and maximum value. The the former (the minimum value) is set to this property value and the latter (the maximum value) depends on the number of users who have submitted applications. For e.g., suppose the value of this property is 25. If two users have submitted applications to a queue, no single user can use more than 50% of the queue resources. If a third user submits an application, no single user can use more than 33% of the queue resources. With 4 or more users, no user can use more than 25% of the queues resources. A value of 100 implies no user limits are imposed. The default is 100. Value is specified as a integer.
yarn.scheduler.capacity.<queue-path>.user-limit-factor	The multiple of the queue capacity which can be configured to allow a single user to acquire more resources. By default this is set to 1 which ensures that a single user can never take more than the queue’s configured capacity irrespective of how idle th cluster is. Value is specified as a float.
yarn.scheduler.capacity.<queue-path>.maximum-allocation-mb	The per queue maximum limit of memory to allocate to each container request at the Resource Manager. This setting overrides the cluster configuration yarn.scheduler.maximum-allocation-mb. This value must be smaller than or equal to the cluster maximum.
yarn.scheduler.capacity.<queue-path>.maximum-allocation-vcores	The per queue maximum limit of virtual cores to allocate to each container request at the Resource Manager. This setting overrides the cluster configuration yarn.scheduler.maximum-allocation-vcores. This value must be smaller than or equal to the cluster maximum.
Running and Pending Application Limits
The CapacityScheduler supports the following parameters to control the running and pending applications:
Property	Description
yarn.scheduler.capacity.maximum-applications / yarn.scheduler.capacity.<queue-path>.maximum-applications	Maximum number of applications in the system which can be concurrently active both running and pending. Limits on each queue are directly proportional to their queue capacities and user limits. This is a hard limit and any applications submitted when this limit is reached will be rejected. Default is 10000. This can be set for all queues withyarn.scheduler.capacity.maximum-applications and can also be overridden on a per queue basis by setting yarn.scheduler.capacity.<queue-path>.maximum-applications. Integer value expected.
yarn.scheduler.capacity.maximum-am-resource-percent /yarn.scheduler.capacity.<queue-path>.maximum-am-resource-percent	Maximum percent of resources in the cluster which can be used to run application masters - controls number of concurrent active applications. Limits on each queue are directly proportional to their queue capacities and user limits. Specified as a float - ie 0.5 = 50%. Default is 10%. This can be set for all queues with yarn.scheduler.capacity.maximum-am-resource-percent and can also be overridden on a per queue basis by setting yarn.scheduler.capacity.<queue-path>.maximum-am-resource-percent
Queue Administration & Permissions
The CapacityScheduler supports the following parameters to the administer the queues:
Property	Description
yarn.scheduler.capacity.<queue-path>.state	The state of the queue. Can be one of RUNNING or STOPPED. If a queue is in STOPPED state, new applications cannot be submitted to itself or any of its child queues. Thus, if the root queue is STOPPEDno applications can be submitted to the entire cluster. Existing applications continue to completion, thus the queue can be drained gracefully. Value is specified as Enumeration.
yarn.scheduler.capacity.root.<queue-path>.acl_submit_applications	The ACL which controls who can submit applications to the given queue. If the given user/group has necessary ACLs on the given queue or one of the parent queues in the hierarchy they can submit applications. ACLs for this property are inherited from the parent queue if not specified.
yarn.scheduler.capacity.root.<queue-path>.acl_administer_queue	The ACL which controls who can administer applications on the given queue. If the given user/group has necessary ACLs on the given queue or one of the parent queues in the hierarchy they can administer applications. ACLs for this property are inherited from the parent queue if not specified.
Note: An ACL is of the form user1, user2spacegroup1, group2. The special value of * implies anyone. The special value of space implies no one. The default is * for the root queue if not specified.
Queue Mapping based on User or Group
The CapacityScheduler supports the following parameters to configure the queue mapping based on user or group:
Property	Description
yarn.scheduler.capacity.queue-mappings	This configuration specifies the mapping of user or group to aspecific queue. You can map a single user or a list of users to queues. Syntax: [u or g]:[name]:[queue_name][,next_mapping]*. Here, u or gindicates whether the mapping is for a user or group. The value is u for user and g for group. name indicates the user name or group name. To specify the user who has submitted the application, %user can be used. queue_name indicates the queue name for which the application has to be mapped. To specify queue name same as user name, %user can be used. To specify queue name same as the name of the primary group for which the user belongs to, %primary_group can be used.
yarn.scheduler.capacity.queue-mappings-override.enable	This function is used to specify whether the user specified queues can be overridden. This is a Boolean value and the default value is false.
Example:
```xml
 <property>
   <name>yarn.scheduler.capacity.queue-mappings</name>
   <value>u:user1:queue1,g:group1:queue2,u:%user:%user,u:user2:%primary_group</value>
   <description>
     Here, <user1> is mapped to <queue1>, <group1> is mapped to <queue2>, 
     maps users to queues with the same name as user, <user2> is mapped 
     to queue name same as <primary group> respectively. The mappings will be 
     evaluated from left to right, and the first valid mapping will be used.
   </description>
 </property>
 ```
#### 其他属性

Resource Calculator
Property	Description
yarn.scheduler.capacity.resource-calculator	The ResourceCalculator implementation to be used to compare Resources in the scheduler. The default i.e. org.apache.hadoop.yarn.util.resource.DefaultResourseCalculator only uses Memory while DominantResourceCalculator uses Dominant-resource to compare multi-dimensional resources such as Memory, CPU etc. A Java ResourceCalculator class name is expected.
Data Locality
Property	Description
yarn.scheduler.capacity.node-locality-delay	Number of missed scheduling opportunities after which the CapacityScheduler attempts to schedule rack-local containers. Typically, this should be set to number of nodes in the cluster. By default is setting approximately number of nodes in one rack which is 40. Positive integer value is expected.
Reviewing the configuration of the CapacityScheduler

Once the installation and configuration is completed, you can review it after starting the YARN cluster from the web-ui.
Start the YARN cluster in the normal manner.
Open the ResourceManager web UI.
The /scheduler web-page should show the resource usages of individual queues.
Changing Queue Configuration

Changing queue properties and adding new queues is very simple. You need to edit conf/capacity-scheduler.xml and run yarn rmadmin -refreshQueues.
$ vi $HADOOP_CONF_DIR/capacity-scheduler.xml
$ $HADOOP_YARN_HOME/bin/yarn rmadmin -refreshQueues
Note: Queues cannot be deleted, only addition of new queues is supported - the updated queue configuration should be a valid one i.e. queue-capacity at each level should be equal to 100%.

