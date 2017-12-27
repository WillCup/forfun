
title: presto之hive-connector
date: 2017-12-15 11:40:46
tags: [youdaonote]
---

#### 概览
hive是由三个部分组成的
- 特定格式的HDFS文件
- metadata数据库，通常是mysql
- HQL及其执行引擎

presto会用到前面两者。

#### 支持的文件类型


ORC、Parquet、Avro、RCFile、SequenceFile、JSON、Text


#### 配置
创建`etc/catalog/hive.properties`文件，挂载`hive-hadoop2` connector作为`hive` catalog，替换掉你的hive metastore的thrift host和port：

```
connector.name=hive-hadoop2
hive.metastore.uri=thrift://example.net:9083
```

#### 多个hive集群

随便你使用多少catalog，添加其他的hive 集群就是了，新增properties文件到`etc/catalog`就可以了。

#### HDFS配置

对于基本设置，presto自动配置了HDFS client，不需要额外的配置文件。但是对于hdfs联邦和NN HA的情况，需要额外指定一下参数才能正常访问HDFS cluster。这个要添加`hive.config.resources`引用到你的hdfs配置文件中：
```
hive.config.resources=/etc/hadoop/conf/core-site.xml,/etc/hadoop/conf/hdfs-site.xml
```

官方建议尽量少添加配置项，多余的配置项可能更容易引起问题。


还有就是这些配置文件必须在每个presto节点上都是有效存在的。


#### HDFS用户

在没有整合Kerberos的HDFS中，presto会使用presto进程的运行用户访问HDFS。我们可以通过配置presto的JVM参数指定访问HDFS的用户。
```
-DHADOOP_USER_NAME=hdfs_user
```


#### 访问带有kerberos认证的HDFS

kerberos认证对于HDFS和hive metastore是都支持的。但是，通过ticket cache进行认证的还没有支持。

#### hive配置项

Property | Name |	Description	Default
--- | --- | ---
hive.metastore.uri | 	Hive metastore 的thrift URI. 如果多个的话，默认使用第一个，后面的作为备用，逗号隔开。Example: thrift://192.0.2.3:9083 or thrift://192.0.2.3:9083,thrift://192.0.2.4:9083	 |
hive.config.resources |逗号分隔的HDFS配置文件, 每个presto server所在的节点都要有效存在.  Example: /etc/hdfs-site.xml	 | 
hive.storage-format	| 创建新表的默认文件格式 |	RCBINARY
hive.compression-codec |	写文件的压缩格式 |	GZIP
hive.force-local-scheduling	| Force splits to be scheduled on the same node as the Hadoop DataNode process serving the split data. This is useful for installations where Presto is collocated with every DataNode. |	false
hive.respect-table-format |	Should new partitions be written using the existing table format or the default Presto format? |	true
hive.immutable-partitions | 	新数据能不能insert到已经存在的partitions? |	false
hive.max-partitions-per-writers |每个writer的最大partition数. |	100
hive.max-partitions-per-scan | 	单个table scan可扫描的最大partition数 |	100,000
hive.metastore.authentication.type |	Hive metastore认证类型,可是是 NONE 或者 KERBEROS. |	NONE
hive.metastore.service.principal | 	The Kerberos principal of the Hive metastore service. |	 
hive.metastore.client.principal | 	The Kerberos principal that Presto will use when connecting to the Hive metastore service.	|  
hive.metastore.client.keytab |	Hive metastore client keytab 位置. |	 
hive.hdfs.authentication.type |	HDFS 认证类型，可以是 NONE 或者 KERBEROS. |	NONE
hive.hdfs.impersonation.enabled	| 启用 HDFS终端user可以假装. |	false
hive.hdfs.presto.principal	| The Kerberos principal that Presto will use when connecting to HDFS.	 | 
hive.hdfs.presto.keytab |	HDFS client keytab 位置.	| 
hive.security |	参考Hive Security Configuration. |	 
security.config-file |	Path of config file to use when hive.security=file. See File Based Authorization for details.	 |
hive.non-managed-table-writes-enabled |	Enable writes to non-managed (external) Hive tables.	| false


#### schema演化

hive允许同一个table的不同partition的数据有不同的schema。有的时候字段类型也有变化，Hive connector 也支持。
- tinyint, smallint, integer，bigint与varchar 之间的相互转化
- real转为double
- int类型的扩展转换，比如tinyint、smallint。

和hive一样，转化失败就会返回null，比如把'foo'转为int


#### 例子

hive connector支持查询与操作hive表和库。然而一些特殊的操作还是需要通过hive执行。

下面创建一个指定位置的hive库：
```sql
CREATE SCHEMA hive.web
WITH (location = 's3://my-bucket/')
```

创建一个ORC格式存储的page_views 表，使用date和conuntry进行分区，根据user进行分桶，共计50个桶(注意分区字段必须是最后的字段)

```sql
CREATE TABLE hive.web.page_views (
  view_time timestamp,
  user_id bigint,
  page_url varchar,
  ds date,
  country varchar
)
WITH (
  format = 'ORC',
  partitioned_by = ARRAY['ds', 'country'],
  bucketed_by = ARRAY['user_id'],
  bucket_count = 50
)
```


删除分区
```sql
DELETE FROM hive.web.page_views
WHERE ds = DATE '2016-08-09'
  AND country = 'US'
```

查询：
```sql
SELECT * FROM hive.web.page_views
```

创建外部表：
```sql
CREATE TABLE hive.web.request_logs (
  request_time timestamp,
  url varchar,
  ip varchar,
  user_agent varchar
)
WITH (
  format = 'TEXTFILE',
  external_location = 's3://my-bucket/data/logs/'
)
```


删除外部表，只会删除metadata，不会删除真实数据。

```
DROP TABLE hive.web.request_logs
```

#### 不足

delete只支持where语句能满足整个分区的情况，也就是说只能一个分区为单位进行delete。
