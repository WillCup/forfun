
title: 在yarn上安装presto
date: 2017-12-04 18:18:20
tags: [youdaonote]
---

#### 前置条件
- hdp 2.2+ 或者CDH5.4+
- slider 0.80.0+
- jdk 1.8
- zookeeper
- openssl >= 1.0.1e-16
- ambari 2.1

#### presto安装目录结构

使用ambari slider view安装基于yarn的presto集群的时候，安装目录与标准的是不一样的。

如果使用slider脚本或者ambari slider view来部署presto到yarn上的话，presto是会使用presto server的tar包的形式进行安装的(不是通过rpm)。当yarn app启动后，可发现presto server安装在yarn上的nodemanager的`yarn.nodemanager.local-dirs`。例如，配置`yarn.nodemanager.local-dirs`为`/mnt/hadoop/nm-local-dirs `，且`app_user`配置为`yarn`，那么就安装在`/mnt/hadoop-hdfs/nm-local-dir/usercache/yarn/appcache/application_<id>/container_<id>/app/install/presto-server-<version>`。container_id之前的部分在slider中叫做AGENT_WORK_ROOT，也就是说`AGENT_WORK_ROOT/app/install/presto-server-<version>`。

通常，使用tar安装的presto，catalog、plugin、lib目录等都在presto-server主目录下。catalog目录在`AGENT_WORK_ROOT/app/install/presto-server-<version>/etc/catalog`, plugin和lib目录在`AGENT_WORK_ROOT/app/install/presto-server-<version>/plugin`和`AGENT_WORK_ROOT/app/install/presto-server-<version>/lib`.启动脚本在`AGENT_WORK_ROOT/app/install/presto-server-<version>/bin`.


presto日志是基于数据目录的配置的。

参考：https://prestodb.io/presto-yarn/installation-yarn-directory-structure.html

#### presto配置
安装过程中，ambari slider view允许你进行配置。

如果使用mabri进行安装，可以通过UI配置，如果是手动安装的话，就自己编辑配置文件。

主要配置文件为appConfig.json和resources-[singlenode|multinode].json，需要在运行presto之前配置好。在下面提出的位置有样例配置：
```
presto-yarn-package/src/main/resources
```

` presto-yarn-package/src/main/resources/appConfig.json`和`presto-yarn-package/src/main/resources/resources-multinode.json`是对应的默认配置。

##### appConfig.json

- site.global.app_user。 默认是yarn，启动presto的用户。确认app_user要有一个HDFS home目录。如果要访问hive的话，也要确认这个app_user有相应的权限
- site.global.user_group。默认是hadoop
- site.global.data_dir。默认是`/var/lib/presto/data`，presto的数据目录，应该在启动之前就存在，并且是属于app_user，否则slider就会因权限问题不能启动了。
```
mkdir -p /var/lib/presto/data
chown -R yarn:hadoop /var/lib/presto/data
```
- site.global.config_dir,默认是/var/lib/presto/etc。presto配置文件所在的目录，包含node.properties, jvm.config, config.properties以及connector配置文件等。这些文件会从模板`presto-yarn-package/package/templates/*.j2`和相关appConfig.json的参数生成。
- site.global.singlenode。默认true，当前node既做为coordinator也作为worker。
- site.global.presto_query_max_memory。在config.properties文件中是query.max_memroy，默认50G.
- site.global.presto_query_max_memory_per_node，默认1G
- site.global.presto_server_port，默认8080
- site.global.catalog.默认是tpch connector。这个是用来配置presto的connector的。应该对应于非基于yarn的presto集群的connector.properites。格式一般为：` {‘connector1’ : [‘key1=value1’, ‘key2=value2’..], ‘connector2’ : [‘key1=value1’, ‘key2=value2’..]..}.`。这个会创建connector1.properties, connector2.properties两个配置文件，带有不同的entry。看一个hive.properties的例子
```
"site.global.catalog": "{'hive': ['connector.name=hive-cdh5', 'hive.metastore.uri=thrift://${NN_HOST}:9083'], 'tpch': ['connector.name=tpch']}"
```
这里的NN_HOST运行时会被替换成NN的地址，如果hive metastore跟NN没在一起，需要自己修改一下。
- site.global.jvm_args。这个是生成presto的jvm.properties文件的，默认是heapsize是1G.
```
"site.global.jvm_args": "['-server', '-Xmx1024M', '-XX:+UseG1GC', '-XX:G1HeapRegionSize=32M', '-XX:+UseGCOverheadLimit', '-XX:+ExplicitGCInvokesConcurrent', '-XX:+HeapDumpOnOutOfMemoryError', '-XX:OnOutOfMemoryError=kill -9 %p']",
```
- site.global.log_properties. 配置presto的日志级别默认是`[‘com.facebook.presto=INFO’]`。应该是一行一个表达式。例子
```
"site.global.log_properties": "['com.facebook.presto.hive=WARN', 'com.facebook.presto.server=INFO']"
```
- site.global.additional_node_properties和site.global.additional_config_properties。
- site.global.plugin
```
"site.global.plugin": "{'ml': ['presto-ml-${presto.version}.jar']}",
```
- site.global.app_name
- application.def. 对于slider用户，安装presto的命令运行的时候，日志会打印出这个参数。如果不是自定义的presto包，不必理会这个参数。
- java_home ， 默认是/usr/lib/jvm/java
- appConfig.json里的类似${COORDINATOR_HOST}, ${AGENT_WORK_ROOT}的变量是在运行时确认的。

##### resources.json

这个配置可以对应于全局，也可以针对每个组件
- yarn.vcores: 默认是全局的，-1
- yarn.component.instances，默认coordinator是-1，worker是3.多节点模式下的例子配置中`presto-yarn-package/src/main/resources/resources-multinode.json`是1个coordinator，3个worker。他们的分布有着较严格的策略，每个node上只会运行一个实例。当节点数不够申请的worker时，这个app就会失败。如果都用作presot节点的话，那么worker的数量应该是集群中的nodemanager数量 -1 ，留一个作为coordinator。
- yarn.memory。默认1500M，是site.global.jvm_args的-Xmx参数，被presto的jvm所使用的。slider推荐要比这个大一些。yarn.memory应该比任何jvm申请的堆都要大一丢丢，推荐最少大50%。
- yarn.label.expression. coordinator或者worker。






参考：https://prestodb.io/presto-yarn/installation-yarn-configuration-options.html


#### 使用ambari slider view 安装基于yarn的presto集群

1. 安装ambari server
2. 下载slider包
3. 把presto包copy到ambari server的`/var/lib/ambari-server/resources/apps/`路径下
4. 重启ambari-server
5. 登陆ambari
6. 安装基础组件：HDFS, YARN, Zk， Slider等
7. 保证slider-env.sh里已经指定了JAVA_HOME和HADOOP_CONF_DIR。
8. 对于zk，如果不是安装在/usr/lib/zookeeper:
    - 在slider配置里添加zk.home配置变量
    - 如果不是2181端口，添加slider.zookeeper.quorum配置
9. 启动服务，到ambari创建slider view，然后创建app
10. 提供presto服务的相关配置细节，UI会提供一些默认参数。
11. app名字应该是小写的：presto1
12. 配置
13. 准备slider的HDFS目录，这个根据global.app_user来的。
14. 修改global.presto_server_port为8080之外的其他端口，以免跟ambari srever冲突
15. 预创建所有节点上的数据目录`var/lib/presto`，可以自己修改
16. 其他自定义配置
17. 完成。这个动作相当于在bin/slider脚本执行package --install和create，然后就可以看到presto已经在yarn上运行起来了。
    - 从slider view监控运行状态
    - Quick Links，观察yarn UI
18. 如果job失败，可以到对应节点上找日志看
19. 可以在View界面管理app的生命周期(start, stop, fliex, destory)。


#### 手动使用slider安装presto集群

