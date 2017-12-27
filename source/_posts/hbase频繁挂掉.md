
title: hbase频繁挂掉
date: 2017-06-25 09:57:58
tags: [youdaonote]
---

```
2017-06-25 09:42:34,262 ERROR [RS_OPEN_REGION-datanode20:16020-0] coprocessor.CoprocessorHost: The coprocessor org.apache.kylin.stora
ge.hbase.cube.v2.coprocessor.endpoint.CubeVisitService threw org.apache.hadoop.ipc.RemoteException(org.apache.hadoop.ipc.StandbyExcep
tion): Operation category READ is not supported in state standby
```



```
2017-06-25 09:42:34,336 FATAL [RS_OPEN_REGION-datanode20:16020-2] regionserver.HRegionServer: ABORTING region server datanode20.yinke
r.com,16020,1498354685968: The coprocessor org.apache.kylin.storage.hbase.cube.v1.coprocessor.observer.AggregateRegionObserver threw 
org.apache.hadoop.ipc.RemoteException(org.apache.hadoop.ipc.StandbyException): Operation category READ is not supported in state stan
dby


2017-06-25 09:42:34,336 FATAL [RS_OPEN_REGION-datanode20:16020-2] regionserver.HRegionServer: RegionServer abort: loaded coprocessors are: [org.apache.hadoop.hbase.security.access.SecureBulkLoadEndpoint]


2017-06-25 09:42:34,338 FATAL [RS_OPEN_REGION-datanode20:16020-0] regionserver.HRegionServer: ABORTING region server datanode20.will.com,16020,1498354685968: The coprocessor org.apache.kylin.storage.hbase.cube.v1.coprocessor.observer.AggregateRegionObserver threw org.apache.hadoop.ipc.RemoteException(org.apache.hadoop.ipc.StandbyException): Operation category READ is not supported in state standby


2017-06-25 09:42:34,370 INFO  [RS_OPEN_REGION-datanode20:16020-1] regionserver.RegionCoprocessorHost: Loaded coprocessor org.apache.k
ylin.storage.hbase.cube.v2.coprocessor.endpoint.CubeVisitService from HTD of KYLIN_4L40B34P11 successfully.
2017-06-25 09:42:34,379 ERROR [RS_OPEN_REGION-datanode20:16020-2] handler.OpenRegionHandler: Failed open of region=KYLIN_HBWCQ5ZQR1,\
x001,1494915979852.71f4a908f1e460ac95dfadcf5ab8ecfd., starting to roll back the global memstore size.
2017-06-25 09:42:34,381 INFO  [RS_OPEN_REGION-datanode20:16020-2] coordination.ZkOpenRegionCoordination: Opening of region {ENCODED =
> 71f4a908f1e460ac95dfadcf5ab8ecfd, NAME => 'KYLIN_HBWCQ5ZQR1,\x001,1494915979852.71f4a908f1e460ac95dfadcf5ab8ecfd.', STARTKEY => '\x
001', ENDKEY => '\x002'} failed, transitioning from OPENING to FAILED_OPEN in ZK, expecting version 3
2017-06-25 09:42:34,382 ERROR [RS_OPEN_REGION-datanode20:16020-0] handler.OpenRegionHandler: Failed open of region=KYLIN_HBWCQ5ZQR1,\
x00\x15,1494915979852.c4940efcc8efa3f3a9b7c2a546a1d1f5., starting to roll back the global memstore size.
org.apache.hadoop.ipc.RemoteException(org.apache.hadoop.ipc.StandbyException): Operation category READ is not supported in state stan
dby




java.io.IOException: Cannot append; log is closed
        at org.apache.hadoop.hbase.regionserver.wal.FSHLog.append(FSHLog.java:1223)
        at org.apache.hadoop.hbase.regionserver.wal.WALUtil.writeRegionEventMarker(WALUtil.java:95)
        at org.apache.hadoop.hbase.regionserver.HRegion.writeRegionOpenMarker(HRegion.java:978)
        at org.apache.hadoop.hbase.regionserver.HRegion.openHRegion(HRegion.java:6335)
        at org.apache.hadoop.hbase.regionserver.HRegion.openHRegion(HRegion.java:6289)
        at org.apache.hadoop.hbase.regionserver.HRegion.openHRegion(HRegion.java:6260)
        at org.apache.hadoop.hbase.regionserver.HRegion.openHRegion(HRegion.java:6216)
        at org.apache.hadoop.hbase.regionserver.HRegion.openHRegion(HRegion.java:6167)
        at org.apache.hadoop.hbase.regionserver.handler.OpenRegionHandler.openRegion(OpenRegionHandler.java:362)
        ...............
2017-06-25 09:42:44,168 INFO  [regionserver/datanode20.will.com/10.2.19.109:16020.logRoller] regionserver.LogRoller: LogRoller exit
ing.
2017-06-25 09:42:44,168 INFO  [regionserver/datanode20.will.com/10.2.19.109:16020] regionserver.CompactSplitThread: Waiting for Spl
it Thread to finish...
2017-06-25 09:42:44,168 INFO  [regionserver/datanode20.will.com/10.2.19.109:16020] regionserver.CompactSplitThread: Waiting for Mer
ge Thread to finish...
2017-06-25 09:42:44,168 INFO  [regionserver/datanode20.will.com/10.2.19.109:16020] regionserver.CompactSplitThread: Waiting for Lar
ge Compaction Thread to finish...
2017-06-25 09:42:44,168 INFO  [regionserver/datanode20.will.com/10.2.19.109:16020] regionserver.CompactSplitThread: Waiting for Sma
ll Compaction Thread to finish...
2017-06-25 09:42:44,170 INFO  [regionserver/datanode20.will.com/10.2.19.109:16020.leaseChecker] regionserver.Leases: regionserver/d
atanode20.will.com/10.2.19.109:16020.leaseChecker closing leases
2017-06-25 09:42:44,171 INFO  [regionserver/datanode20.will.com/10.2.19.109:16020.leaseChecker] regionserver.Leases: regionserver/d
atanode20.will.com/10.2.19.109:16020.leaseChecker closed leases
2017-06-25 09:42:44,183 INFO  [regionserver/datanode20.will.com/10.2.19.109:16020] ipc.RpcServer: Stopping server on 16020

```
