
title: hive-on-spark配置
date: 2017-12-12 18:15:55
tags: [youdaonote]
---

#### 安装spark

hive on spark默认也是支持spark on yarn的模式的。

#### 配置YARN
不能使用capacity scheduler，必须使用fair scheduler【....看到这里就想放弃了】

yarn.resourcemanager.scheduler.class=org.apache.hadoop.yarn.server.resourcemanager.scheduler.fair.FairScheduler

#### 配置Hive

##### 添加spark依赖：
-  hive2.2.0之前，把spark-assembly jar链接到HIVE_HOME/lib中
-  2.2.0之后，hive on spark需要运行在spark 2.0.0以上的版本，没有assembly jar包 
    - 要运行YARN模式的话，添加下面的包到HIVE_HOME/lib
        - scala-library
        - spark-core
        - spark-network-common
    - 运行local模式的话，添加下面包的链接到hive_home/lib中
        - chill-java  chill  jackson-module-paranamer  jackson-module-scala  jersey-container-servlet-core
        - jersey-server  json4s-ast  kryo-shaded  minlog  scala-xml  spark-launcher
        - spark-network-shuffle  spark-unsafe  xbean-asm5-shaded


##### 配置hive的执行引擎
```
set hive.execution.engine=spark;
```

##### 配置spark相关的参数
```
set spark.master=<Spark Master URL>
set spark.eventLog.enabled=true;
set spark.eventLog.dir=<Spark event log folder (must exist)>
set spark.executor.memory=512m;             
set spark.serializer=org.apache.spark.serializer.KryoSerializer;
```
hive 2.2.0以前，要把spark-assembly jar到HDS中
```
<property>
  <name>spark.yarn.jar</name>
  <value>hdfs://xxxx:8020/spark-assembly.jar</value>
</property>
```

2.2.0以上版本后, 把$SPARK_HOME/jars中的所有jar都上传到hdfs指定文件夹中：
```

<property>
  <name>spark.yarn.jars</name>
  <value>hdfs://xxxx:8020/spark-jars/*</value>
</property>
```

#### 配置spark
属性 | 推荐设置
---|---
spark.executor.cores	| Between 5-7, See tuning details section
spark.executor.memory	| yarn.nodemanager.resource.memory-mb * (spark.executor.cores / yarn.nodemanager.resource.cpu-vcores) 
spark.yarn.executor.memoryOverhead | 	15-20% of spark.executor.memory
spark.executor.instances |	Depends on spark.executor.memory + spark.yarn.executor.memoryOverhead, see tuning details section.



#### 推荐配置
```
mapreduce.input.fileinputformat.split.maxsize=750000000
hive.vectorized.execution.enabled=true

hive.cbo.enable=true
hive.optimize.reducededuplication.min.reducer=4
hive.optimize.reducededuplication=true
hive.orc.splits.include.file.footer=false
hive.merge.mapfiles=true
hive.merge.sparkfiles=false
hive.merge.smallfiles.avgsize=16000000
hive.merge.size.per.task=256000000
hive.merge.orcfile.stripe.level=true
hive.auto.convert.join=true
hive.auto.convert.join.noconditionaltask=true
hive.auto.convert.join.noconditionaltask.size=894435328
hive.optimize.bucketmapjoin.sortedmerge=false
hive.map.aggr.hash.percentmemory=0.5
hive.map.aggr=true
hive.optimize.sort.dynamic.partition=false
hive.stats.autogather=true
hive.stats.fetch.column.stats=true
hive.vectorized.execution.reduce.enabled=false
hive.vectorized.groupby.checkinterval=4096
hive.vectorized.groupby.flush.percent=0.1
hive.compute.query.using.stats=true
hive.limit.pushdown.memory.usage=0.4
hive.optimize.index.filter=true
hive.exec.reducers.bytes.per.reducer=67108864
hive.smbjoin.cache.rows=10000
hive.exec.orc.default.stripe.size=67108864
hive.fetch.task.conversion=more
hive.fetch.task.conversion.threshold=1073741824
hive.fetch.task.aggr=false
mapreduce.input.fileinputformat.list-status.num-threads=5
spark.kryo.referenceTracking=false
spark.kryo.classesToRegister=org.apache.hadoop.hive.ql.io.HiveKey,org.apache.hadoop.io.BytesWritable,org.apache.hadoop.hive.ql.exec.vector.VectorizedRowBatch
```
