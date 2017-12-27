
title: hiveserver2负载均衡
date: 2017-04-26 16:47:44
tags: [youdaonote]
---

hiveserver2是通过在zookeeper注册一个namespace，然后管理所有的hiveserver实例，实现动态服务发现的。

znode样式：
```
/<hiveserver2_namespace>/serverUri=<host:port>;version=<versionInfo>; sequence=<sequence_number>,
```

hiveserver实例会在znode上设置一个watch。当znode被修改的时候，watch会发送给hiveserver相关信息。这个通知能够让所有的hiveserver实例知道它是不是对于client端可用。

当有hiveserver实例退出时，会从zk的对应node里移除，但是只对新的客户端连接生效。(已经连接的session不能生效了)。只有已经连接到这个hiveserver的最后一个client的session结束后，才会自动把这个hiveserver完全关闭，下面这个命令就是做这个工作的。


使用下面命令移除一个hiveserver
```
hive --service hiveserver2 --deregister <package ID>
```

#### 没有zk时的查询
下面是一个传统的查询流程：
 - 通过JDBC/ODBC driver连接到HS2实例，建立一个session
 - 然后每次查询的时候client都发送语句给HS2，转化成Hadoop上的执行任务
 - 每个查询的结果都写道一个临时文件中
 - 客户端driver从HS2抽取临时文件中的数据记录

![典型查询流程](https://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.4.0/bk_hadoop-ha/content/figures/2/figures/Query_Ex_Path_No_ZK.png)


#### 带有zk的查询

因为可以使用动态服务发现，所以客户端driver必须知道怎样使用这个特性。对于HDP2.2或者JDBC driver2.0.0版本之后才能支持。

动态服务发现实现如下：
- 多个HS2实例使用zk注册自己
- 客户端driver连接zk
    >  jdbc:hive2://<zookeeper_ensemble>;serviceDiscoveryMode=zooKeeper; zooKeeperNamespace=<hiveserver2_namespace
- zk随机返回一个host:port给客户端
- 客户端执行单个服务传统查询过程

![带有zk的查询过程](https://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.4.0/bk_hadoop-ha/content/figures/2/figures/Query_Ex_Path_With_ZK.png)


#### 配置
hive.zookeeper.quorum zk列表
hive.zookeeper.session.timeout 超时就关闭session
hive.server2.support.dynamic.service.discovery  设置为true
hive.server2.zookeeper.namespace    指定一个就行了，默认是hiveserver2




- https://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.4.0/bk_hadoop-ha/content/ha-hs2-requests.html
