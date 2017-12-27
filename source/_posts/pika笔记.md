
title: pika笔记
date: 2017-12-22 16:17:13
tags: [youdaonote]
---

redis的角色从缓存，现在慢慢转向内存数据库了。


但是因为自身的内存限制上有限制，所以诞生了pika。借助了redis的很多机制和rocksdb的存储，基于rocksdb的接口实现pika的功能。

#### pika 1.0

ttl功能通过在插入数据时，修改rocksdb时带上ttl时间戳字段。修改rocksdb的get接口，进行时间戳判断。

使用version实现了大量数据的的秒删，先把这个hash当前version递加，其实原理跟hbase是一样的原理，后面再在compact的过程中进行真正物理的删除。

问题：
- 同步的数据缓冲区容易写满，扫瞄binlog的进程比较快，传输比较慢
- 存活检测与数据同步相互影响，存活检测迟迟得不到响应
- 全同步bgsave，主库dump，然后发给从库。很慢，而且数据量有很大

#### pika 2.0

###### 解决1.0的问题

- 线程合并与拆分，进行重构优化
- 存活检测与数据同步隔离
- 快照式备份，百G秒级相应。使用rocksdb的checkpoint执行，阻塞一下，确保数据一致，创建文件硬链

###### 解耦rocksdb

rocksdb更新速度非常快，使用新版特性的时候，要修改很多代码。

在rocksdb之上实现一个adaptor，也就是nemo-rocksdb。

回收metakey，metakey其实是hash表的唯一key。主要是比如hash1被删除了，但是还没有compact的时候，然后又马上重建了hash1，那么这个metakey的version就又成了0了。期望version是绝对的增长，后来把新建version改成当前时间戳了，删的时候就直接删，再新建的时候再弄这个当前时间戳。


#### pika 2.3.0

- pika双主，高可用。但是还需要切流量。当原主恢复后，可以先全同步数据，再增量同步，因为可能挂掉的时间比较长，binlog会很多。 还有：不支持双主写同一个key的value，可能会出现问题。推荐写只走一个实例，另一个作为standby。
- 跨机房同步。pika_hub以slave的身份加入多个机房集群，相互同步。注意pika_hub也要多实例高可用，借用外部的高可用一致性raft组件存储元数据、竞锁选主。如果有多个对于同一个key的操作，就以时间戳的形式兼容。pika_hub之间不同步log，只发送raft记录断点，主节点定时记录checkpoint到raft集群。
- 支持订阅。




