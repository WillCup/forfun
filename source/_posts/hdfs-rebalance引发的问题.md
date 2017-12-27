
title: hdfs-rebalance引发的问题
date: 2016-12-23 18:08:23
tags: [youdaonote]
---

原来的集群节点是8台datanode.......HDFS的使用总量已经到了

```
 Slow waitForAckedSeqno took 179279ms (threshold=30000ms)
```


```
 Bad response ERROR for block BP-1029563541-10.0.1.72-1463106887558:blk_1077490973_3758370 from datanode DatanodeInfoWithStorage[10.0.1.98:50010,DS-a387140d-7
```

```
Got error, status message , ack with firstBadLink as 10.0.1.101:50010
```


```
java.io.IOException: Bad response ERROR for block BP-1029563541-10.0.1.72-1463106887558:blk_1077490815_3758189 from datanode DatanodeInfoWithStorage[10.0.1.101:50010,DS-8786adb7-d8ae-4b29-a8af-4cb73a8a5e72,DISK]
	at org.apache.hadoop.hdfs.DFSOutputStream$DataStreamer$ResponseProcessor.run(DFSOutputStream.java:785)
```
	
	
map失败log信息：
```
2016-12-24 22:19:01,865 WARN [main] org.apache.hadoop.metrics2.impl.MetricsConfig: Cannot locate configuration: tried hadoop-metrics2-maptask.properties,hadoop-metrics2.properties
2016-12-24 22:19:05,431 INFO [main] org.apache.hadoop.metrics2.impl.MetricsSystemImpl: Scheduled snapshot period at 10 second(s).
2016-12-24 22:19:05,431 INFO [main] org.apache.hadoop.metrics2.impl.MetricsSystemImpl: MapTask metrics system started
2016-12-24 22:19:05,442 INFO [main] org.apache.hadoop.mapred.YarnChild: Executing with tokens:
2016-12-24 22:19:05,442 INFO [main] org.apache.hadoop.mapred.YarnChild: Kind: mapreduce.job, Service: job_1482504625052_0529, Ident: (org.apache.hadoop.mapreduce.security.token.JobTokenIdentifier@57a78e3)
2016-12-24 22:20:33,408 INFO [main] org.apache.hadoop.mapred.YarnChild: Sleeping for 0ms before retrying again. Got null now.
2016-12-24 22:20:46,112 INFO [main] org.apache.hadoop.metrics2.impl.MetricsSystemImpl: Stopping MapTask metrics system...
2016-12-24 22:20:46,119 INFO [main] org.apache.hadoop.metrics2.impl.MetricsSystemImpl: MapTask metrics system stopped.
2016-12-24 22:20:46,119 INFO [main] org.apache.hadoop.metrics2.impl.MetricsSystemImpl: MapTask metrics system shutdown complete.
```



MR任务之间有一段空白期
```
2016-12-24 21:29:50,046 Stage-1 map = 100%, reduce = 99%, Cumulative CPU 4516.43 sec
2016-12-24 21:30:04,499 Stage-1 map = 100%, reduce = 100%, Cumulative CPU 4528.6 sec
MapReduce Total cumulative CPU time: 0 days 1 hours 15 minutes 28 seconds 600 msec
Ended Job = job_1482504625052_0527
Launching Job 2 out of 3
Number of reduce tasks not specified. Defaulting to jobconf value of: 30
In order to change the average load for a reducer (in bytes):
set hive.exec.reducers.bytes.per.reducer=
In order to limit the maximum number of reducers:
set hive.exec.reducers.max=
In order to set a constant number of reducers:
set mapreduce.job.reduces=
Starting Job = job_1482504625052_0528, Tracking URL = http://datanode02.will.com:8088/proxy/application_1482504625052_0528/
Kill Command = /usr/hdp/2.4.2.0-258/hadoop/bin/hadoop job -kill job_1482504625052_0528
Hadoop job information for Stage-5: number of mappers: 7; number of reducers: 30
2016-12-24 21:54:09,652 Stage-5 map = 0%, reduce = 0%
2016-12-24 21:54:19,037 Stage-5 map = 14%, reduce = 0%, Cumulative CPU 7.53 sec
2016-12-24 21:54:20,077 Stage-5 map = 29%, reduce = 0%, Cumulative CPU 15.38 sec
2016-12-24 21:54:36,677 Stage-5 map = 39%, reduce = 0%, Cumulative CPU 44.43 sec
2016-12-24 21:54:37,759 Stage-5 map = 71%, reduce = 0%, Cumulative CPU 52.41 sec
2016-12-24 21:54:48,153 Stage-5 map = 71%, reduce = 1%, Cumulative CPU 53.96 sec
2016-12-24 21:54:49,186 Stage-5 map = 71%, reduce = 2%, Cumulative CPU 56.57 sec
```
21:30到21:54,这段时间namenode的log里的一些内容：
```
2016-12-24 21:49:01,939 INFO  hdfs.StateChange (FSNamesystem.java:completeFile(3545)) - DIR* completeFile: /apps/hbase/data/data/default/KYLIN_U1V82JA44K/8c474006d2f4332bdaaee58105bb465a/.tmp/79460c3b99ba48f8b67cfadba06d277e is closed by DFSClient_NONMAPREDUCE_-90235743_1
2016-12-24 21:49:02,045 INFO  hdfs.StateChange (FSNamesystem.java:logAllocatedBlock(3652)) - BLOCK* allocate blk_1077490355_3757707, replicas=10.0.1.86:50010, 10.0.1.95:50010, 10.0.1.84:50010 for /apps/hbase/data/data/default/KYLIN_U1V82JA44K/9674d0c2aec6aa86cb092088d9461163/.tmp/7d5802bddb694c25a84cdd2263bcca42


2016-12-24 21:42:49,930 INFO  hdfs.StateChange (FSNamesystem.java:completeFile(3545)) - DIR* completeFile: /apps/hbase/data/data/default/KYLIN_V5P8H4CZJF/2c1b15e259130ca37d1c4808aade03e6/.tmp/d702471e4c784abba520d494c29b03f2 is closed by DFSClient_NONMAPREDUCE_434452785_1


namenode.FSNamesystem (FSNamesystem.java:updatePipeline(6535)) - updatePipeline(blk_1077490302_3757650, newGS=3757651, newLength=114137522, newNodes=[10.0.1.96:50010, 10.0.1.83:50010], client=DFSClient_NONMAPREDUCE_829342269_1)
2016-12-24 21:42:49,567 INFO  namenode.FSNamesystem (FSNamesystem.java:updatePipeline(6554)) - updatePipeline(blk_1077490302_3757650 => blk_1077490302_3757651) success


chooseUnderReplicatedBlocks selected 1 blocks at priority level 2;  Total=1 Reset bookmarks? true

namenode.FSEditLog (FSEditLog.java:printStatistics(699)) - Number of transactions: 106401 Total time fo
r transactions(ms): 961 Number of transactions batched in Syncs: 3666 Number of syncs: 87634 SyncTimes(ms): 239376

```


failed的reducer的提示note：
```
AttemptID:attempt_1482504625052_0537_r_000003_0 Timed out after 300 secs Container killed by the ApplicationMaster. Container killed on request. Exit code is 143 Container exited with a non-zero exit code 143
```



balance完成之后，运行了几天，每天的任务从2小时、1个小时40分钟、1小时30分钟、1小时20分钟递减~~也很少出现这种问题了......诡异

参考下面的资料，基本思想就是timeout的时间内，没有完成某个任务，所以把timeout延长就可以了。还没有实践测试....就好了

还是看一下这附近的源码比较好一些~

参考：
- http://www.superwu.cn/2014/11/13/1334/
- http://blog.sina.com.cn/s/blog_72827fb1010198j7.html
- http://blog.163.com/zhengjiu_520/blog/static/3559830620130743644473/
