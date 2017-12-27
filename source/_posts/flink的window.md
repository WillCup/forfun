
title: flink的window
date: 2017-11-21 15:28:54
tags: [youdaonote]
---

滑动窗口,sliding windows，比如我们每30s统计一下上一分钟的和。
![](https://flink.apache.org/img/blog/window-intro/window-sliding-window.png)


当只有一个窗口处理器的时候，我们就只能串行处理一个数据流。每个流里的书都要去向某个指定的window。在Flink中对于Windows on a full stream are【一个完整的流的窗口】 称为 AllWindows。对很多app来说，数据流都要分发进入多个逻辑流，然后会有window operator处理每个逻辑流。假设我们从多个交通传感器获取交通工具的流量，每个交通传感器都监控不同地址的流量。我们可以把这些信息流通过交通传感器的id进行分组，然后分别并发的计算每个交通传感器所在位置的流量信息。在Flink中，我们把这种partitioned window叫做simple window，因为这是对分布式数据流很常见的处理方式。下图展示了一个通过(sernsorID, count)的流进行数据收集的滚动窗口
![](https://flink.apache.org/img/blog/window-intro/windows-keyed.png)

通常，一个window在无穷的数据流中定义了一组有穷数据。这组数据可以是基于时间、计数、时间与技术结合、自定义的一些逻辑去分window。Flink的DataStream API提供了简洁的操作供常用的窗口操作，也留了接口让用户提供自定义的分窗口的逻辑。下面我们详细看一下基于时间与计数的窗口机制。

基于时间的窗口
---
下面是一个每分钟滚动一次的窗口，对所有的数据执行某个函数操作。
```
// (sensorId, carCnt)形式的数据流
val vehicleCnts: DataStream[(Int, Int)] = ...

// 0，1是在数据中的位置
val tumblingCnts: DataStream[(Int, Int)] = vehicleCnts
  // 通过sensorId分区
  .keyBy(0) 
  // 1分钟的窗口时间
  .timeWindow(Time.minutes(1))
  // 计算carCnt的和
  .sum(1) 

val slidingCnts: DataStream[(Int, Int)] = vehicleCnts
  .keyBy(0) 
  // 每30秒触发一次1分钟的数据窗口的执行
  .timeWindow(Time.minutes(1), Time.seconds(30))
  .sum(1)
```

还有一个时间的概念我们要说明
- processing time。窗口是根据当前主机的1分钟去构建与计算窗口内数据的。
- event time。event产生时的时间。相比processing time更好一些
- Ingestion time。是processing  time和event time的杂交品种。一旦数据到达，就把当前机器的时间戳赋给这个数据记录，然后基于这个时间戳，使用event time去持续处理。



基于计数的窗口
---
一个100个event作为窗口的程序。
```
// (sensorId, carCnt)形式的数据流
val vehicleCnts: DataStream[(Int, Int)] = ...

val tumblingCnts: DataStream[(Int, Int)] = vehicleCnts
  // 用sensorId分区
  .keyBy(0)
  // 100个数据为一个窗口
  .countWindow(100)
  .sum(1)

val slidingCnts: DataStream[(Int, Int)] = vehicleCnts
  .keyBy(0)
  // 100个数据作为一个窗口，每10个触发一次窗口处理
  .countWindow(100, 10)
  .sum(1)

```

深入了解
---

Flink内置的基于时间、计数的窗口已经覆盖了大多数的应用，但是对于一些需要自定义窗口划分逻辑的，需要使用DataStream API暴露的接口，这些接口给了窗口构建与计算的很有条理的控制方式。

下图是Flink窗口机制的详细图
![](https://flink.apache.org/img/blog/window-intro/window-mechanics.png)


到达window operator的数据会先发给`WindowAssigner`. `WindowAssigner`把数据分配给一个或者多个window，也有可能要创建新的window。一个`Window`是一个一组数据、元数据(对于`TimeWindow`来说是起止时间)的唯一标识符。注意数据是可以被加入到多个window的，也就是说可能会同时存在多个window。

每个window拥有一个`Trigger`，它来决定当前window什么时候计算或者清空。这个trigger在每条数据到来的时候都会检验，如果前面注册的timer超时就会触发操作：执行计算、清空窗口、先计算再清空。**如果只是触发计算，那么所有的数据就还留在window里**，下次触发的时候还可以计算。**一个window在被清空之前一直都可以被计算，这也代表着内存占用**。

计算函数接收window的所有数据，输出一个或多个结果。DataStream API接受不同类型的计算函数，包括一些预定义的sum、min、max，ReduceFunction、FlodFunciton、WindowFunction等。最常见的是`WindowFunction`，它接收window对象的元数据，一组window对象，window key(如果是分区的window)作为参数。

这些组件构成了Flink的窗口机制。我们现在一步步看一下怎样实现自定义窗口逻辑。我们以DataStream[IN]类型的stream开始，用一个key选择器函数来抽取key，获得一个Keydtream[IN, KEY].
```
val input: DataStream[IN] = ...

val keyed: KeyedStream[IN, KEY] = input
  .keyBy(myKeySel: (IN) => KEY)


// 通过WindowAssigner创建一个分窗口的stream。WindowAssigner 有默认的Trigger实现
var windowed: WindowedStream[IN, KEY, WINDOW] = keyed
  .window(myAssigner: WindowAssigner[IN, WINDOW])
  
// 不适用WindowAssigner默认的trigger
windowed = windowed
  .trigger(myTrigger: Trigger[IN, WINDOW])

// 指定可选的evictor
windowed = windowed
  .evictor(myEvictor: Evictor[IN, WINDOW])

// 最后，把window function传递给windowed stream
val output: DataStream[OUT] = windowed
  .apply(myWinFunc: WindowFunction[IN, OUT, KEY, WINDOW])
```



