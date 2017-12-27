
title: slider-部署presto实战
date: 2017-12-15 15:22:36
tags: [youdaonote]
---

首先，使用ambari么有成功，不是特别清楚问题出在哪里，主要是因为对slider还不太熟悉。索性自己手动搞一下slider部署presto，就能相对清楚些，学的也稍微透彻些。


#### 版本构建

- slider-0.90.0-incubating(从github上下载了打包，只所以没用现成的，是因为自己打了一个小patch)。注释掉不关注的子项目，比如`slider-funtest`，注释掉。


找一个跟自己hadoop版本相对应的分支进行打包，我的是2.7，所以就找的0.90.0的tag分支。(有很多人会通过指定hadoop.version的属性来指定依赖的hadoop版本，窃以为有些冒险，因为有可能有依赖，自己的版本高还好些)

- presto-yarn-1.2.1(这个没有现成的包，只能github上弄下来，自己打包)。这个要加上presto server的版本号，自己想用哪个版本就加哪个就成了。
- presto 0.184要求JDK 8U92+, 我安装了8U152
```
mvn clean package -Dpresto.version=0.184
```
#### slider相关配置

##### slider-site.xml
```xml
  <property>
      <name>hadoop.registry.zk.quorum</name>
      <value>datanode01.will.com:2181</value>
    </property>

```


##### slider-env.sh
```
export JAVA_HOME=/server/java/jdk1.8.0_152/
export HADOOP_CONF_DIR=/usr/hdp/current/hadoop-client/conf/
```

#### app相关配置


把上面构建好的slider和presto-yarn的tar包copy到指定机器上解压。

```
cp appConfig-default.json appConfig.json
cp resources-default.json resources.json
```

##### 配置appConfig.json

这个文件是slider针对不同app的个性配置的文件：
```json
{
  "schema" : "http://example.org/specification/v2.0.0",
  "metadata" : { },
  "global" : {
    "site.global.catalog" : "{'hive': ['hive.config.resources=/usr/hdp/2.4.2.0-258/hadoop/etc/hadoop/core-site.xml,/usr/hdp/2.4.2.0-258/hadoop/etc/hadoop/hdfs-site.xml', 'connector.name=hive-hadoop2', 'hive.metastore.uri=thrift://datanode03.will.com:9083'],'tpch': ['connector.name=tpch']}",
    "java_home" : "/server/java/jdk1.8.0_152/",
    "zookeeper.quorum" : "datanode01.will.com:2181",
    "env.MALLOC_ARENA_MAX" : "4",
    "site.global.config_dir" : "/server/presto/etc",
    "application.def" : ".slider/package/presto/presto-yarn-package-1.2.1-0.184.zip",
    "zookeeper.hosts" : "datanode01.will.com",
    "site.global.app_name" : "presto-server-0.184",
    "site.global.coordinator_host" : "${COORDINATOR_HOST}",
    "zookeeper.path" : "/services/slider/users/yarn/presto-will",
    "site.global.app_user" : "yarn",
    "site.global.app_pkg_plugin" : "${AGENT_WORK_ROOT}/app/definition/package/plugins/",
    "site.global.user_group" : "hadoop",
    "site.global.data_dir" : "/server/presto/data",
    "site.global.presto_query_max_memory" : "8GB",
    "site.global.jvm_args" : "['-server', '-Xmx1024M', '-XX:+UseG1GC', '-XX:G1HeapRegionSize=32M', '-XX:+UseGCOverheadLimit', '-XX:+ExplicitGCInvokesConcurrent', '-XX:+HeapDumpOnOutOfMemoryError', '-XX:OnOutOfMemoryError=kill -9 %p']",
    "site.global.presto_query_max_memory_per_node" : "512MB",
    "site.fs.defaultFS" : "hdfs://dd",
    "site.global.presto_server_port" : "8099",
    "site.global.singlenode" : "true",
    "site.fs.default.name" : "hdfs://dd"
  },
  "credentials" : { },
  "components" : {
    "slider-appmaster" : {
      "jvm.heapsize" : "128M"
    }
  }
}

```

##### 配置resources.json

这个文件主要是针对yarn资源申请的。
```json
{
  "schema": "http://example.org/specification/v2.0.0",
  "metadata": {
  },
  "global": {
    "yarn.vcores": "1"
  },
  "components": {
    "slider-appmaster": {
    },
    "COORDINATOR": {
      "yarn.role.priority": "1",
      "yarn.component.instances": "1",
      "yarn.component.placement.policy": "1",
      "yarn.memory": "1500"
    },
    "WORKER": {
      "yarn.role.priority": "2",
      "yarn.component.instances": "3",
      "yarn.component.placement.policy": "1",
      "yarn.memory": "1500"
    }
  }
}

```
默认的resource申请中其实是可以为WORKER或者COORDINATOR指定yarn label的节点的，因为我们的集群暂时没有启用这个，测试阶段先不用这特性了。

参考：https://prestodb.io/presto-yarn/installation-yarn-configuration-options.html

#### 文件夹准备

1. HDFS中要为运行slider应用，也就是presto的用户创建家目录：/user/yarn，而且拥有此目录所有权限。
```
hdfs dfs -mkdr /user/yarn && hdfs dfs -chwon -R yarn /user/yarn
```
2. 所有可能运行presto节点的机器都要有appConfig.json中配置的presto数据目录，而且拥有此目录所有权限。
```
mkdir -p /server/presto/data && chown -R yarn /server/presto/data
```

#### 创建package

到slider目录下，使用yarn用户创建presto的package
```
./bin/slider package --install --name presto --package ../presto-yarn-package-1.2.1-0.184.zip
```

到HDFS中看一下文件：
```
[root@schedule metamap_django]#hdfs dfs -ls /user/yarn/.slider/package/presto
Found 1 items
-rw-r--r--   3 yarn hdfs  449926122 2017-12-14 11:04 /user/yarn/.slider/package/presto/presto-yarn-package-1.2.1-0.184.zip
```


#### 创建app集群
```
./bin/slider create presto-will --template appConfig.json --resources resources.json
```

这个命令会创建presto-will的集群，并启动之。

可以看到HDFS目录里多了一些内容：
```
[root@schedule metamap_django]#hdfs dfs -ls /user/yarn/.slider/cluster/presto-will
Found 9 items
-rw-r--r--   3 root hdfs       1740 2017-12-15 15:16 /user/yarn/.slider/cluster/presto-will/app_config.json
drwxrwxrwx   - yarn hdfs          0 2017-12-15 15:16 /user/yarn/.slider/cluster/presto-will/confdir
drwxr-x---   - yarn hdfs          0 2017-12-15 15:16 /user/yarn/.slider/cluster/presto-will/database
drwxr-x---   - yarn hdfs          0 2017-12-15 14:43 /user/yarn/.slider/cluster/presto-will/generated
drwxr-x---   - yarn hdfs          0 2017-12-15 15:17 /user/yarn/.slider/cluster/presto-will/history
-rw-r--r--   3 yarn hdfs       1436 2017-12-15 14:43 /user/yarn/.slider/cluster/presto-will/internal.json
-rw-r--r--   3 yarn hdfs        651 2017-12-15 14:43 /user/yarn/.slider/cluster/presto-will/resources.json
drwxr-x---   - yarn hdfs          0 2017-12-15 14:43 /user/yarn/.slider/cluster/presto-will/snapshot
drwxr-xr-x   - yarn hdfs          0 2017-12-15 15:16 /user/yarn/.slider/cluster/presto-will/tmp

```

然后到yarn上观察就可以了，名字为`presto-will`的application，点击开，观察所有的component的`Desired`和`Actual`一致了，就代表启动完成了。下面还会有各个组件所在的nodemanager及其container的细腻些，可以由此定位到log位置与coodinator的web界面位置等。


#### Hive配置与验证

配置主要包括两方面：
##### connector的配置
主要是指定hive的thrift位置，以查询hive元数据。具体参考上面的配置文件。
##### hive的HDFS的配置
这个主要是为了解决presto使用hive connector的时候对于HDFS HA的识别，需要在hive connector的配置文件中添加`hive.config.resources=/etc/hadoop/conf/core-site.xml,/etc/hadoop/conf/hdfs-site.xml`属性。对于yarn-presto，就加在appConfig.json中的hive connector(`site.global.catalog`)中了。

##### 验证

主要就是验证一下Hive Connector的正确性吧。下载一个presto的客户端[presto-cli-0.184-executable.jar](https://repo1.maven.org/maven2/com/facebook/presto/presto-cli/0.184/presto-cli-0.184-executable.jar)，重命名为presto，然后给个可执行权限。

```sql
[root@datanode21 presto]# ./presto --server datanode29.will.com:8099 --catalog hive --schema default
presto:default> show tables;
Query 20171215_071741_00000_pxck3 failed: Presto server is still initializing

presto:default> show tables;
                                         Table                                         
---------------------------------------------------------------------------------------
 ast_loan_asset                                                                        
 batting                                                                               
 batting2                                                                              
 customers                                                                             
 h5_test                                                                               
 kylin_intermediate_app_active_cube_4aef52f1_11ae_4dce_b548_f5d3a249331c   
 ...
 ...
 presto:default> select * from web_logs limit 100;
      _version_      |    app     | bytes |     city      |   client_ip   | code | country_code | country_code3 | country_name  | device_family | extension |
---------------------+------------+-------+---------------+---------------+------+--------------+---------------+---------------+---------------+-----------+
 1480895575619534853 | sqoop      |   460 | Hyderabad     | 49.206.186.56 | NULL | IN           | IND           | India         | Other         |           |
 1480895575619534854 | jobbrowser |   269 | Bangkok       | 61.90.20.30   | NULL | TH           | THA           | Thailand      | Other         |           |
 .....
 .....
 
 
```

也可以在coordinator的web界面看到相关的查询信息。


参考：https://prestodb.io/docs/current/installation/cli.html

##### 参考
- https://prestodb.io/presto-yarn/installation-yarn-configuration-options.html
- https://prestodb.io/docs/current/installation/cli.html

#### TODO 
- [benchmark dirver](https://prestodb.io/docs/current/installation/benchmark-driver.html)测试一下性能
- 开启node label，让presto集群跑在计算能力比较强的node上
- 探测合理的配置信息(主要是内存方面)

#### 坑

##### slider不能识别HA的HDFS

在`slider-site.xml`中不配置的dfs.defaultFS的时候slider会自己去判断HDFS全路径，代码如下：
```java
public Path getHomeDirectory() {
    return fileSystem.getHomeDirectory();
}



```

然后生成的启动SliderAppMaster的launcher脚本中就会成为：
```
$JAVA_HOME/bin/java 
 -Djava.net.preferIPv4Stack=true 
 -Djava.awt.headless=true -Xmx128M -ea -esa 
 -Dlog4j.configuration=log4j-server.properties 
 -DLOG_DIR=/yk_data/hadoop/yarn/local/application_1511064572746_35970/container_e51_1511064572746_35970_01_000001 org.apache.slider.server.appmaster.SliderAppMaster create presto-will -cluster-uri hdfs://namenode01.will.com:8020/user/yarn/.slider/cluster/presto-will --rm datanode02.will.com:8030 
 -D hadoop.registry.zk.root=/registry 
 -D hadoop.registry.zk.quorum=datanode01.will.com:2181 1>/yk_data/hadoop/yarn/local/application_1511064572746_35970/container_e51_1511064572746_35970_01_000001/slider-out.txt 2>/yk_data/hadoop/yarn/local/application_1511064572746_35970/container_e51_1511064572746_35970_01_000001/slider-err.txt
```

然后SliderAppMaster验证HDFS文件的时候就会报错：
```
Wrong FS: hdfs://namenode01.will.com:8020/user/yarn, expected: hfds://dd/
```

我们的集群配置了HA，集群namespace是dd。

考虑了一下，源头在于slider获取路径不准确，就直接修改了源码，也就是上面提到的patch：
```java
public Path getHomeDirectory() {
    return new Path("hdfs://dd/user/"+System.getProperty("user.name"));
}
```

其实可以把dd弄成一个slider-client.xml的配置项，暂时图测试方便。

重新build，覆盖既有slider就可以了。

##### 找不到满足条件的node
主要是最开始的时候resources.json中带有coordinator和worker的yarn.label.expression选项，而我们的yarn集群没有开启node label功能，当然node就都不满足条件了。

去掉这个配置，或者开启node label功能，并指定一些计算型节点的node为特定label就可以了。


##### hive配置appConfig.json不生效

就是在create prest-will cluster的时候不能生效，每次集群正常启动之后，使用`./bin/slider status presto-will`都发现相关配置`site.global.catalog`没有更新，但是其他的配置项又是跟本地的appConfig.json是一致的。

解决方案：通过直接`hdfs dfs -get /user/yarn/.slider/cluster/presto-will/app_config.json .`，编辑之后再上传，最后重启slider的presto-will app搞定。

具体为什么有待研究。

##### hive找不到connector Factory

开始以为`connector.name`是可以自己命名的，自己改成`hive-will`了。官方给的例子是`hive-hadoop2`，presto-yarn中给的是`hive-cdh5`,应该也有hdp版本的。虽然我使用的是ambari的hive，属于hdp的，但应该无大碍暂时使用官网的（没有搜到hdp版本的connecor.name对应名字）。


#### 同node多presto worker实例

发现yarn上会出现一个问题，就是当flex的worker数比较多的时候，会出现同一个node上出现多个worker的情况，这时候就会出现失败。
```

http:// http://datanode02.will.com:8088/proxy/application_1511064572746_40916/
classpath:org.apache.slider.client.rest http://datanode02.will.com:8088/proxy/application_1511064572746_40916/
classpath:org.apache.slider.management http://datanode02.will.com:8088/proxy/application_1511064572746_40916/ws/v1/slider/mgmt
classpath:org.apache.slider.publisher http://datanode02.will.com:8088/proxy/application_1511064572746_40916/ws/v1/slider/publisher
classpath:org.apache.slider.registry http://datanode02.will.com:8088/proxy/application_1511064572746_40916/ws/v1/slider/registry
classpath:org.apache.slider.publisher.configurations http://datanode02.will.com:8088/proxy/application_1511064572746_40916/ws/v1/slider/publisher/slider
classpath:org.apache.slider.publisher.exports http://datanode02.will.com:8088/proxy/application_1511064572746_40916/ws/v1/slider/publisher/exports
COORDINATOR Host(s)/Container(s): [datanode16.will.com/container_e51_1511064572746_40916_01_000009]
slider-appmaster Host(s)/Container(s): [datanode23.will.com/container_e51_1511064572746_40916_01_000001]
WORKER Host(s)/Container(s): [datanode21.will.com/container_e51_1511064572746_40916_01_000008, datanode13.will.com/container_e51_1511064572746_40916_01_000007, datanode32.will.com/container_e51_1511064572746_40916_01_000010, datanode11.will.com/container_e51_1511064572746_40916_01_000002, datanode29.will.com/container_e51_1511064572746_40916_01_000004, datanode11.will.com/container_e51_1511064572746_40916_01_000003, datanode12.will.com/container_e51_1511064572746_40916_01_000006, datanode20.will.com/container_e51_1511064572746_40916_01_000005] 
```

这里就是datanode11出现重复了，一会儿就变成了别的，因为重试的原因。直到不再重复位置，如果yarn资源池有些问题，比如只有2个node有足够的资源，但是我们需要8个worker，这2个node的总资源又大于8个worker申请的资源，就可能出现这种情况。

首先想到的是slider应该支持有单个node上只能部署app的一种component的一个实例。



##### failed: Failed to list directory: hdfs://xxxx/premium_record/log_day=20171212

hive里可以查询，但是presto查询单表报错。一般是因为presto server不能找到这个目录，或者没有读取这个目录的权限导致。到HDFS上确认了目录确实存在，因为hive里确实能查的到，所以应该是第二个问题。

从上面步骤中也可以知道我们是使用yarn用户启动的presto集群，找到这个目录，使用HDFS dfs命令，让yarn用户能够访问这个目录就可以了。

##### query.max-memory-per-node set to but only of useable heap available

一般是jvm的内存参数太小，而worker的max-memory-per-node又比较大的原因。
