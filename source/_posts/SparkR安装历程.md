
title: SparkR安装历程
date: 2017-01-09 10:35:52
tags: [youdaonote]
---

1. 在centos上安装一下R的基础包。
```
yum install -y R

R CMD javareconf
```
2. 安装Rstudio server
```
wget https://download2.rstudio.org/rstudio-server-rhel-1.0.136-x86_64.rpm
yum install --nogpgcheck rstudio-server-rhel-1.0.136-x86_64.rpm
rstudio-server verify-installation
```
3. 为Rstudio server添加用户，默认是使用linux用户的
```
useradd rtest
passwd rtest
```

4. R安装依赖
为R安装sparkR的库
```
R CMD INSTALL /usr/hdp/current/spark-client/R/lib/SparkR/

```

安装forecast的时候报错, g++版本低，通过安装devtools-2解决：
```
sudo rpm --import http://ftp.scientificlinux.org/linux/scientific/5x/x86_64/RPM-GPG-KEYs/RPM-GPG-KEY-cern
wget -O /etc/yum.repos.d/slc6-devtoolset.repo http://linuxsoft.cern.ch/cern/devtoolset/slc6-devtoolset.repo
sudo yum install devtoolset-2
scl enable devtoolset-2 bash
```
这样版本就已经好了。
```
$ gcc --version
gcc (GCC) 4.8.2 20140120 (Red Hat 4.8.2-15)
...

$ g++ --version
g++ (GCC) 4.8.2 20140120 (Red Hat 4.8.2-15)
...

$ gfortran --version
GNU Fortran (GCC) 4.8.2 20140120 (Red Hat 4.8.2-15)
...
```

参考： https://gist.github.com/stephenturner/e3bc5cfacc2dc67eca8b


```
> sc<-sparkR.init(master = "yarn-client")
Launching java with spark-submit command spark-submit   sparkr-shell /tmp/RtmpERrwOX/backend_port628c57c68104 
17/01/10 15:28:50 INFO SparkContext: Running Spark version 1.5.2
17/01/10 15:28:53 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
17/01/10 15:28:54 INFO SecurityManager: Changing view acls to: zhangjiayi
17/01/10 15:28:54 INFO SecurityManager: Changing modify acls to: zhangjiayi
17/01/10 15:28:54 INFO SecurityManager: SecurityManager: authentication disabled; ui acls disabled; users with view permissions: Set(zhangjiayi); users with modify permissions: Set(zhangjiayi)
17/01/10 15:29:00 INFO Slf4jLogger: Slf4jLogger started
17/01/10 15:29:00 INFO Remoting: Starting remoting
17/01/10 15:29:01 INFO Remoting: Remoting started; listening on addresses :[akka.tcp://sparkDriver@10.1.5.65:34109]
17/01/10 15:29:01 INFO Utils: Successfully started service 'sparkDriver' on port 34109.
17/01/10 15:29:01 INFO SparkEnv: Registering MapOutputTracker
17/01/10 15:29:01 INFO SparkEnv: Registering BlockManagerMaster
17/01/10 15:29:02 INFO DiskBlockManager: Created local directory at /tmp/blockmgr-ce31222a-c674-4391-be3c-43b075e758d4
17/01/10 15:29:02 INFO MemoryStore: MemoryStore started with capacity 530.0 MB
17/01/10 15:29:02 INFO HttpFileServer: HTTP File server directory is /tmp/spark-c2196ae2-3c1b-46e0-97dd-583cd7a240a3/httpd-af4e2720-d455-4d5a-93de-6a0e7231da34
17/01/10 15:29:02 INFO HttpServer: Starting HTTP Server
17/01/10 15:29:03 INFO Server: jetty-8.y.z-SNAPSHOT
17/01/10 15:29:03 INFO AbstractConnector: Started SocketConnector@0.0.0.0:44708
17/01/10 15:29:03 INFO Utils: Successfully started service 'HTTP file server' on port 44708.
17/01/10 15:29:03 INFO SparkEnv: Registering OutputCommitCoordinator
17/01/10 15:29:03 INFO Server: jetty-8.y.z-SNAPSHOT
17/01/10 15:29:03 INFO AbstractConnector: Started SelectChannelConnector@0.0.0.0:4040
17/01/10 15:29:03 INFO Utils: Successfully started service 'SparkUI' on port 4040.
17/01/10 15:29:03 INFO SparkUI: Started SparkUI at http://10.1.5.65:4040
17/01/10 15:29:04 WARN MetricsSystem: Using default name DAGScheduler for source because spark.app.id is not set.
spark.yarn.driver.memoryOverhead is set but does not apply in client mode.
17/01/10 15:29:05 INFO TimelineClientImpl: Timeline service address: http://slavenode4:8188/ws/v1/timeline/
17/01/10 15:29:06 INFO RMProxy: Connecting to ResourceManager at slavenode3/10.1.5.64:8050
17/01/10 15:29:11 WARN DomainSocketFactory: The short-circuit local reads feature cannot be used because libhadoop cannot be loaded.
17/01/10 15:29:12 INFO Client: Requesting a new application from cluster with 5 NodeManagers
17/01/10 15:29:12 INFO Client: Verifying our application has not requested more than the maximum memory capability of the cluster (5120 MB per container)
17/01/10 15:29:12 INFO Client: Will allocate AM container, with 896 MB memory including 384 MB overhead
17/01/10 15:29:12 INFO Client: Setting up container launch context for our AM
17/01/10 15:29:12 INFO Client: Setting up the launch environment for our AM container
17/01/10 15:29:12 INFO Client: Preparing resources for our AM container
17/01/10 15:29:12 INFO Client: Uploading resource file:/usr/hdp/2.3.4.0-3485/spark/lib/spark-assembly-1.5.2.2.3.4.0-3485-hadoop2.7.1.2.3.4.0-3485.jar -> hdfs://masternode:8020/user/zhangjiayi/.sparkStaging/application_1483512430911_0002/spark-assembly-1.5.2.2.3.4.0-3485-hadoop2.7.1.2.3.4.0-3485.jar
17/01/10 15:29:22 INFO Client: Uploading resource file:/tmp/spark-c2196ae2-3c1b-46e0-97dd-583cd7a240a3/__spark_conf__1393429344923620787.zip -> hdfs://masternode:8020/user/zhangjiayi/.sparkStaging/application_1483512430911_0002/__spark_conf__1393429344923620787.zip
17/01/10 15:29:22 INFO SecurityManager: Changing view acls to: zhangjiayi
17/01/10 15:29:22 INFO SecurityManager: Changing modify acls to: zhangjiayi
17/01/10 15:29:22 INFO SecurityManager: SecurityManager: authentication disabled; ui acls disabled; users with view permissions: Set(zhangjiayi); users with modify permissions: Set(zhangjiayi)
17/01/10 15:29:22 INFO Client: Submitting application 2 to ResourceManager
17/01/10 15:29:22 INFO YarnClientImpl: Submitted application application_1483512430911_0002
17/01/10 15:29:22 INFO YarnExtensionServices: Starting Yarn extension services with app application_1483512430911_0002 and attemptId None
17/01/10 15:29:22 INFO YarnHistoryService: Starting YarnHistoryService for application application_1483512430911_0002 attempt None; state=1; endpoint=http://slavenode4:8188/ws/v1/timeline/; bonded to ATS=false; listening=false; batchSize=10; flush count=0; total number queued=0, processed=0; attempted entity posts=0 successful entity posts=0 failed entity posts=0; events dropped=0; app start event received=false; app end event received=false;
17/01/10 15:29:22 INFO YarnHistoryService: Spark events will be published to the Timeline service at http://slavenode4:8188/ws/v1/timeline/
17/01/10 15:29:22 INFO TimelineClientImpl: Timeline service address: http://slavenode4:8188/ws/v1/timeline/
17/01/10 15:29:22 INFO YarnHistoryService: History Service listening for events: YarnHistoryService for application application_1483512430911_0002 attempt None; state=1; endpoint=http://slavenode4:8188/ws/v1/timeline/; bonded to ATS=true; listening=true; batchSize=10; flush count=0; total number queued=0, processed=0; attempted entity posts=0 successful entity posts=0 failed entity posts=0; events dropped=0; app start event received=false; app end event received=false;
17/01/10 15:29:22 INFO YarnExtensionServices: Service org.apache.spark.deploy.yarn.history.YarnHistoryService started
17/01/10 15:29:23 INFO Client: Application report for application_1483512430911_0002 (state: ACCEPTED)
17/01/10 15:29:23 INFO Client: 
	 client token: N/A
	 diagnostics: N/A
	 ApplicationMaster host: N/A
	 ApplicationMaster RPC port: -1
	 queue: default
	 start time: 1484033362415
	 final status: UNDEFINED
	 tracking URL: http://slavenode3:8088/proxy/application_1483512430911_0002/
	 user: zhangjiayi
17/01/10 15:29:24 INFO Client: Application report for application_1483512430911_0002 (state: ACCEPTED)
17/01/10 15:29:25 INFO Client: Application report for application_1483512430911_0002 (state: ACCEPTED)
17/01/10 15:29:26 INFO Client: Application report for application_1483512430911_0002 (state: ACCEPTED)
17/01/10 15:29:27 INFO YarnSchedulerBackend$YarnSchedulerEndpoint: ApplicationMaster registered as AkkaRpcEndpointRef(Actor[akka.tcp://sparkYarnAM@10.1.5.66:34209/user/YarnAM#-468278157])
17/01/10 15:29:27 INFO YarnClientSchedulerBackend: Add WebUI Filter. org.apache.hadoop.yarn.server.webproxy.amfilter.AmIpFilter, Map(PROXY_HOSTS -> slavenode3, PROXY_URI_BASES -> http://slavenode3:8088/proxy/application_1483512430911_0002), /proxy/application_1483512430911_0002
17/01/10 15:29:27 INFO JettyUtils: Adding filter: org.apache.hadoop.yarn.server.webproxy.amfilter.AmIpFilter
17/01/10 15:29:27 INFO Client: Application report for application_1483512430911_0002 (state: RUNNING)
17/01/10 15:29:27 INFO Client: 
	 client token: N/A
	 diagnostics: N/A
	 ApplicationMaster host: 10.1.5.66
	 ApplicationMaster RPC port: 0
	 queue: default
	 start time: 1484033362415
	 final status: UNDEFINED
	 tracking URL: http://slavenode3:8088/proxy/application_1483512430911_0002/
	 user: zhangjiayi
17/01/10 15:29:27 INFO YarnClientSchedulerBackend: Application application_1483512430911_0002 has started running.
17/01/10 15:29:27 INFO Utils: Successfully started service 'org.apache.spark.network.netty.NettyBlockTransferService' on port 44133.
17/01/10 15:29:27 INFO NettyBlockTransferService: Server created on 44133
17/01/10 15:29:27 INFO BlockManagerMaster: Trying to register BlockManager
17/01/10 15:29:27 INFO BlockManagerMasterEndpoint: Registering block manager 10.1.5.65:44133 with 530.0 MB RAM, BlockManagerId(driver, 10.1.5.65, 44133)
17/01/10 15:29:27 INFO BlockManagerMaster: Registered BlockManager
17/01/10 15:29:28 INFO YarnHistoryService: Application started: SparkListenerApplicationStart(SparkR,Some(application_1483512430911_0002),1484033330463,zhangjiayi,None,None)
17/01/10 15:29:28 INFO YarnHistoryService: About to POST entity application_1483512430911_0002 with 3 events to timeline service http://slavenode4:8188/ws/v1/timeline/
17/01/10 15:29:34 INFO YarnClientSchedulerBackend: SchedulerBackend is ready for scheduling beginning after waiting maxRegisteredResourcesWaitingTime: 30000(ms)
> 
17/01/10 15:29:35 INFO YarnClientSchedulerBackend: Registered executor: AkkaRpcEndpointRef(Actor[akka.tcp://sparkExecutor@slavenode3:41731/user/Executor#207450695]) with ID 1
17/01/10 15:29:35 INFO BlockManagerMasterEndpoint: Registering block manager slavenode3:45665 with 530.0 MB RAM, BlockManagerId(1, slavenode3, 45665)
17/01/10 15:29:39 INFO YarnClientSchedulerBackend: Registered executor: AkkaRpcEndpointRef(Actor[akka.tcp://sparkExecutor@slavenode2:42294/user/Executor#1616446929]) with ID 2
17/01/10 15:29:39 INFO BlockManagerMasterEndpoint: Registering block manager slavenode2:42279 with 530.0 MB RAM, BlockManagerId(2, slavenode2, 42279)
> sqlcontext<-sparkRSQL.init(sc)
> df<-createDataFrame(sqlContext = sqlcontext,data = iris)
17/01/10 15:29:54 INFO SparkContext: Starting job: collectPartitions at NativeMethodAccessorImpl.java:-2
17/01/10 15:29:54 INFO DAGScheduler: Got job 0 (collectPartitions at NativeMethodAccessorImpl.java:-2) with 1 output partitions
17/01/10 15:29:54 INFO DAGScheduler: Final stage: ResultStage 0(collectPartitions at NativeMethodAccessorImpl.java:-2)
17/01/10 15:29:54 INFO DAGScheduler: Parents of final stage: List()
17/01/10 15:29:54 INFO DAGScheduler: Missing parents: List()
17/01/10 15:29:54 INFO DAGScheduler: Submitting ResultStage 0 (ParallelCollectionRDD[0] at parallelize at RRDD.scala:454), which has no missing parents
17/01/10 15:30:03 INFO MemoryStore: ensureFreeSpace(1280) called with curMem=0, maxMem=555755765
17/01/10 15:30:03 INFO MemoryStore: Block broadcast_0 stored as values in memory (estimated size 1280.0 B, free 530.0 MB)
17/01/10 15:30:03 INFO MemoryStore: ensureFreeSpace(854) called with curMem=1280, maxMem=555755765
17/01/10 15:30:03 INFO MemoryStore: Block broadcast_0_piece0 stored as bytes in memory (estimated size 854.0 B, free 530.0 MB)
17/01/10 15:30:03 INFO BlockManagerInfo: Added broadcast_0_piece0 in memory on 10.1.5.65:44133 (size: 854.0 B, free: 530.0 MB)
17/01/10 15:30:03 INFO SparkContext: Created broadcast 0 from broadcast at DAGScheduler.scala:861
17/01/10 15:30:03 INFO DAGScheduler: Submitting 1 missing tasks from ResultStage 0 (ParallelCollectionRDD[0] at parallelize at RRDD.scala:454)
17/01/10 15:30:03 INFO YarnScheduler: Adding task set 0.0 with 1 tasks
17/01/10 15:30:03 INFO TaskSetManager: Starting task 0.0 in stage 0.0 (TID 0, slavenode2, PROCESS_LOCAL, 16553 bytes)
17/01/10 15:30:05 INFO BlockManagerInfo: Added broadcast_0_piece0 in memory on slavenode2:42279 (size: 854.0 B, free: 530.0 MB)
17/01/10 15:30:05 INFO TaskSetManager: Finished task 0.0 in stage 0.0 (TID 0) in 1661 ms on slavenode2 (1/1)
17/01/10 15:30:05 INFO DAGScheduler: ResultStage 0 (collectPartitions at NativeMethodAccessorImpl.java:-2) finished in 1.786 s
17/01/10 15:30:05 INFO DAGScheduler: Job 0 finished: collectPartitions at NativeMethodAccessorImpl.java:-2, took 11.381818 s
17/01/10 15:30:05 INFO YarnScheduler: Removed TaskSet 0.0, whose tasks have all completed, from pool 
17/01/10 15:30:06 INFO YarnHistoryService: About to POST entity application_1483512430911_0002 with 10 events to timeline service http://slavenode4:8188/ws/v1/timeline/
Warning messages:
1: In FUN(X[[i]], ...) :
  Use Sepal_Length instead of Sepal.Length  as column name
2: In FUN(X[[i]], ...) :
  Use Sepal_Width instead of Sepal.Width  as column name
3: In FUN(X[[i]], ...) :
  Use Petal_Length instead of Petal.Length  as column name
4: In FUN(X[[i]], ...) :
  Use Petal_Width instead of Petal.Width  as column name
> 
> typeof(df)
[1] "S4"
> a<-'sdfsd'
> typeof(a)
[1] "character"
> head(df)
17/01/10 15:31:24 INFO SparkContext: Starting job: dfToCols at NativeMethodAccessorImpl.java:-2
17/01/10 15:31:24 INFO DAGScheduler: Got job 1 (dfToCols at NativeMethodAccessorImpl.java:-2) with 1 output partitions
17/01/10 15:31:24 INFO DAGScheduler: Final stage: ResultStage 1(dfToCols at NativeMethodAccessorImpl.java:-2)
17/01/10 15:31:24 INFO DAGScheduler: Parents of final stage: List()
17/01/10 15:31:24 INFO DAGScheduler: Missing parents: List()
17/01/10 15:31:24 INFO DAGScheduler: Submitting ResultStage 1 (MapPartitionsRDD[4] at dfToCols at NativeMethodAccessorImpl.java:-2), which has no missing parents
17/01/10 15:31:24 INFO MemoryStore: ensureFreeSpace(9176) called with curMem=2134, maxMem=555755765
17/01/10 15:31:24 INFO MemoryStore: Block broadcast_1 stored as values in memory (estimated size 9.0 KB, free 530.0 MB)
17/01/10 15:31:24 INFO MemoryStore: ensureFreeSpace(3690) called with curMem=11310, maxMem=555755765
17/01/10 15:31:24 INFO MemoryStore: Block broadcast_1_piece0 stored as bytes in memory (estimated size 3.6 KB, free 530.0 MB)
17/01/10 15:31:24 INFO BlockManagerInfo: Added broadcast_1_piece0 in memory on 10.1.5.65:44133 (size: 3.6 KB, free: 530.0 MB)
17/01/10 15:31:24 INFO SparkContext: Created broadcast 1 from broadcast at DAGScheduler.scala:861
17/01/10 15:31:24 INFO DAGScheduler: Submitting 1 missing tasks from ResultStage 1 (MapPartitionsRDD[4] at dfToCols at NativeMethodAccessorImpl.java:-2)
17/01/10 15:31:24 INFO YarnScheduler: Adding task set 1.0 with 1 tasks
17/01/10 15:31:24 INFO TaskSetManager: Starting task 0.0 in stage 1.0 (TID 1, slavenode3, PROCESS_LOCAL, 16553 bytes)
17/01/10 15:31:24 INFO BlockManagerInfo: Added broadcast_1_piece0 in memory on slavenode3:45665 (size: 3.6 KB, free: 530.0 MB)
17/01/10 15:31:35 WARN TaskSetManager: Lost task 0.0 in stage 1.0 (TID 1, slavenode3): java.net.SocketTimeoutException: Accept timed out
	at java.net.PlainSocketImpl.socketAccept(Native Method)
	at java.net.AbstractPlainSocketImpl.accept(AbstractPlainSocketImpl.java:409)
	at java.net.ServerSocket.implAccept(ServerSocket.java:545)
	at java.net.ServerSocket.accept(ServerSocket.java:513)
	at org.apache.spark.api.r.RRDD$.createRWorker(RRDD.scala:426)
	at org.apache.spark.api.r.BaseRRDD.compute(RRDD.scala:62)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
	at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
	at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
	at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
	at org.apache.spark.scheduler.ResultTask.runTask(ResultTask.scala:66)
	at org.apache.spark.scheduler.Task.run(Task.scala:88)
	at org.apache.spark.executor.Executor$TaskRunner.run(Executor.scala:214)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at java.lang.Thread.run(Thread.java:745)

17/01/10 15:31:35 INFO TaskSetManager: Starting task 0.1 in stage 1.0 (TID 2, slavenode2, PROCESS_LOCAL, 16553 bytes)
17/01/10 15:31:35 INFO BlockManagerInfo: Added broadcast_1_piece0 in memory on slavenode2:42279 (size: 3.6 KB, free: 530.0 MB)
17/01/10 15:31:37 WARN TaskSetManager: Lost task 0.1 in stage 1.0 (TID 2, slavenode2): java.io.IOException: Cannot run program "Rscript": error=13, Permission denied
	at java.lang.ProcessBuilder.start(ProcessBuilder.java:1048)
	at org.apache.spark.api.r.RRDD$.createRProcess(RRDD.scala:407)
	at org.apache.spark.api.r.RRDD$.createRWorker(RRDD.scala:423)
	at org.apache.spark.api.r.BaseRRDD.compute(RRDD.scala:62)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
	at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
	at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
	at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
	at org.apache.spark.scheduler.ResultTask.runTask(ResultTask.scala:66)
	at org.apache.spark.scheduler.Task.run(Task.scala:88)
	at org.apache.spark.executor.Executor$TaskRunner.run(Executor.scala:214)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at java.lang.Thread.run(Thread.java:745)
Caused by: java.io.IOException: error=13, Permission denied
	at java.lang.UNIXProcess.forkAndExec(Native Method)
	at java.lang.UNIXProcess.<init>(UNIXProcess.java:248)
	at java.lang.ProcessImpl.start(ProcessImpl.java:134)
	at java.lang.ProcessBuilder.start(ProcessBuilder.java:1029)
	... 20 more

17/01/10 15:31:37 INFO TaskSetManager: Starting task 0.2 in stage 1.0 (TID 3, slavenode3, PROCESS_LOCAL, 16553 bytes)
17/01/10 15:31:47 WARN TaskSetManager: Lost task 0.2 in stage 1.0 (TID 3, slavenode3): java.net.SocketTimeoutException: Accept timed out
	at java.net.PlainSocketImpl.socketAccept(Native Method)
	at java.net.AbstractPlainSocketImpl.accept(AbstractPlainSocketImpl.java:409)
	at java.net.ServerSocket.implAccept(ServerSocket.java:545)
	at java.net.ServerSocket.accept(ServerSocket.java:513)
	at org.apache.spark.api.r.RRDD$.createRWorker(RRDD.scala:426)
	at org.apache.spark.api.r.BaseRRDD.compute(RRDD.scala:62)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
	at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
	at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
	at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
	at org.apache.spark.scheduler.ResultTask.runTask(ResultTask.scala:66)
	at org.apache.spark.scheduler.Task.run(Task.scala:88)
	at org.apache.spark.executor.Executor$TaskRunner.run(Executor.scala:214)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at java.lang.Thread.run(Thread.java:745)

17/01/10 15:31:47 INFO TaskSetManager: Starting task 0.3 in stage 1.0 (TID 4, slavenode2, PROCESS_LOCAL, 16553 bytes)
17/01/10 15:31:47 WARN TaskSetManager: Lost task 0.3 in stage 1.0 (TID 4, slavenode2): java.io.IOException: Cannot run program "Rscript": error=13, Permission denied
	at java.lang.ProcessBuilder.start(ProcessBuilder.java:1048)
	at org.apache.spark.api.r.RRDD$.createRProcess(RRDD.scala:407)
	at org.apache.spark.api.r.RRDD$.createRWorker(RRDD.scala:423)
	at org.apache.spark.api.r.BaseRRDD.compute(RRDD.scala:62)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
	at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
	at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
	at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
	at org.apache.spark.scheduler.ResultTask.runTask(ResultTask.scala:66)
	at org.apache.spark.scheduler.Task.run(Task.scala:88)
	at org.apache.spark.executor.Executor$TaskRunner.run(Executor.scala:214)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at java.lang.Thread.run(Thread.java:745)
Caused by: java.io.IOException: error=13, Permission denied
	at java.lang.UNIXProcess.forkAndExec(Native Method)
	at java.lang.UNIXProcess.<init>(UNIXProcess.java:248)
	at java.lang.ProcessImpl.start(ProcessImpl.java:134)
	at java.lang.ProcessBuilder.start(ProcessBuilder.java:1029)
	... 20 more

17/01/10 15:31:47 ERROR TaskSetManager: Task 0 in stage 1.0 failed 4 times; aborting job
17/01/10 15:31:47 INFO YarnScheduler: Removed TaskSet 1.0, whose tasks have all completed, from pool 
17/01/10 15:31:47 INFO YarnScheduler: Cancelling stage 1
17/01/10 15:31:47 INFO DAGScheduler: ResultStage 1 (dfToCols at NativeMethodAccessorImpl.java:-2) failed in 23.005 s
17/01/10 15:31:47 INFO DAGScheduler: Job 1 failed: dfToCols at NativeMethodAccessorImpl.java:-2, took 23.056582 s
17/01/10 15:31:47 ERROR RBackendHandler: dfToCols on org.apache.spark.sql.api.r.SQLUtils failed
17/01/10 15:31:47 INFO YarnHistoryService: About to POST entity application_1483512430911_0002 with 10 events to timeline service http://slavenode4:8188/ws/v1/timeline/
Error in invokeJava(isStatic = TRUE, className, methodName, ...) : 
  org.apache.spark.SparkException: Job aborted due to stage failure: Task 0 in stage 1.0 failed 4 times, most recent failure: Lost task 0.3 in stage 1.0 (TID 4, slavenode2): java.io.IOException: Cannot run program "Rscript": error=13, Permission denied
	at java.lang.ProcessBuilder.start(ProcessBuilder.java:1048)
	at org.apache.spark.api.r.RRDD$.createRProcess(RRDD.scala:407)
	at org.apache.spark.api.r.RRDD$.createRWorker(RRDD.scala:423)
	at org.apache.spark.api.r.BaseRRDD.compute(RRDD.scala:62)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
	at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
	at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
	at org.apache.spark.rdd.RDD
17/01/10 16:30:49 INFO ContextCleaner: Cleaned accumulator 1
17/01/10 16:30:49 INFO BlockManagerInfo: Removed broadcast_1_piece0 on slavenode3:45665 in memory (size: 3.6 KB, free: 530.0 MB)
17/01/10 16:30:49 INFO BlockManagerInfo: Removed broadcast_1_piece0 on slavenode2:42279 in memory (size: 3.6 KB, free: 530.0 MB)
17/01/10 16:30:49 INFO BlockManagerInfo: Removed broadcast_1_piece0 on 10.1.5.65:44133 in memory (size: 3.6 KB, free: 530.0 MB)
17/01/10 16:30:49 INFO ContextCleaner: Cleaned accumulator 2
17/01/10 16:30:49 INFO BlockManagerInfo: Removed broadcast_0_piece0 on 10.1.5.65:44133 in memory (size: 854.0 B, free: 530.0 MB)
17/01/10 16:30:49 INFO BlockManagerInfo: Removed broadcast_0_piece0 on slavenode2:42279 in memory (size: 854.0 B, free: 530.0 MB)
```

搜了一下没有理想答案，所以只能看源码了,下面是报错的代码位置：
```scala
private def createRProcess(port: Int, script: String): BufferedStreamThread = {
    // "spark.sparkr.r.command" is deprecated and replaced by "spark.r.command",
    // but kept here for backward compatibility.
    val sparkConf = SparkEnv.get.conf
    var rCommand = sparkConf.get("spark.sparkr.r.command", "Rscript")
    rCommand = sparkConf.get("spark.r.command", rCommand)

    val rOptions = "--vanilla"
    val rLibDir = RUtils.sparkRPackagePath(isDriver = false)
    val rExecScript = rLibDir(0) + "/SparkR/worker/" + script
    print(rCommand)
    print(rOptions)
    print(rExecScript)
    val pb = new ProcessBuilder(Arrays.asList(rCommand, rOptions, rExecScript))
    // Unset the R_TESTS environment variable for workers.
    // This is set by R CMD check as startup.Rs
    // (http://svn.r-project.org/R/trunk/src/library/tools/R/testing.R)
    // and confuses worker script which tries to load a non-existent file
    pb.environment().put("R_TESTS", "")
    pb.environment().put("SPARKR_RLIBDIR", rLibDir.mkString(","))
    pb.environment().put("SPARKR_WORKER_PORT", port.toString)
    pb.redirectErrorStream(true)  // redirect stderr into stdout
    val proc = pb.start()
    val errThread = startStdoutThread(proc)
    errThread
  }
```
上面的print语句是我加上去的。


忘了一个前置条件，使用sparkR的时候不指定master，也就是local模式的话是可以成功的。
```
>sc<-sparkR.init()
Launching java with spark-submit command spark-submit   sparkr-shell /tmp/RtmpRKsp2x/backend_port306c2e180534 
17/01/10 18:20:53 INFO SparkContext: Running Spark version 1.5.2
17/01/10 18:20:54 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
17/01/10 18:20:54 INFO SecurityManager: Changing view acls to: zhangjiayi
17/01/10 18:20:54 INFO SecurityManager: Changing modify acls to: zhangjiayi
17/01/10 18:20:54 INFO SecurityManager: SecurityManager: authentication disabled; ui acls disabled; users with view permissions: Set(zhangjiayi); users with modify permissions: Set(zhangjiayi)
17/01/10 18:20:59 INFO Slf4jLogger: Slf4jLogger started
17/01/10 18:20:59 INFO Remoting: Starting remoting
17/01/10 18:21:01 INFO Remoting: Remoting started; listening on addresses :[akka.tcp://sparkDriver@10.1.5.65:45017]
17/01/10 18:21:01 INFO Utils: Successfully started service 'sparkDriver' on port 45017.
17/01/10 18:21:01 INFO SparkEnv: Registering MapOutputTracker
17/01/10 18:21:02 INFO SparkEnv: Registering BlockManagerMaster
17/01/10 18:21:02 INFO DiskBlockManager: Created local directory at /tmp/blockmgr-a7f2bf0d-c363-4220-ade9-5210fe54f9f7
17/01/10 18:21:03 INFO MemoryStore: MemoryStore started with capacity 530.0 MB
17/01/10 18:21:03 INFO HttpFileServer: HTTP File server directory is /tmp/spark-d5f07559-6482-4523-aa40-8b99b7fbc84b/httpd-00d9c253-b8f8-41c2-ae1e-ad5bac726bf1
17/01/10 18:21:03 INFO HttpServer: Starting HTTP Server
17/01/10 18:21:04 INFO Server: jetty-8.y.z-SNAPSHOT
17/01/10 18:21:04 INFO AbstractConnector: Started SocketConnector@0.0.0.0:35349
17/01/10 18:21:04 INFO Utils: Successfully started service 'HTTP file server' on port 35349.
17/01/10 18:21:04 INFO SparkEnv: Registering OutputCommitCoordinator
17/01/10 18:21:05 INFO Server: jetty-8.y.z-SNAPSHOT
17/01/10 18:21:05 INFO AbstractConnector: Started SelectChannelConnector@0.0.0.0:4040
17/01/10 18:21:05 INFO Utils: Successfully started service 'SparkUI' on port 4040.
17/01/10 18:21:05 INFO SparkUI: Started SparkUI at http://10.1.5.65:4040
17/01/10 18:21:05 WARN MetricsSystem: Using default name DAGScheduler for source because spark.app.id is not set.
17/01/10 18:21:05 INFO Executor: Starting executor ID driver on host localhost
17/01/10 18:21:05 INFO Utils: Successfully started service 'org.apache.spark.network.netty.NettyBlockTransferService' on port 38751.
17/01/10 18:21:05 INFO NettyBlockTransferService: Server created on 38751
17/01/10 18:21:05 INFO BlockManagerMaster: Trying to register BlockManager
17/01/10 18:21:05 INFO BlockManagerMasterEndpoint: Registering block manager localhost:38751 with 530.0 MB RAM, BlockManagerId(driver, localhost, 38751)
17/01/10 18:21:05 INFO BlockManagerMaster: Registered BlockManager
> 
> sqlcontext<-sparkRSQL.init(sc)sc
Error: unexpected symbol in "sqlcontext<-sparkRSQL.init(sc)sc"
> sqlcontext<-sparkRSQL.init(sc)
> df<-createDataFrame(sqlContext = sqlcontext,data = iris)
17/01/10 18:23:34 INFO SparkContext: Starting job: collectPartitions at NativeMethodAccessorImpl.java:-2
17/01/10 18:23:34 INFO DAGScheduler: Got job 0 (collectPartitions at NativeMethodAccessorImpl.java:-2) with 1 output partitions
17/01/10 18:23:34 INFO DAGScheduler: Final stage: ResultStage 0(collectPartitions at NativeMethodAccessorImpl.java:-2)
17/01/10 18:23:34 INFO DAGScheduler: Parents of final stage: List()
17/01/10 18:23:34 INFO DAGScheduler: Missing parents: List()
17/01/10 18:23:34 INFO DAGScheduler: Submitting ResultStage 0 (ParallelCollectionRDD[0] at parallelize at RRDD.scala:454), which has no missing parents
17/01/10 18:23:35 INFO MemoryStore: ensureFreeSpace(1280) called with curMem=0, maxMem=555755765
17/01/10 18:23:35 INFO MemoryStore: Block broadcast_0 stored as values in memory (estimated size 1280.0 B, free 530.0 MB)
17/01/10 18:23:35 INFO MemoryStore: ensureFreeSpace(854) called with curMem=1280, maxMem=555755765
17/01/10 18:23:35 INFO MemoryStore: Block broadcast_0_piece0 stored as bytes in memory (estimated size 854.0 B, free 530.0 MB)
17/01/10 18:23:35 INFO BlockManagerInfo: Added broadcast_0_piece0 in memory on localhost:38751 (size: 854.0 B, free: 530.0 MB)
17/01/10 18:23:35 INFO SparkContext: Created broadcast 0 from broadcast at DAGScheduler.scala:861
17/01/10 18:23:35 INFO DAGScheduler: Submitting 1 missing tasks from ResultStage 0 (ParallelCollectionRDD[0] at parallelize at RRDD.scala:454)
17/01/10 18:23:35 INFO TaskSchedulerImpl: Adding task set 0.0 with 1 tasks
17/01/10 18:23:36 INFO TaskSetManager: Starting task 0.0 in stage 0.0 (TID 0, localhost, PROCESS_LOCAL, 16553 bytes)
17/01/10 18:23:36 INFO Executor: Running task 0.0 in stage 0.0 (TID 0)
17/01/10 18:23:36 INFO Executor: Finished task 0.0 in stage 0.0 (TID 0). 15464 bytes result sent to driver
17/01/10 18:23:36 INFO TaskSetManager: Finished task 0.0 in stage 0.0 (TID 0) in 249 ms on localhost (1/1)
17/01/10 18:23:36 INFO TaskSchedulerImpl: Removed TaskSet 0.0, whose tasks have all completed, from pool 
17/01/10 18:23:36 INFO DAGScheduler: ResultStage 0 (collectPartitions at NativeMethodAccessorImpl.java:-2) finished in 0.300 s
17/01/10 18:23:36 INFO DAGScheduler: Job 0 finished: collectPartitions at NativeMethodAccessorImpl.java:-2, took 1.618354 s
Warning messages:
1: In FUN(X[[i]], ...) :
  Use Sepal_Length instead of Sepal.Length  as column name
2: In FUN(X[[i]], ...) :
  Use Sepal_Width instead of Sepal.Width  as column name
3: In FUN(X[[i]], ...) :
  Use Petal_Length instead of Petal.Length  as column name
4: In FUN(X[[i]], ...) :
  Use Petal_Width instead of Petal.Width  as column name
> 
> head(df)
17/01/10 18:23:45 INFO SparkContext: Starting job: dfToCols at NativeMethodAccessorImpl.java:-2
17/01/10 18:23:45 INFO DAGScheduler: Got job 1 (dfToCols at NativeMethodAccessorImpl.java:-2) with 1 output partitions
17/01/10 18:23:45 INFO DAGScheduler: Final stage: ResultStage 1(dfToCols at NativeMethodAccessorImpl.java:-2)
17/01/10 18:23:45 INFO DAGScheduler: Parents of final stage: List()
17/01/10 18:23:45 INFO DAGScheduler: Missing parents: List()
17/01/10 18:23:45 INFO DAGScheduler: Submitting ResultStage 1 (MapPartitionsRDD[4] at dfToCols at NativeMethodAccessorImpl.java:-2), which has no missing parents
17/01/10 18:23:45 INFO MemoryStore: ensureFreeSpace(9176) called with curMem=2134, maxMem=555755765
17/01/10 18:23:45 INFO MemoryStore: Block broadcast_1 stored as values in memory (estimated size 9.0 KB, free 530.0 MB)
17/01/10 18:23:45 INFO MemoryStore: ensureFreeSpace(3690) called with curMem=11310, maxMem=555755765
17/01/10 18:23:45 INFO MemoryStore: Block broadcast_1_piece0 stored as bytes in memory (estimated size 3.6 KB, free 530.0 MB)
17/01/10 18:23:45 INFO BlockManagerInfo: Added broadcast_1_piece0 in memory on localhost:38751 (size: 3.6 KB, free: 530.0 MB)
17/01/10 18:23:45 INFO SparkContext: Created broadcast 1 from broadcast at DAGScheduler.scala:861
17/01/10 18:23:45 INFO DAGScheduler: Submitting 1 missing tasks from ResultStage 1 (MapPartitionsRDD[4] at dfToCols at NativeMethodAccessorImpl.java:-2)
17/01/10 18:23:45 INFO TaskSchedulerImpl: Adding task set 1.0 with 1 tasks
17/01/10 18:23:45 INFO TaskSetManager: Starting task 0.0 in stage 1.0 (TID 1, localhost, PROCESS_LOCAL, 16553 bytes)
17/01/10 18:23:45 INFO Executor: Running task 0.0 in stage 1.0 (TID 1)
17/01/10 18:23:51 INFO Executor: Finished task 0.0 in stage 1.0 (TID 1). 1794 bytes result sent to driver
17/01/10 18:23:51 INFO DAGScheduler: ResultStage 1 (dfToCols at NativeMethodAccessorImpl.java:-2) finished in 5.469 s
17/01/10 18:23:51 INFO DAGScheduler: Job 1 finished: dfToCols at NativeMethodAccessorImpl.java:-2, took 5.493984 s
17/01/10 18:23:51 INFO TaskSetManager: Finished task 0.0 in stage 1.0 (TID 1) in 5468 ms on localhost (1/1)
17/01/10 18:23:51 INFO TaskSchedulerImpl: Removed TaskSet 1.0, whose tasks have all completed, from pool 
  Sepal_Length Sepal_Width Petal_Length Petal_Width Species
1          5.1         3.5          1.4         0.2  setosa
2          4.9         3.0          1.4         0.2  setosa
3          4.7         3.2          1.3         0.2  setosa
4          4.6         3.1          1.5         0.2  setosa
5          5.0         3.6          1.4         0.2  setosa
6          5.4         3.9          1.7         0.4  setosa
```


单机 :
```
-----------will---------------------will mid----------zhangjiayi
-----------will-end----------rCommand is : RscriptrOptions is : --vanillarExecScript is : /usr/hdp/2.3.4.0-3485/spark/R/lib/SparkR/worker/daemon.R
```
用户是zhangjiayi ,
执行的系统xi

集群是打印在yarn的container的stdout里的：
```
-----------will---------------------will mid----------yarn
-----------will-end----------rCommand is : RscriptrOptions is : --vanillarExecScript is : /server/hadoop/yarn/local/usercache/zhangjiayi/appcache/application_1483512430911_0006/container_e18_1483512430911_0006_01_000002/sparkr/SparkR/worker/daemon.R-----------will---------------------will mid----------yarn
-----------will-end----------rCommand is : RscriptrOptions is : --vanillarExecScript is : /server/hadoop/yarn/local/usercache/zhangjiayi/appcache/application_1483512430911_0006/container_e18_1483512430911_0006_01_000002/sparkr/SparkR/worker/daemon.R
```

结果找不到：
```
[root@slavenode5 ~]# ll /server/hadoop/yarn/local/usercache/zhangjiayi/appcache/application_1483512430911_0006/container_e18_1483512430911_0006_01_000002/sparkr/SparkR/worker/daemon.R
ls: cannot access /server/hadoop/yarn/local/usercache/zhangjiayi/appcache/application_1483512430911_0006/container_e18_1483512430911_0006_01_000002/sparkr/SparkR/worker/daemon.R: No such file or directory
[root@slavenode5 ~]# ll /server/hadoop/yarn/local/usercache/zhangjiayi/appcache/application_1483512430911_0006/container_e18_1483512430911_0006_01_000002/sparkr/SparkR/worker/daemon.R
ls: cannot access /server/hadoop/yarn/local/usercache/zhangjiayi/appcache/application_1483512430911_0006/container_e18_1483512430911_0006_01_000002/sparkr/SparkR/worker/daemon.R: No such file or directory
[root@slavenode5 ~]# ll /server/hadoop/yarn/local/usercache/zhangjiayi/appcache/application_1483512430911_0006/container_e18_1483512430911_0006_01_000002/
total 24
-rw-r--r--. 1 yarn hadoop   88 Jan 11 12:25 container_tokens
-rwx------. 1 yarn hadoop  687 Jan 11 12:25 default_container_executor_session.sh
-rwx------. 1 yarn hadoop  741 Jan 11 12:25 default_container_executor.sh
-rwx------. 1 yarn hadoop 4042 Jan 11 12:25 launch_container.sh
lrwxrwxrwx. 1 yarn hadoop  122 Jan 11 12:25 __spark__.jar -> /server/hadoop/yarn/local/usercache/zhangjiayi/filecache/19/spark-assembly-1.5.2.2.3.4.0-3485-hadoop2.7.1.2.3.4.0-3485.jar
drwx--x---. 2 yarn hadoop 4096 Jan 11 12:25 tmp

```

此时控制台的报错只有：

```

17/01/11 12:25:42 INFO TaskSetManager: Starting task 0.2 in stage 1.0 (TID 3, slavenode5, PROCESS_LOCAL, 16553 bytes)
17/01/11 12:25:52 WARN TaskSetManager: Lost task 0.2 in stage 1.0 (TID 3, slavenode5): java.net.SocketTimeoutException: Accept timed out
	at java.net.PlainSocketImpl.socketAccept(Native Method)
	at java.net.AbstractPlainSocketImpl.accept(AbstractPlainSocketImpl.java:409)
	at java.net.ServerSocket.implAccept(ServerSocket.java:545)
	at java.net.ServerSocket.accept(ServerSocket.java:513)
	at org.apache.spark.api.r.RRDD$.createRWorker(RRDD.scala:446)
	at org.apache.spark.api.r.BaseRRDD.compute(RRDD.scala:62)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
	at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
	at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
	at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
	at org.apache.spark.scheduler.ResultTask.runTask(ResultTask.scala:66)
	at org.apache.spark.scheduler.Task.run(Task.scala:88)
	at org.apache.spark.executor.Executor$TaskRunner.run(Executor.scala:214)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at java.lang.Thread.run(Thread.java:745)
17/01/11 12:25:52 INFO TaskSetManager: Starting task 0.3 in stage 1.0 (TID 4, slavenode1, PROCESS_LOCAL, 16553 bytes)
17/01/11 12:26:02 WARN TaskSetManager: Lost task 0.3 in stage 1.0 (TID 4, slavenode1): java.net.SocketTimeoutException: Accept timed out
	at java.net.PlainSocketImpl.socketAccept(Native Method)
	at java.net.AbstractPlainSocketImpl.accept(AbstractPlainSocketImpl.java:409)
	at java.net.ServerSocket.implAccept(ServerSocket.java:545)
	at java.net.ServerSocket.accept(ServerSocket.java:513)
	at org.apache.spark.api.r.RRDD$.createRWorker(RRDD.scala:446)
	at org.apache.spark.api.r.BaseRRDD.compute(RRDD.scala:62)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
	at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
	at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
	at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
	at org.apache.spark.scheduler.ResultTask.runTask(ResultTask.scala:66)
	at org.apache.spark.scheduler.Task.run(Task.scala:88)
	at org.apache.spark.executor.Executor$TaskRunner.run(Executor.scala:214)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at java.lang.Thread.run(Thread.java:745)

17/01/11 12:26:02 ERROR TaskSetManager: Task 0 in stage 1.0 failed 4 times; aborting job
17/01/11 12:26:02 INFO YarnScheduler: Removed TaskSet 1.0, whose tasks have all completed, from pool 
17/01/11 12:26:02 INFO YarnScheduler: Cancelling stage 1
17/01/11 12:26:02 INFO DAGScheduler: ResultStage 1 (dfToCols at NativeMethodAccessorImpl.java:-2) failed in 42.286 s
17/01/11 12:26:02 INFO DAGScheduler: Job 1 failed: dfToCols at NativeMethodAccessorImpl.java:-2, took 42.314808 s
17/01/11 12:26:02 ERROR RBackendHandler: dfToCols on org.apache.spark.sql.api.r.SQLUtils failed
Error in invokeJava(isStatic = TRUE, className, methodName, ...) : 
  org.apache.spark.SparkException: Job aborted due to stage failure: Task 0 in stage 1.0 failed 4 times, most recent failure: Lost task 0.3 in stage 1.0 (TID 4, slavenode1): java.net.SocketTimeoutException: Accept timed out
	at java.net.PlainSocketImpl.socketAccept(Native Method)
	at java.net.AbstractPlainSocketImpl.accept(AbstractPlainSocketImpl.java:409)
	at java.net.ServerSocket.implAccept(ServerSocket.java:545)
	at java.net.ServerSocket.accept(ServerSocket.java:513)
	at org.apache.spark.api.r.RRDD$.createRWorker(RRDD.scala:446)
	at org.apache.spark.api.r.BaseRRDD.compute(RRDD.scala:62)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
	at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
	at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:300)
	at org.apache.spark.rdd.RDD.iterator(RDD.scala:264)
	at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
```

可以看到是createRWorker里而不是createRProcess报错了。。





---

```
Launching java with command  /server/java/jdk1.8.0_60/bin/java   -Xmx512m -cp '/usr/lib64/R/library/SparkR/sparkr-assembly-0.1.jar:' edu.berkeley.cs.amplab.sparkr.SparkRBackend /tmp/Rtmp34rELP/backend_port4e6c126d914 
createSparkContext on edu.berkeley.cs.amplab.sparkr.RRDD failed with java.lang.NullPointerException
java.lang.NullPointerException
	at edu.berkeley.cs.amplab.sparkr.SparkRBackendHandler.handleMethodCall(SparkRBackendHandler.scala:111)
	at edu.berkeley.cs.amplab.sparkr.SparkRBackendHandler.channelRead0(SparkRBackendHandler.scala:58)
	at edu.berkeley.cs.amplab.sparkr.SparkRBackendHandler.channelRead0(SparkRBackendHandler.scala:19)
	at io.netty.channel.SimpleChannelInboundHandler.channelRead(SimpleChannelInboundHandler.java:105)
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:308)
	at io.netty.channel.AbstractChannelHandlerContext.fireChannelRead(AbstractChannelHandlerContext.java:294)
	at io.netty.handler.codec.MessageToMessageDecoder.channelRead(MessageToMessageDecoder.java:103)
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:308)
	at io.netty.channel.AbstractChannelHandlerContext.fireChannelRead(AbstractChannelHandlerContext.java:294)
	at io.netty.handler.codec.ByteToMessageDecoder.channelRead(ByteToMessageDecoder.java:244)
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:308)
	at io.netty.channel.AbstractChannelHandlerContext.fireChannelRead(AbstractChannelHandlerContext.java:294)
	at io.netty.channel.DefaultChannelPipeline.fireChannelRead(DefaultChannelPipeline.java:846)
	at io.netty.channel.nio.AbstractNioByteChannel$NioByteUnsafe.read(AbstractNioByteChannel.java:131)
	at io.netty.channel.nio.NioEventLoop.processSelectedKey(NioEventLoop.java:511)
	at io.netty.channel.nio.NioEventLoop.processSelectedKeysOptimized(NioEventLoop.java:468)
	at io.netty.channel.nio.NioEventLoop.processSelectedKeys(NioEventLoop.java:382)
	at io.netty.channel.nio.NioEventLoop.run(NioEventLoop.java:354)
	at io.netty.util.concurrent.SingleThreadEventExecutor$2.run(SingleThreadEventExecutor.java:111)
	at io.netty.util.concurrent.DefaultThreadFactory$DefaultRunnableDecorator.run(DefaultThreadFactory.java:137)
	at java.lang.Thread.run(Thread.java:745)

```
