
title: NN-HA后引发的问题
date: 2017-06-22 11:19:07
tags: [youdaonote]
---

###### 最表面问题:
几乎所有用户执行权限都出现问题。
###### 问题原因：
 - 所有HDFS之上的应用都是使用linux的posix文件权限控制
 - 当Active NN是DN17的时候，DN17本机并没有这些用户的相关权限信息。
###### 解决方法：
 - 尝试copyNN01的用户以及组信息到DN17
```

/etc/passwd
/etc/shadow


/etc/group
/etc/gshadow



[root@namenode01 hdfs]# cat /etc/group | grep xiaov
xiaov:x:1015:zhoujian,wangchenqi,zhangyuntao,wangxu,wanghao_fengkong,linzhixin,yangqiankun,tianye,zhaoyanchao,tinyv,wangyin,lihaoyue,zhuxiuwei,yangluan,qizhenpeng,zhanglining,xiaovtest,yanghuanyuzi,zhanghaigang,shichengjie,luanzhiwei,wangyasen,kangyimin,luiyahui,liuyahui,liulinyuan,caoyunqing,tianyang,chengyao,gezhao,hehaojun,hujunjie,liuwei,xiongpeng
xiaovtest:x:1040:
metastoremanager:x:1041:xiaovtest,wangchenqi


```

将上面所有的用户保存到另一台的xiaov文件中，然后执行下面的逻辑：
```
for name in `cat xiaov | awk '{split($0, atr,",");   
 
 for (i=0;i<length(atr);i++) {
 print(atr[i])
 }
 }'`;
do
useradd $name -G xiaov
done

```

 - 使用fabric维护用户权限相关信息，每次都同时操作两台NN。
 
```
-- fabric.py

nns=('namenode01.will.com', 'datanode17.will.com')

env.roledefs = {
    'services': services,
    'datanodes': datanodes,
    'tmp': tmpnodes,
    'nns': nns
}

------------------

[root@datanode21 ~]# fab cmd:roles=nns,cmd="echo hello from `hostname`"
This text is green!
This sentence is red, except for these words, which are green.
[namenode01.will.com] Executing task 'cmd'
[namenode01.will.com] run: echo hello from datanode21.will.com
[namenode01.will.com] out: hello from datanode21.will.com
[namenode01.will.com] out: 

[datanode17.will.com] Executing task 'cmd'
[datanode17.will.com] run: echo hello from datanode21.will.com
[datanode17.will.com] out: hello from datanode21.will.com
[datanode17.will.com] out: 


Done.
Disconnecting from datanode17.will.com... done.
Disconnecting from namenode01.will.com... done
```


#### NN切换的相关日志
NN01的log。

出现这个错误后，NN自己直接shutdown
```
2017-06-15 22:35:35,329 WARN  client.QuorumJournalManager (QuorumCall.java:waitFor(134)) - Waited 18014 ms (timeout=20000 ms) for a r
esponse for sendEdits. Succeeded so far: [10.2.19.121:8485]
2017-06-15 22:35:36,331 WARN  client.QuorumJournalManager (QuorumCall.java:waitFor(134)) - Waited 19015 ms (timeout=20000 ms) for a r
esponse for sendEdits. Succeeded so far: [10.2.19.121:8485]
2017-06-15 22:35:37,317 FATAL namenode.FSEditLog (JournalSet.java:mapJournalsAndReportErrors(398)) - Error: flush failed for required
 journal (JournalAndStream(mgr=QJM to [10.2.19.123:8485, 10.2.19.127:8485, 10.2.19.121:8485], stream=QuorumOutputStream starting at t
xid 118442295))
java.io.IOException: Timed out waiting 20000ms for a quorum of nodes to respond.
        at org.apache.hadoop.hdfs.qjournal.client.AsyncLoggerSet.waitForWriteQuorum(AsyncLoggerSet.java:137)
        at org.apache.hadoop.hdfs.qjournal.client.QuorumOutputStream.flushAndSync(QuorumOutputStream.java:107)
        at org.apache.hadoop.hdfs.server.namenode.EditLogOutputStream.flush(EditLogOutputStream.java:113)
        at org.apache.hadoop.hdfs.server.namenode.EditLogOutputStream.flush(EditLogOutputStream.java:107)
        at org.apache.hadoop.hdfs.server.namenode.JournalSet$JournalSetOutputStream$8.apply(JournalSet.java:533)
        at org.apache.hadoop.hdfs.server.namenode.JournalSet.mapJournalsAndReportErrors(JournalSet.java:393)
        at org.apache.hadoop.hdfs.server.namenode.JournalSet.access$100(JournalSet.java:57)
        at org.apache.hadoop.hdfs.server.namenode.JournalSet$JournalSetOutputStream.flush(JournalSet.java:529)
        at org.apache.hadoop.hdfs.server.namenode.FSEditLog.logSync(FSEditLog.java:647)
        at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.startFileInt(FSNamesystem.java:2512)
        at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.startFile(FSNamesystem.java:2377)
        at org.apache.hadoop.hdfs.server.namenode.NameNodeRpcServer.create(NameNodeRpcServer.java:708)
        at org.apache.hadoop.hdfs.protocolPB.ClientNamenodeProtocolServerSideTranslatorPB.create(ClientNamenodeProtocolServerSideTran
slatorPB.java:405)
        at org.apache.hadoop.hdfs.protocol.proto.ClientNamenodeProtocolProtos$ClientNamenodeProtocol$2.callBlockingMethod(ClientNamen
odeProtocolProtos.java)
        at org.apache.hadoop.ipc.ProtobufRpcEngine$Server$ProtoBufRpcInvoker.call(ProtobufRpcEngine.java:616)
        at org.apache.hadoop.ipc.RPC$Server.call(RPC.java:969)
        at org.apache.hadoop.ipc.Server$Handler$1.run(Server.java:2206)
        at org.apache.hadoop.ipc.Server$Handler$1.run(Server.java:2202)
        at java.security.AccessController.doPrivileged(Native Method)
        at javax.security.auth.Subject.doAs(Subject.java:422)
        at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1709)
        at org.apache.hadoop.ipc.Server$Handler.run(Server.java:2200)
2017-06-15 22:35:37,318 WARN  client.QuorumJournalManager (QuorumOutputStream.java:abort(72)) - Aborting QuorumOutputStream starting 
at txid 118442295
2017-06-15 22:35:37,337 INFO  BlockStateChange (BlockManager.java:computeReplicationWorkForBlocks(1531)) - BLOCK* neededReplications 
= 0, pendingReplications = 0.
2017-06-15 22:35:37,342 INFO  util.ExitUtil (ExitUtil.java:terminate(124)) - Exiting with status 1
2017-06-15 22:35:37,396 INFO  namenode.NameNode (LogAdapter.java:info(47)) - SHUTDOWN_MSG: 
/************************************************************
SHUTDOWN_MSG: Shutting down NameNode at namenode01.will.com/10.2.19.72

```



DN17的log
```
2017-06-19 16:34:39,682 WARN  client.QuorumJournalManager (QuorumCall.java:waitFor(134)) - Waited 27324 ms (timeout=20000 ms) for a r
esponse for selectInputStreams. No responses yet.

.........
2017-06-19 16:34:46,859 INFO  util.JvmPauseMonitor (JvmPauseMonitor.java:run(196)) - Detected pause in JVM or host machine (eg GC): p
ause of approximately 6029ms
No GCs detected
2017-06-19 16:34:49,739 INFO  util.JvmPauseMonitor (JvmPauseMonitor.java:run(196)) - Detected pause in JVM or host machine (eg GC): p
ause of approximately 2380ms
No GCs detected

```


```

2017-06-26 13:39:04,029 INFO  ipc.Server (Server.java:doRead(891)) - Socket Reader #1 for port 8020: readAndProcess from client 10.2.
19.83 threw exception [java.io.IOException: Connection reset by peer]
java.io.IOException: Connection reset by peer
        at sun.nio.ch.FileDispatcherImpl.read0(Native Method)
        at sun.nio.ch.SocketDispatcher.read(SocketDispatcher.java:39)
        at sun.nio.ch.IOUtil.readIntoNativeBuffer(IOUtil.java:223)
        at sun.nio.ch.IOUtil.read(IOUtil.java:197)
        at sun.nio.ch.SocketChannelImpl.read(SocketChannelImpl.java:380)
        at org.apache.hadoop.ipc.Server.channelRead(Server.java:2773)
        at org.apache.hadoop.ipc.Server.access$2800(Server.java:136)
        at org.apache.hadoop.ipc.Server$Connection.readAndProcess(Server.java:1606)
        at org.apache.hadoop.ipc.Server$Listener.doRead(Server.java:880)
        at org.apache.hadoop.ipc.Server$Listener$Reader.doRunLoop(Server.java:746)
        at org.apache.hadoop.ipc.Server$Listener$Reader.run(Server.java:717)
2017-06-26 13:39:04,066 INFO  ipc.Server (Server.java:doRead(891)) - Socket Reader #1 for port 8020: readAndProcess from client 10.2.
19.110 threw exception [java.io.IOException: Connection reset by peer]
java.io.IOException: Connection reset by peer
        at sun.nio.ch.FileDispatcherImpl.read0(Native Method)
        at sun.nio.ch.SocketDispatcher.read(SocketDispatcher.java:39)
        at sun.nio.ch.IOUtil.readIntoNativeBuffer(IOUtil.java:223)
        at sun.nio.ch.IOUtil.read(IOUtil.java:197)

.......


2017-06-26 13:39:08,310 WARN  client.QuorumJournalManager (IPCLoggerChannel.java:call(406)) - Took 4136ms to send a batch of 3 edits (262 bytes) to remote journal 10.2.19.121:8485
2017-06-26 13:39:08,310 WARN  client.QuorumJournalManager (IPCLoggerChannel.java:call(406)) - Took 4130ms to send a batch of 3 edits (262 bytes) to remote journal 10.2.19.123:8485
2017-06-26 13:39:08,310 WARN  client.QuorumJournalManager (IPCLoggerChannel.java:call(388)) - Remote journal 10.2.19.127:8485 failed to write txns 134233020-134233022. Will try to write to this JN again after the next log roll.
org.apache.hadoop.ipc.RemoteException(java.io.IOException): IPC's epoch 15 is less than the last promised epoch 16
        at org.apache.hadoop.hdfs.qjournal.server.Journal.checkRequest(Journal.java:418)
        at org.apache.hadoop.hdfs.qjournal.server.Journal.checkWriteRequest(Journal.java:446)
        at org.apache.hadoop.hdfs.qjournal.server.Journal.journal(Journal.java:341)
        at org.apache.hadoop.hdfs.qjournal.server.JournalNodeRpcServer.journal(JournalNodeRpcServer.java:148)
        at org.apache.hadoop.hdfs.qjournal.protocolPB.QJournalProtocolServerSideTranslatorPB.journal(QJournalProtocolServerSideTranslatorPB.java:158)
        at org.apache.hadoop.hdfs.qjournal.protocol.QJournalProtocolProtos$QJournalProtocolService$2.callBlockingMethod(QJournalProto
colProtos.java:25421)
        at org.apache.hadoop.ipc.ProtobufRpcEngine$Server$ProtoBufRpcInvoker.call(ProtobufRpcEngine.java:616)
        at org.apache.hadoop.ipc.RPC$Server.call(RPC.java:969)
        at org.apache.hadoop.ipc.Server$Handler$1.run(Server.java:2206)
        at org.apache.hadoop.ipc.Server$Handler$1.run(Server.java:2202)
        at java.security.AccessController.doPrivileged(Native Method)
        at javax.security.auth.Subject.doAs(Subject.java:422)
        at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1709)
        at org.apache.hadoop.ipc.Server$Handler.run(Server.java:2200)



..........

org.apache.hadoop.hdfs.qjournal.client.QuorumException: Got too many exceptions to achieve quorum size 2/3. 3 exceptions thrown:
10.2.19.127:8485: IPC's epoch 15 is less than the last promised epoch 16

```


又一次
```
2017-06-28 10:05:21,397 INFO  namenode.FSNamesystem (FSNamesystem.java:stopStandbyServices(1297)) - Stopping services started for standby state
2017-06-28 10:05:21,398 WARN  ha.EditLogTailer (EditLogTailer.java:doWork(349)) - Edit log tailer interrupted
java.lang.InterruptedException: sleep interrupted
        at java.lang.Thread.sleep(Native Method)
        at org.apache.hadoop.hdfs.server.namenode.ha.EditLogTailer$EditLogTailerThread.doWork(EditLogTailer.java:347)
        at org.apache.hadoop.hdfs.server.namenode.ha.EditLogTailer$EditLogTailerThread.access$200(EditLogTailer.java:284)
        at org.apache.hadoop.hdfs.server.namenode.ha.EditLogTailer$EditLogTailerThread$1.run(EditLogTailer.java:301)
        at org.apache.hadoop.security.SecurityUtil.doAsLoginUserOrFatal(SecurityUtil.java:449)
        at org.apache.hadoop.hdfs.server.namenode.ha.EditLogTailer$EditLogTailerThread.run(EditLogTailer.java:297)
2017-06-28 10:05:21,403 INFO  namenode.FSNamesystem (FSNamesystem.java:startActiveServices(1098)) - Starting services required for active state

```
