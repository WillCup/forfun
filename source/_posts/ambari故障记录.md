
title: ambari故障记录
date: 2016-12-05 17:11:36
tags: [youdaonote]
---

## 问题
  - ambari的metric界面上全都load不出来
  - hive命令行进不去，通过beeline连接成功，但是执行命令卡住
  

##### 观察ambari-server.log里：
```
27 May 2016 06:00:16,468  WARN [C3P0PooledConnectionPoolManager[identityToken->2rvy2w9g1rvd7t51qlbfjd|229f66ed]-AdminTaskTimer] ThreadPoolAsynchronousRunner:220 - Task com.mchange.v2.resourcepool.BasicResourcePool$ScatteredAcquireTask@13d18137 (in deadlocked PoolThread) failed to complete in maximum time 60000ms. Trying interrupt().
27 May 2016 06:00:16,481  WARN [C3P0PooledConnectionPoolManager[identityToken->2rvy2w9g1rvd7t51qlbfjd|229f66ed]-AdminTaskTimer] ThreadPoolAsynchronousRunner:220 - Task com.mchange.v2.resourcepool.BasicResourcePool$ScatteredAcquireTask@28fdf8 (in deadlocked PoolThread) failed to complete in maximum time 60000ms. Trying interrupt().
27 May 2016 06:00:16,481  WARN [C3P0PooledConnectionPoolManager[identityToken->2rvy2w9g1rvd7t51qlbfjd|229f66ed]-AdminTaskTimer] ThreadPoolAsynchronousRunner:220 - Task com.mchange.v2.resourcepool.BasicResourcePool$ScatteredAcquireTask@7a7239f6 (in deadlocked PoolThread) failed to complete in maximum time 60000ms. Trying interrupt().
27 May 2016 06:32:26,157  WARN [C3P0PooledConnectionPoolManager[identityToken->2rvy2w9g1rvd7t51qlbfjd|74e262f6]-AdminTaskTimer] ThreadPoolAsynchronousRunner:220 - com.mchange.v2.async.ThreadPoolAsynchronousRunner$DeadlockDetector@40bbff68 -- APPARENT DEADLOCK!!! Creating emergency threads for unassigned pending tasks!
27 May 2016 06:32:26,171  WARN [C3P0PooledConnectionPoolManager[identityToken->2rvy2w9g1rvd7t51qlbfjd|74e262f6]-AdminTaskTimer] ThreadPoolAsynchronousRunner:220 - com.mchange.v2.async.ThreadPoolAsynchronousRunner$DeadlockDetector@40bbff68 -- APPARENT DEADLOCK!!! Complete Status: 
	Managed Threads: 3
	Active Threads: 3
	Active Tasks: 
		com.mchange.v2.resourcepool.BasicResourcePool$ScatteredAcquireTask@5383aba6
			on thread: C3P0PooledConnectionPoolManager[identityToken->2rvy2w9g1rvd7t51qlbfjd|74e262f6]-HelperThread-#2
		com.mchange.v2.resourcepool.BasicResourcePool$ScatteredAcquireTask@38f907f5
			on thread: C3P0PooledConnectionPoolManager[identityToken->2rvy2w9g1rvd7t51qlbfjd|74e262f6]-HelperThread-#1
		com.mchange.v2.resourcepool.BasicResourcePool$ScatteredAcquireTask@204f2da
			on thread: C3P0PooledConnectionPoolManager[identityToken->2rvy2w9g1rvd7t51qlbfjd|74e262f6]-HelperThread-#0
	Pending Tasks: 
		com.mchange.v2.resourcepool.BasicResourcePool$ScatteredAcquireTask@1ddb0d4d
		com.mchange.v2.resourcepool.BasicResourcePool$ScatteredAcquireTask@534b5bcb
Pool thread stack traces:
	Thread[C3P0PooledConnectionPoolManager[identityToken->2rvy2w9g1rvd7t51qlbfjd|74e262f6]-HelperThread-#2,5,main]
		java.net.SocketInputStream.socketRead0(Native Method)
		java.net.SocketInputStream.socketRead(SocketInputStream.java:116)
		java.net.SocketInputStream.read(SocketInputStream.java:170)
		java.net.SocketInputStream.read(SocketInputStream.java:141)
		com.mysql.jdbc.util.ReadAheadInputStream.fill(ReadAheadInputStream.java:100)
		com.mysql.jdbc.util.ReadAheadInputStream.readFromUnderlyingStreamIfNecessary(ReadAheadInputStream.java:143)
		com.mysql.jdbc.util.ReadAheadInputStream.read(ReadAheadInputStream.java:173)
		com.mysql.jdbc.MysqlIO.readFully(MysqlIO.java:2911)
		com.mysql.jdbc.MysqlIO.readPacket(MysqlIO.java:559)
		com.mysql.jdbc.MysqlIO.doHandshake(MysqlIO.java:1013)
		com.mysql.jdbc.ConnectionImpl.coreConnect(ConnectionImpl.java:2239)
		com.mysql.jdbc.ConnectionImpl.connectOneTryOnly(ConnectionImpl.java:2270)
		com.mysql.jdbc.ConnectionImpl.createNewIO(ConnectionImpl.java:2069)
		com.mysql.jdbc.ConnectionImpl.<init>(ConnectionImpl.java:794)
		com.mysql.jdbc.JDBC4Connection.<init>(JDBC4Connection.java:44)
		sun.reflect.GeneratedConstructorAccessor186.newInstance(Unknown Source)
		sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
		java.lang.reflect.Constructor.newInstance(Constructor.java:422)
		com.mysql.jdbc.Util.handleNewInstance(Util.java:389)
		com.mysql.jdbc.ConnectionImpl.getInstance(ConnectionImpl.java:399)
		com.mysql.jdbc.NonRegisteringDriver.connect(NonRegisteringDriver.java:325)
		com.mchange.v2.c3p0.DriverManagerDataSource.getConnection(DriverManagerDataSource.java:175)
		com.mchange.v2.c3p0.WrapperConnectionPoolDataSource.getPooledConnection(WrapperConnectionPoolDataSource.java:220)
		com.mchange.v2.c3p0.WrapperConnectionPoolDataSource.getPooledConnection(WrapperConnectionPoolDataSource.java:206)
		com.mchange.v2.c3p0.impl.C3P0PooledConnectionPool$1PooledConnectionResourcePoolManager.acquireResource(C3P0PooledConnectionPool.java:203)
		com.mchange.v2.resourcepool.BasicResourcePool.doAcquire(BasicResourcePool.java:1138)
		com.mchange.v2.resourcepool.BasicResourcePool.doAcquireAndDecrementPendingAcquiresWithinLockOnSuccess(BasicResourcePool.java:1125)
		com.mchange.v2.resourcepool.BasicResourcePool.access$700(BasicResourcePool.java:44)
		com.mchange.v2.resourcepool.BasicResourcePool$ScatteredAcquireTask.run(BasicResourcePool.java:1870)
		com.mchange.v2.async.ThreadPoolAsynchronousRunner$PoolThread.run(ThreadPoolAsynchronousRunner.java:696)
	Thread[C3P0PooledConnectionPoolManager[identityToken->2rvy2w9g1rvd7t51qlbfjd|74e262f6]-HelperThread-#1,5,main]
		java.net.SocketInputStream.socketRead0(Native Method)
		java.net.SocketInputStream.socketRead(SocketInputStream.java:116)
		java.net.SocketInputStream.read(SocketInputStream.java:170)
		java.net.SocketInputStream.read(SocketInputStream.java:141)
		com.mysql.jdbc.util.ReadAheadInputStream.fill(ReadAheadInputStream.java:100)
		com.mysql.jdbc.util.ReadAheadInputStream.readFromUnderlyingStreamIfNecessary(ReadAheadInputStream.java:143)
		com.mysql.jdbc.util.ReadAheadInputStream.read(ReadAheadInputStream.java:173)
		com.mysql.jdbc.MysqlIO.readFully(MysqlIO.java:2911)
		com.mysql.jdbc.MysqlIO.readPacket(MysqlIO.java:559)
		com.mysql.jdbc.MysqlIO.doHandshake(MysqlIO.java:1013)
		com.mysql.jdbc.ConnectionImpl.coreConnect(ConnectionImpl.java:2239)
		com.mysql.jdbc.ConnectionImpl.connectOneTryOnly(ConnectionImpl.java:2270)
		com.mysql.jdbc.ConnectionImpl.createNewIO(ConnectionImpl.java:2069)
		com.mysql.jdbc.ConnectionImpl.<init>(ConnectionImpl.java:794)
		com.mysql.jdbc.JDBC4Connection.<init>(JDBC4Connection.java:44)
		sun.reflect.GeneratedConstructorAccessor186.newInstance(Unknown Source)
		sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
		java.lang.reflect.Constructor.newInstance(Constructor.java:422)
		com.mysql.jdbc.Util.handleNewInstance(Util.java:389)
		com.mysql.jdbc.ConnectionImpl.getInstance(ConnectionImpl.java:399)
		com.mysql.jdbc.NonRegisteringDriver.connect(NonRegisteringDriver.java:325)
		com.mchange.v2.c3p0.DriverManagerDataSource.getConnection(DriverManagerDataSource.java:175)
		com.mchange.v2.c3p0.WrapperConnectionPoolDataSource.getPooledConnection(WrapperConnectionPoolDataSource.java:220)
		com.mchange.v2.c3p0.WrapperConnectionPoolDataSource.getPooledConnection(WrapperConnectionPoolDataSource.java:206)
		com.mchange.v2.c3p0.impl.C3P0PooledConnectionPool$1PooledConnectionResourcePoolManager.acquireResource(C3P0PooledConnectionPool.java:203)
		com.mchange.v2.resourcepool.BasicResourcePool.doAcquire(BasicResourcePool.java:1138)
		com.mchange.v2.resourcepool.BasicResourcePool.doAcquireAndDecrementPendingAcquiresWithinLockOnSuccess(BasicResourcePool.java:1125)
		com.mchange.v2.resourcepool.BasicResourcePool.access$700(BasicResourcePool.java:44)
		com.mchange.v2.resourcepool.BasicResourcePool$ScatteredAcquireTask.run(BasicResourcePool.java:1870)
		com.mchange.v2.async.ThreadPoolAsynchronousRunner$PoolThread.run(ThreadPoolAsynchronousRunner.java:696)
	Thread[C3P0PooledConnectionPoolManager[identityToken->2rvy2w9g1rvd7t51qlbfjd|74e262f6]-HelperThread-#0,5,main]
		java.net.SocketInputStream.socketRead0(Native Method)
		java.net.SocketInputStream.socketRead(SocketInputStream.java:116)
		java.net.SocketInputStream.read(SocketInputStream.java:170)
		java.net.SocketInputStream.read(SocketInputStream.java:141)
		com.mysql.jdbc.util.ReadAheadInputStream.fill(ReadAheadInputStream.java:100)
		com.mysql.jdbc.util.ReadAheadInputStream.readFromUnderlyingStreamIfNecessary(ReadAheadInputStream.java:143)
		com.mysql.jdbc.util.ReadAheadInputStream.read(ReadAheadInputStream.java:173)
		com.mysql.jdbc.MysqlIO.readFully(MysqlIO.java:2911)
		com.mysql.jdbc.MysqlIO.readPacket(MysqlIO.java:559)
		com.mysql.jdbc.MysqlIO.doHandshake(MysqlIO.java:1013)
		com.mysql.jdbc.ConnectionImpl.coreConnect(ConnectionImpl.java:2239)
		com.mysql.jdbc.ConnectionImpl.connectOneTryOnly(ConnectionImpl.java:2270)
		com.mysql.jdbc.ConnectionImpl.createNewIO(ConnectionImpl.java:2069)
		com.mysql.jdbc.ConnectionImpl.<init>(ConnectionImpl.java:794)
		com.mysql.jdbc.JDBC4Connection.<init>(JDBC4Connection.java:44)
		sun.reflect.GeneratedConstructorAccessor186.newInstance(Unknown Source)
		sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
		java.lang.reflect.Constructor.newInstance(Constructor.java:422)
		com.mysql.jdbc.Util.handleNewInstance(Util.java:389)
		com.mysql.jdbc.ConnectionImpl.getInstance(ConnectionImpl.java:399)
		com.mysql.jdbc.NonRegisteringDriver.connect(NonRegisteringDriver.java:325)
		com.mchange.v2.c3p0.DriverManagerDataSource.getConnection(DriverManagerDataSource.java:175)
		com.mchange.v2.c3p0.WrapperConnectionPoolDataSource.getPooledConnection(WrapperConnectionPoolDataSource.java:220)
		com.mchange.v2.c3p0.WrapperConnectionPoolDataSource.getPooledConnection(WrapperConnectionPoolDataSource.java:206)
		com.mchange.v2.c3p0.impl.C3P0PooledConnectionPool$1PooledConnectionResourcePoolManager.acquireResource(C3P0PooledConnectionPool.java:203)
		com.mchange.v2.resourcepool.BasicResourcePool.doAcquire(BasicResourcePool.java:1138)
		com.mchange.v2.resourcepool.BasicResourcePool.doAcquireAndDecrementPendingAcquiresWithinLockOnSuccess(BasicResourcePool.java:1125)
		com.mchange.v2.resourcepool.BasicResourcePool.access$700(BasicResourcePool.java:44)
		com.mchange.v2.resourcepool.BasicResourcePool$ScatteredAcquireTask.run(BasicResourcePool.java:1870)
		com.mchange.v2.async.ThreadPoolAsynchronousRunner$PoolThread.run(ThreadPoolAsynchronousRunner.java:696)
		27 May 2016 06:33:26,173  WARN [C3P0PooledConnectionPoolManager[identityToken->2rvy2w9g1rvd7t51qlbfjd|74e262f6]-AdminTaskTimer] ThreadPoolAsynchronousRunner:220 - Task com.mchange.v2.resourcepool.BasicResourcePool$ScatteredAcquireTask@5383aba6 (in deadlocked PoolThread) failed to complete in maximum time 60000ms. Trying interrupt().
27 May 2016 06:33:26,173  WARN [C3P0PooledConnectionPoolManager[identityToken->2rvy2w9g1rvd7t51qlbfjd|74e262f6]-AdminTaskTimer] ThreadPoolAsynchronousRunner:220 - Task com.mchange.v2.resourcepool.BasicResourcePool$ScatteredAcquireTask@38f907f5 (in deadlocked PoolThread) failed to complete in maximum time 60000ms. Trying interrupt().
27 May 2016 06:33:26,174  WARN [C3P0PooledConnectionPoolManager[identityToken->2rvy2w9g1rvd7t51qlbfjd|74e262f6]-AdminTaskTimer] ThreadPoolAsynchronousRunner:220 - Task com.mchange.v2.resourcepool.BasicResourcePool$ScatteredAcquireTask@204f2da (in deadlocked PoolThread) failed to complete in maximum time 60000ms. Trying interrupt().

```

尝试重启后，log如下
```
27 May 2016 10:24:07,167  INFO [main] Configuration:791 - Reading password from existing file
27 May 2016 10:24:07,197  INFO [main] Configuration:1128 - Hosts Mapping File null
27 May 2016 10:24:07,197  INFO [main] HostsMap:60 - Using hostsmap file null
27 May 2016 10:24:08,493  INFO [main] ControllerModule:195 - Detected MYSQL as the database type from the JDBC URL
27 May 2016 10:24:08,522  INFO [main] ControllerModule:237 - Using c3p0 ComboPooledDataSource as the EclipsLink DataSource
27 May 2016 10:24:08,582  INFO [MLog-Init-Reporter] MLog:212 - MLog clients using slf4j logging.
27 May 2016 10:24:08,963  INFO [main] C3P0Registry:212 - Initializing c3p0-0.9.5.2 [built 08-December-2015 22:06:04 -0800; debug? true; trace: 10]
27 May 2016 10:24:11,656  INFO [main] ControllerModule:596 - Binding and registering notification dispatcher class org.apache.ambari.server.notifications.dispatchers.SNMPDispatcher
27 May 2016 10:24:11,665  INFO [main] ControllerModule:596 - Binding and registering notification dispatcher class org.apache.ambari.server.notifications.dispatchers.AlertScriptDispatcher
```


想了一下会不会是mysql的问题，然后就尝试连接一下mysql —— 端口通，但是连接不上。找运维看了一下，mysql所在的机器的cpu爆满，有一个线程给打满了。

运维重启了，昨天集群跑任务的时候也是资源没有变，但是cpu被打满，以往同样的资源根本不会耗时那么多。个人怀疑是被人攻击了。



hive meta库重启之后，通过beeline是可以成功连接hive并查询的。但是hive命令行还是进不去，卡在:
```
[hive@datanode01 root]$ hive
WARNING: Use "yarn jar" to launch YARN applications.

Logging initialized using configuration in file:/etc/hive/2.4.2.0-258/0/hive-log4j.properties
```
查看hive的log:
```
2016-05-27 11:33:14,153 INFO  [main]: SessionState (SessionState.java:printInfo(953)) - 
Logging initialized using configuration in file:/etc/hive/2.4.2.0-258/0/hive-log4j.properties
2016-05-27 11:33:14,653 INFO  [main]: hive.metastore (HiveMetaStoreClient.java:open(382)) - Trying to connect to metastore with URI thrift://datanode03.will.com:9083
2016-05-27 11:33:14,736 INFO  [main]: hive.metastore (HiveMetaStoreClient.java:open(478)) - Connected to metastore.
```
正常命令行log：
```
2016-05-19 10:40:59,299 INFO  [main]: SessionState (SessionState.java:printInfo(953)) -
Logging initialized using configuration in file:/etc/hive/2.4.2.0-258/0/hive-log4j.properties
2016-05-19 10:41:00,519 INFO  [main]: hive.metastore (HiveMetaStoreClient.java:open(382)) - Trying to connect to metastore with URI thrift://datanode03.will.com:9083
2016-05-19 10:41:01,125 INFO  [main]: hive.metastore (HiveMetaStoreClient.java:open(478)) - Connected to metastore.
2016-05-19 10:41:04,652 INFO  [main]: session.SessionState (SessionState.java:createPath(633)) - Created local directory: /tmp/1d1a2120-6339-40c1-b63b-5dbd4dc71074_resources
2016-05-19 10:41:04,664 INFO  [main]: session.SessionState (SessionState.java:createPath(633)) - Created HDFS directory: /tmp/hive/hive/1d1a2120-6339-40c1-b63b-5dbd4dc71074
2016-05-19 10:41:04,666 INFO  [main]: session.SessionState (SessionState.java:createPath(633)) - Created local directory: /tmp/hive/1d1a2120-6339-40c1-b63b-5dbd4dc71074
2016-05-19 10:41:04,669 INFO  [main]: session.SessionState (SessionState.java:createPath(633)) - Created HDFS directory: /tmp/hive/hive/1d1a2120-6339-40c1-b63b-5dbd4dc71074/_tmp_space.db
2016-05-19 10:41:04,717 INFO  [main]: sqlstd.SQLStdHiveAccessController (SQLStdHiveAccessController.java:<init>(95)) - Created SQLStdHiveAccessController for session context : HiveAuthzSessionContext [sessionString=1d1a2120-6339-40c1-b63b-5dbd4dc71074, clientType=HIVECLI]
2016-05-19 10:41:04,720 INFO  [main]: hive.metastore (HiveMetaStoreClient.java:isCompatibleWith(296)) - Mestastore configuration hive.metastore.filter.hook changed from org.apache.hadoop.hive.metastore.DefaultMetaStoreFilterHookImpl to org.apache.hadoop.hive.ql.security.authorization.plugin.AuthorizationMetaStoreFilterHook
2016-05-19 11:21:13,947 INFO  [main]: hive.metastore (HiveMetaStoreClient.java:open(382)) - Trying to connect to metastore with URI thrift://datanode03.will.com:9083
2016-05-19 11:21:13,951 INFO  [main]: hive.metastore (HiveMetaStoreClient.java:open(478)) - Connected to metastore.
2016-05-19 11:21:14,674 INFO  [main]: SessionState (SessionState.java:printInfo(953)) - Added [/server/app/hive/will_hive_udf.jar] to class path
2016-05-19 11:21:14,674 INFO  [main]: SessionState (SessionState.java:printInfo(953)) - Added resources: [/server/app/hive/will_hive_udf.jar]
2016-05-19 11:21:14,858 INFO  [main]: log.PerfLogger (PerfLogger.java:PerfLogBegin(135)) - <PERFLOG method=Driver.run from=org.apache.hadoop.hive.ql.Driver>
2016-05-19 11:21:14,858 INFO  [main]: log.PerfLogger (PerfLogger.java:PerfLogBegin(135)) - <PERFLOG method=TimeToSubmit from=org.apache.hadoop.hive.ql.Driver>
2016-05-19 11:21:14,858 INFO  [main]: log.PerfLogger (PerfLogger.java:PerfLogBegin(135)) - <PERFLOG method=compile from=org.apache.hadoop.hive.ql.Driver>
2016-05-19 11:21:14,923 INFO  [main]: ql.Driver (Driver.java:compile(420)) - We are setting the hadoop caller context from  to hive_20160519112114_e4649b5a-c094-40ff-a9b1-aa6ab595c967
2016-05-19 11:21:14,928 INFO  [main]: log.PerfLogger (PerfLogger.java:PerfLogBegin(135)) - <PERFLOG method=parse from=org.apache.hadoop.hive.ql.Driver>
2016-05-19 11:21:14,937 INFO  [main]: parse.ParseDriver (ParseDriver.java:parse(185)) - Parsing command: 
create temporary function  strstart as 'com.will.common.hive.StrStart'
2016-05-19 11:21:16,408 INFO  [main]: parse.ParseDriver (ParseDriver.java:parse(209)) - Parse Completed
2016-05-19 11:21:16,411 INFO  [main]: log.PerfLogger (PerfLogger.java:PerfLogEnd(162)) - </PERFLOG method=parse start=1463628074928 end=1463628076411 duration=1483 from=org.apache.hadoop.hi
ve.ql.Driver>
2016-05-19 11:21:16,412 INFO  [main]: log.PerfLogger (PerfLogger.java:PerfLogBegin(135)) - <PERFLOG method=semanticAnalyze from=org.apache.hadoop.hive.ql.Driver>
2016-05-19 11:21:17,249 INFO  [main]: parse.FunctionSemanticAnalyzer (FunctionSemanticAnalyzer.java:analyzeInternal(66)) - analyze done
2016-05-19 11:21:17,249 INFO  [main]: ql.Driver (Driver.java:compile(471)) - Semantic Analysis Completed


```

可以看到是createPath那卡住了，建立临时会话相关文件。同事那边要得急，就直接把hive metastore重启了，就恢复正常了。
这样看来如果hive metastore和mysql meta库失去连接的话，即便mysql恢复正常，启动hive命令行的时候，HiveMetaStoreClient也会有问题。


**这个可以修复一下，然后commit给hive官网。。。**
