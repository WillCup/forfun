
title: flink的容错机制
date: 2017-11-21 16:44:02
tags: [youdaonote]
---

简介
---
Flink容错机制保证了，即便出现失败，程序的state也最终reflect每条记录exactly once，也可以降级到at least once。

这个容错机制持续的描绘出分布式数据流的snapshot。对于少有state的流式应用来说，这些snapshot是非常轻量的，可以在不影响到性能的前提下快速完成。流式应用的state会被存储在配置好的地方(比如master节点、hdfs等)。

如果程序失败了(机器原因、网络原因、软件问题等)，Flink会踢掉分布式数据流。然后系统会重启所有的operator，然后把他们重置到最近的成功的checkpoint。输入流也被重置到这个state snapshot的点。作为重新启动的并行dataflow的一部分，已经处理过的所有的记录都保证不属于前一个checkpoint state里的一部分。

注意：默认checkpoint是未启用的。


注意：要完全保证这个机制，需要数据流的source(一般是消息队列或者broker)能够支持重置位移。例如kafka就可以重置位移。

注意：以你为Flink的checkpoints是通过分布式snapshot实现的，我们可以交换使用snapshot和checkpoint这两个词语。


checkpoint
---
Flinke容错机制的核心就是为分布式的数据流和操作状态进行snapshot。这些snapshot作为一致性的checkpoint供错误恢复使用。Flink执行snapshot的机制在[ “Lightweight Asynchronous Snapshots for Distributed Dataflows”.](http://arxiv.org/abs/1506.08603)有介绍。是由[Chandy-Lamport algorithm](http://research.microsoft.com/en-us/um/people/lamport/pubs/chandy.pdf)实现的，这是一种分布式snapshot方法，而且兼容Flink的执行模型。

#### Barrier栅栏
Flink分布式snapshot的核心元素是stream barrier。这些栅栏被注入到data stream和flow当中作为data stream的一部分存在。barrier永远不会超过记录，flow是严格线性的。barrier负责把data stream中的数据进行切分，切分为两部分，一部分到当前的snapshot中，然后另外一部分是下一个snapshot里的。Each barrier carries the ID of the snapshot whose records it pushed in front of it. barrier并不会中断数据流处理，十分轻量。不同snampshot的多个barrier可能同时存在于stream中，这就是说可能会多个snapshot并发执行。

![](https://ci.apache.org/projects/flink/flink-docs-release-1.3/fig/stream_barriers.svg)

Stream barrier在stream source被注入到并发的data flow中。snapshot n的barrier的点Sn被注入的位置是source stream知道覆盖到数据的snapshot【The point where the barriers for snapshot n are injected (let’s call it Sn) is the position in the source stream up to which the snapshot covers the data】。例如在kafka中，这个位置就是当前分区上一次访问的记录的offset。这个位置Sn就被汇报给checkpoint coordinator(Flink里就是JobManager)。


然后barrier继续向下执行。当一个中间oerator抽取到snapshot n的barrier时，它发射一个snapshot n的barrier给它所有的outgoing stream。一旦一个sink operator(DAG的最后)收到barrier n的时候，它就向checkpoint coordinator确认ack snapshot n。所有的sink都ack了这个snapshot的时候，它就被认为已经成功了。

snapshot n完成之后，job再也不会请求Sn之前的数据记录了，因为这些数据记录已经通过了整个数据流处理拓扑。

![](https://ci.apache.org/projects/flink/flink-docs-release-1.3/fig/stream_aligning.svg)

接收多个input stream的operator必须排列多个input stream的snapshot barriers。上图内容
- operator接到单个input stream的snapshot barrier n的时候，不能立即处理这个stream的数据，需要等着其他input stream的barrier n也到达的时候才行。否则，它会把snapshot n的数据记录和snapshot n+1的数据记录弄混了。
- 已经汇报barrier n的input stream暂时被搁置。收到的数据记录暂时不处理，而是放进一个input buffer
- 最后一个stream也收到barrier n的时候，这个operator就发射所有pending的outing 记录，然后自己也发射snapshot n的barrier。
- 最后，它恢复处理所有input stream的数据记录，处理在此之前的input buffer里的数据记录，然后处理stream的记录。

#### state
如果operator包含任意形式的state，这个state就必须成为snapshot的一部分。operator statue可能是下面的几种：
- user-defined state。通过转化方法(例如map、filter)直接创建与修改的state。
- system state。例如operator计算的一部分，数据缓存等。典型的例子是window buffer，在window buffer中通常收集聚合数据记录，直到window的计算或者被清空。


operator在收到了它所有input stream的barrier n，并且没有发送barrier n给它的所有output stream之前，snapshot它的state。在这个时间点，barrier之前的所有的数据记录导致的state更新已经完成，并且没有根据已经执行的的barrier之后的数据记录进行任何更新。因为snapshot 的state可能会很大，它可以被存储在一个可配置的[后端](https://ci.apache.org/projects/flink/flink-docs-release-1.3/ops/state_backends.html)。默认情况下，是放在JobManager的内存里，对于生产环境，最好放在可靠的存储介质上，例如HDFS。在state被存储之后，operator ack这个checkpoint，发送这个snapshot barrier到output stream中。



现在snapshot包含了：
- 对于每个并行的stream data source，snapshot开始的时候stream中的位置信息或者offset
- 对于每个operator，一个指向snapshot state仓库的pointer


![checkpoint流程图](https://ci.apache.org/projects/flink/flink-docs-release-1.3/fig/checkpointing.svg)

Exactly Once还是At least once
----
对于streaming program来说alignment操作可能会造成一些延迟。通差，这部分延迟是早毫秒级的，但是我们见过一些非常突出的延迟。对于那些坚持要求低延迟的应用，Flink有一个开关可以关闭checkpoint过程中的alignment操作。checkpoint snapshot仍然很快。

当alignment被跳过后，即便一些checkpoint barrier n已经到了，operator也要继续处理所有的输入。这样的话，在snapshot n完结之前，operator也会处理属于 snapshot n+1的数据。再回复的时候，这些记录就会被重复处理，因为他们都被记录在了snapshot n的state snapshot里，所以也会作为checkpoint n之后的数据被replay。

注意：alignment只会发生在具有多个前辈(join)，或者多个sender(在一个stream被repartition/shuffle)的operator。因此，对于只包含并行处理操作(map, flatmap, filter等)的dataflow来说，at least once模式下其实也是exactly once的。


异步state snapshot
---

注意上面描述的机制意味着operator在奥村他们的snapshot state到state backend的时候，需要停止处理输入的记录。这个同步的state snapshot操作会造成一个小的时间延迟。


如果让这个存储操作异步执行的话，就能让operator继续执行下面的步骤了。要这样的话，operator必循能够生成一个state对象，这个state 对象能够被存储，而且后面operator state的更新不会影响到它。例如，copy-on-write数据结构，rocksdb里有这种操作。

接收到所有input stream的checkpoint barrier后，operator开始异步snapshot，copy他的state。它会马上发送barrier到它的output，然后继续下面的数据流处理。一旦后台的copy进程完成后，它就会向checkpoint coordinator(JobManager) ack这个checkpoint。 这个checkpoint现在只在所有的sink都收到barrier后，所有的statefule operator都ack他们完成了backup后(这个可能比barrier到达sink还要慢)才算完成。


recovery
---
在这个机制下的recovery很直接：一旦发生失败，Flink选择最新的完成的checkpoint k。然后这个system重新部署整个的分布式数据流，然后给每个operator这个snapshot k对应的state，因为这也是checkpoint k的一部分。而且需要从数据源的Sk的位置开始读取数据流。如果是kafka的话，就是从offset为Sk的地方开始抽取数据。

如果state是增量snapshot的，那么operator要使用最新full snapshot的state，然后把后续的更新操作应用到这个state上。

[更多](https://ci.apache.org/projects/flink/flink-docs-release-1.3/dev/restart_strategies.html)


operator snapshot的实现
---
当operator执行snapshot的时候，有两个部分：同步的和异步的。

operator和state backend是以java FutureTask的形式提供他们的snapshot的。这个task包含了同步部分已完成的state，但是异步部分可能还在pending状态。异步部分后面会被这个checkpoint的一个后台线程执行。

使用纯同步方式的operator checkpint会返回一个已经完成了的FutrueTask。如果异步操作需要被执行，就执行FutrueTask的run方法。

这些task是可以被cancel的，这样stream和其他的资源消费句柄就可以被release了。
