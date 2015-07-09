title: "flume 源码阅读笔记（2）"
date: 2015-07-08 20:15:48
tags:
---
通过寻找LifecycleAware的子类就可以发现，所有flume的component都继承了这个接口。下面是几个主要的

***
# Channel 
连接source和sink，可以看做是MQ或者buffer，保证自身线程安全。
实在是懒得翻译，直接copy文档
A channel **connects** a Source to a Sink. The source acts as producer while the sink acts as a consumer of events. The channel itself is the **buffer** between the two.
A channel exposes a **Transaction** interface that can be used by its clients to ensure **atomic** put and take semantics. This is necessary to guarantee single hop reliability between agents. For instance, a source will successfully produce an event if and only if that event can be committed to the source's associated channel. Similarly, a sink will consume an event if and only if its respective endpoint can accept the event. The extent of transaction support varies for different channel implementations ranging from strong to best-effort semantics. 
Channels are associated with **unique names** that can be used for separating configuration and working namespaces. 
Channels must be **thread safe**, protecting any internal invariants as no guarantees are given as to when and by how many sources/sinks they may be simultaneously accessed by. 

    *  Transaction getTransaction()
    *  put(Event event)
    *  Event take() 

***
# Sink 
连接channel，然后消费channel里的内容，根据sink类型的不同，把这些message发往不同的目的地。
可以根据不同的行为使用**SinkGroup**和**SinkProcessor**来为Sink分组。**SinkRunner**通过processor定期轮询他们。

    *  Status process() 
        在某个transaction范围内消费channel里的message，如果transaction成功，就提交。若没成功，需要**回滚**这个transaction。
    要确保这个方法只有一个线程调用
    *   Channel getChannel()
    *  setChannel(Channel channel)

***
# Source 
A source generates {@plainlink Event **events**} and calls methods on the configured **ChannelProcessor** to persist those events into the configured channels. 
Sources are associated with **unique names** that can be used for separating configuration and working namespaces. 
不用搭理线程安全。

    * ChannelProcessor getChannelProcessor()
    * setChannelProcessor(ChannelProcessor channelProcessor)

***
# SinkProcessor
Interface for a device that allows abstraction of the behavior of multiple sinks, always assigned to a SinkRunner 。
A sink processors **SinkProcessor.process()** method will only be accessed by a **single** runner thread. However configuration methods such as Configurable.configure may be concurrently accessed.

    *  Status process() 
        Handle a request to **poll the owned sinks**.The processor is expected to call **Sink.process()** on whatever sink(s) appropriate, handling failures as appropriate and throwing EventDeliveryException when there is a failure to deliver any events according to the delivery policy defined by the sink processor implementation. See specific implementations of this interface for delivery behavior and policies.
    *  setSinks(List<Sink> sinks)
看起来这个processor有些像wrapper + manager的意思。

***
# SinkSelector
An interface that allows the **LoadBalancingSinkProcessor** to use a load-balancing strategy such as round-robin, random distribution etc. Implementations of this class can be plugged into the system via **processor configuration** and are used to select a sink on every invocation. 
An instance of the configured sink selector is create during the processor configuration, its setSinks(List) method is invoked following which it is configured via a subcontext. Once configured, **the lifecycle of this selector is tied to the lifecycle of the sink processor**. 
At runtime, the processor invokes the createSinkIterator() method for every process call to create  **an iteration order over the available sinks**. The processor then loops through this iteration order until one of the sinks succeeds in processing the event. If the iterator is exhausted and none of the sinks succeed, the processor will raise an EventDeliveryException. 

    *  setSinks(List<Sink> sinks)
    *  Iterator<Sink> createSinkIterator()
    *  void informSinkFailed(Sink failedSink)

*** 
# SourceRunner
A source runner controls **how a source is driven**. This is an abstract class used for instantiating derived classes.

    *  Source source
    *  SourceRunner forSource(Source source)  
        根据source类型实例化SourceRunner实现类的静态工厂方法

*** 
# SinkRunner    
A **driver** for sinks that polls them, attempting to process events if any are available in the Channel. 
Note that, unlike sources, all sinks are polled.

    *  Thread runnerThread
    *  LifecycleState lifecycleState
    *  **SinkProcessor** policy
    *  CounterGroup counterGroup
    *  PollingRunner runner
轮询调用sink的驱动器，如果channel里有任何可用的event，都会尝试执行。与source不同，所有的sink都会被涉及。Runner像是Processor的Wrapper。

```java
@Override
  public void start() {
    SinkProcessor policy = getPolicy();
    policy.start();
    runner = new PollingRunner(); //封装shouldStop决定是否运行
    runner.policy = policy;
    runner.counterGroup = counterGroup;
    runner.shouldStop = new AtomicBoolean();
    runnerThread = new Thread(runner);
    runnerThread.setName("SinkRunner-PollingRunner-" +
        policy.getClass().getSimpleName());
    runnerThread.start();
    lifecycleState = LifecycleState.START;
  }
```


