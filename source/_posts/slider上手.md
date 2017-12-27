
title: slider上手
date: 2017-12-05 17:42:44
tags: [youdaonote]
---

#### 前提
- hadoop2.6+
- HDFS,YARN,ZKK
- JDK1.7
- python 2.6
- openssl


#### 配置集群
配置hadoop集群。

注意：debug设置为非0的能进行debug。如果使用一个vm或者一个sandbox，那么可以修改yarn配置，允许多个container在同一个host上。在yarn-site.xml中修改下面的配置


例子
```
<property>
  <name>yarn.scheduler.minimum-allocation-mb</name>
  <value>256</value>
</property>
<property>
  <name>yarn.nodemanager.delete.debug-delay-sec</name>
  <value>3600</value>
</property>
```


#### 下载slider
#### 配置slider

进入目录`slider-0.80.0-incubating/conf`后，编辑`slider-env.sh`文件：
```
export JAVA_HOME=/usr/jdk64/jdk1.7.0_67
export HADOOP_CONF_DIR=/etc/hadoop/conf
```
如果运行在一个没有安装hadoop的节点上，只需要把相关配置放到相应目录就可以了。也可以通过`slider-clietn.xml`配置hadoop配置路径：
```
<property>
  <name>HADOOP_CONF_DIR</name>
  <value>/etc/hadoop/conf</value>
</property>
```

或者
```
<property>
  <name>HADOOP_CONF_DIR</name>
  <value>${SLIDER_CONF_DIR}/../hadoop-conf</value>
</property>
```

对于一个没有nn HA和 RM HA的集群，修改slider-client.xml配置如下：
```
<property>
    <name>hadoop.registry.zk.quorum</name>
    <value>yourZooKeeperHost:port</value>
  </property>
  <property>
    <name>yarn.resourcemanager.address</name>
    <value>yourResourceManagerHost:8050</value>
  </property>
  <property>
    <name>yarn.resourcemanager.scheduler.address</name>
    <value>yourResourceManagerHost:8030</value>
  </property>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://yourNameNodeHost:8020</value>
  </property>
```

执行命令
```
${slider-install-dir}/slider-0.80.0-incubating/bin/slider version

python %slider-install-dir%/slider-0.80.0-incubating/bin/slider.py version
```


保证没有错误输出，那么slider就已经正确安装了。


#### 发布slider resource

确保所有的文件目录都可以被app实例的创建者使用，我们这里使用yarn作为app创建者。

##### 确保HDFS home存在
```
su hdfs

hdfs dfs -mkdir /user/yarn

hdfs dfs -chown yarn:hdfs /user/yarn
```

##### 创建app包

有几个简单的例子：
- app-packages/memcached-win
- app-packages/hbase
- app-packages/accumulo
- app-packages/storm

根据各个里面的README，创建一个或多个slider app。

##### 安装，配置，启动，验证

- 安装
```
slider install-package --name *package name* --package *sample-application-package*
```

安装包会被发布到HDFS的<User home dir>/.slider/package/<name provided in the command>。

- 创建。分两部分，一个是resource specification，另一个是app configuration
    - resource specification。slider需要知道要部署多少component，需要多少CPU，内存。这些信息放在resources.json中。
    - application configuration。应用的配置信息，例如jvm堆大小等
- 启动。通过cli启动的话
```
cd ${slider-install-dir}/slider-0.80.0-incubating/bin
./slider create cl1 --template appConfig.json --resources resources.json
```
- 验证。到yarn里，打开appmaster，看到slider app master

##### 获取client配置
一个app发布几个用户的细节信息，用来管理app实例。

可以使用registry命令获取这些数据。

- 发布的数据。在app master的/ws/v1/slider/publisher可以看到，通过slider-client status app1
- client配置。/ws/v1/slider/publisher/slider/<config name>
- log位置。ws/v1/slider/publisher/slider/logfolders
- 一些监控UI, jmx终端等。/ws/v1/slider/publisher/slider/quicklinks



#### app生命周期管理
```
./slider start/stop cl1
./slider destroy
./slider flex cl1 --component worker 5
```


