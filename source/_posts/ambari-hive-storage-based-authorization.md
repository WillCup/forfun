
title: ambari-hive-storage-based-authorization
date: 2016-12-05 17:08:35
tags: [youdaonote]
---

#### 概要
hive使用hdfs的文件系统权限来约束用户对database/table/partition的访问权限，可以让用户属于多个group，不同权限的数据属于不同group就可以随意组合用户的权限了。

#### ambari配置参数
```xml
<property>
  <name>hive.security.metastore.authorization.manager</name>
  <value>org.apache.hadoop.hive.ql.security.authorization.DefaultHiveMetastoreAuthorizationProvider</value>
  <description>authorization manager class name to be used in the metastore for authorization.
  The user defined authorization class should implement interface
  org.apache.hadoop.hive.ql.security.authorization.HiveMetastoreAuthorizationProvider.
  </description>
 </property>

<property>
  <name>hive.security.metastore.authenticator.manager</name>
  <value>org.apache.hadoop.hive.ql.security.HadoopDefaultMetastoreAuthenticator</value>
  <description>authenticator manager class name to be used in the metastore for authentication.
  The user defined authenticator should implement interface 
  org.apache.hadoop.hive.ql.security.HiveAuthenticationProvider.
  </description>
</property>

<property>
  <name>hive.metastore.pre.event.listeners</name>
  <value> </value>
  <description>pre-event listener classes to be loaded on the metastore side to run code
  whenever databases, tables, and partitions are created, altered, or dropped.
  Set to org.apache.hadoop.hive.ql.security.authorization.AuthorizationPreEventListener
  if metastore-side authorization is desired.
  </description>
</property>
```

#### 测试时间
```shell
hive> create database willtest;

hive> CREATE TABLE `batting`(
    >   `id` int, 
    >   `dtdontquery` string, 
    >   `name` string)
    > PARTITIONED BY ( 
    >   `dt` string);
    
hive> insert into willtest.batting partition(dt='20150409') values(12, 'will test string', 'will');

```
1. 创建测试库willtest
2. 创建测试表willtest.batting
3. 插入一条测试数据，使用ambari用户maming到hive view查询此表，正常。
4. 观察一下hdfs上的文件.
```shell
[hdfs@data-test02 root]$ hdfs dfs -ls /apps/hive/warehouse
drwxrwxrwx   - hive hdfs          0 2016-05-27 15:22 /apps/hive/warehouse/willtest.db

[hdfs@data-test02 root]$ hdfs dfs -ls /apps/hive/warehouse/willtest.db
drwxrwxrwx   - hive hdfs          0 2016-05-27 15:23 /apps/hive/warehouse/willtest.db/batting
```
现在这个库和表的权限都是谁都可以随便操作的，而且owner是hive，group是hdfs。
5. 修改batting表的权限，期望我们的ambari用户maming不能查询；
```shell
[hdfs@data-test02 root]$ hdfs dfs -chmod -R 770 /apps/hive/warehouse/willtest.db/batting
[hdfs@data-test02 root]$ hdfs dfs -ls /apps/hive/warehouse/willtest.db
drwxrwx---   - hive hdfs          0 2016-05-27 15:23 /apps/hive/warehouse/willtest.db/batting
```
先把group和all的权限约束一下，这里因为maming本来就不是hdfs group的，所以就暂时不用修改group了。
6. 再使用ambari用户amming查询willtest.batting，报错：
```
  org.apache.ambari.view.hive.client.HiveErrorStatusException: H110 Unable to submit statement. Error while compiling statement: FAILED: SemanticException Unable to fetch table batting. java.security.AccessControlException: Permission denied: user=maming, access=READ, inode="/apps/hive/warehouse/willtest.db/batting":hive:hdfs:drwxrwx---
	...
```
7. 补充一下maming等几个用户的权限
```shell
[hdfs@data-test02 root]$ hdfs groups maming
maming : maming
[hdfs@data-test02 root]$ hdfs groups sqoop
sqoop : hadoop
[hdfs@data-test02 root]$ hdfs groups hdfs
hdfs : hadoop hdfs
```
8. 将maming暂时加入hdfs这个group之后再试一下，成功
```

```
测试成功。


#### 延伸
好，基本需求能够实现。那么如果我们为某个group分配了某个db的权限。如果我们新增一个表batting2呢，再看一下文件属性
```shell
[hdfs@data-test02 root]$ hdfs dfs -ls /apps/hive/warehouse/willtest.db
drwxrwx---   - hive hdfs          0 2016-05-27 15:23 /apps/hive/warehouse/willtest.db/batting
drwxrwxrwx   - hive hdfs          0 2016-05-27 15:51 /apps/hive/warehouse/willtest.db/batting2
```
问题出现，我们需要修改创建数据仓库文件的默认权限才行，不然我们就需要每次增加新库都得改一下table文件权限。

找到两个相关参数
|参数 |含义 |
|---|---|
| hive.warehouse.subdir.inherit.perms |true/false 是否 **继承上级文件夹的umask** |
| hive.files.umask.value | 0770 解释说是被上面那个参数替换了 |
那就设置hive.warehouse.subdir.inherit.perms为true，重启一下hive，再去创建表看一下文件夹的权限。
```
[hdfs@data-test02 root]$ hdfs dfs -chmod -R 770 /apps/hive/warehouse/
[hdfs@data-test02 root]$ hdfs dfs -ls /apps/hive/warehouse/
Found 2 items
drwxrwx---   - hive hdfs          0 2016-05-27 15:10 /apps/hive/warehouse/batting
drwxrwx---   - hive hdfs          0 2016-05-27 16:26 /apps/hive/warehouse/willtest.db
[hdfs@data-test02 root]$ hdfs dfs -ls /apps/hive/warehouse/willtest.db
Found 4 items
drwxrwx---   - hive hdfs          0 2016-05-27 15:23 /apps/hive/warehouse/willtest.db/batting
drwxrwx---   - hive hdfs          0 2016-05-27 15:51 /apps/hive/warehouse/willtest.db/batting2
drwxrwx---   - hive hdfs          0 2016-05-27 16:27 /apps/hive/warehouse/willtest.db/batting4
```
先把willtest.db的文件夹权限改成了770，然后在willtest库里创建了一个新的表batting4，可以看到hdfs里的权限已经跟父目录保持一致了。
这样我们给指定了某个db的文件夹权限之后，下面的table的新增与变化，权限都会稍微固定一些。


#### 参考
https://cwiki.apache.org/confluence/display/Hive/Storage+Based+Authorization+in+the+Metastore+Server
https://cwiki.apache.org/confluence/display/Hive/HCatalog+Authorization
