title: flume-源码阅读笔记（4）
date: 2015-07-10 03:53:24
tags: [flume, 源码]
---
这一章详细分析一下source的运行方式

## 1. 入口
前面我们提到了source的启动入口为：
```java
for (Entry<String, SourceRunner> entry : materializedConfiguration
        .getSourceRunners().entrySet()) {
      try{
        logger.info("Starting Source " + entry.getKey());
        // 下面这行就会以每3秒一次的频率调用SourceRunner
        supervisor.supervise(entry.getValue(),
          new SupervisorPolicy.AlwaysRestartPolicy(), LifecycleState.START);
      } catch (Exception e) {
        logger.error("Error while starting {}", entry.getValue(), e);
      }
    }
```
注意，上面的value不是Source，而是SourceRunner，那么我们的中心就围绕它展开了。

## 2. SourceRunner

简而言之就是source的驱动器和控制器，对source进行了一层包裹。根据source的不同分为PollableSourceRunner和EventDrivenSourceRunner。

先看一下SourceRunner的元素：

 * source : Source
 * forSource(Source)
    静态工厂方法，根据传入的source的类型实例化SourceRunner的实例，也就是上面提到的两种

PollableSourceRunner管理的是**PollableSource**，这种source是需要外界来抽取event的，例如KafkaSource，需要去消费。

EventDrivenSourceRunner管理的是**EventDrivenSource**，这种source自己就满足消息机制，可以向channel产生event，不需要外界来抽取。 很容易看出来，这种是稍微省心点儿的~~~

SourceRunner里的元素比较少，还是分析它的两个子类吧。为方便对比，先看稍微简单一些的

###　EventDrivenSourceRunner
元素只有一个lifecycleState : LifecycleState
启动代码：
```java
public void start() {
    Source source = getSource();
    ChannelProcessor cp = source.getChannelProcessor();
    cp.initialize();	// 初始化interceptorChain
    source.start();		// 启动source
    lifecycleState = LifecycleState.START;
  }
```
上面可见，source是在这里启动的。我们来参照一个具体实现ExecSource：
```java
 public void start() {
    executor = Executors.newSingleThreadExecutor();

    // 好，我们发现了一个Runnable
    runner = new ExecRunnable(shell, command, getChannelProcessor(), sourceCounter,
        restart, restartThrottle, logStderr, bufferCount, batchTimeout, charset);

    // FIXME: Use a callback-like executor / future to signal us upon failure.
    runnerFuture = executor.submit(runner);

    /*
     * NB: This comes at the end rather than the beginning of the method because
     * it sets our state to running. We want to make sure the executor is alive
     * and well first.
     */
    sourceCounter.start();
    super.start();

    logger.debug("Exec source started");
  }
```

看一下上面代码里的runnable：

```java
public void run() {
      do {
        .......
        try {
          // fork进程，执行shell
          if(shell != null) {
            String[] commandArgs = formulateShellCommand(shell, command);
            process = Runtime.getRuntime().exec(commandArgs);
          }  else {
            String[] commandArgs = command.split("\\s+");
            process = new ProcessBuilder(commandArgs).start();
          }
          ....

          // 首先如果当前eventList不为空，先处理
          future = timedFlushService.scheduleWithFixedDelay(new Runnable() {
              @Override
              public void run() {
                try {
                  synchronized (eventList) {
                    if(!eventList.isEmpty() && timeout()) {
                      flushEventBatch(eventList); // 批处理咯
                    }
                  }
                } catch (Exception e) {
                  ...
                }
              }
          },
          batchTimeout, batchTimeout, TimeUnit.MILLISECONDS);

          // 读取shell进程输出	
          while ((line = reader.readLine()) != null) {
            synchronized (eventList) {
              sourceCounter.incrementEventReceivedCount();
              eventList.add(EventBuilder.withBody(line.getBytes(charset)));
              if(eventList.size() >= bufferCount || timeout()) {
                flushEventBatch(eventList); // 批处理咯
              }
            }
          }

         .....
        } catch (Exception e) {
          ....
        } finally {
         ....
        }
        ......
      } while(restart);
    }
```

前面总是看到ChannelProcessor，还被迷惑地以为这个东西是给channel用的，后来自己才想明白，**channel只是个容器**。主动方应该是Source和Sink才对。。。果然，ChannelProcessor是给Source用的：

```java
private void flushEventBatch(List<Event> eventList){
      channelProcessor.processEventBatch(eventList);
      sourceCounter.addToEventAcceptedCount(eventList.size());
      eventList.clear();
      lastPushToChannel = systemClock.currentTimeMillis();
    }
```
ChannelProcessor提供的元素如下：

 - interceptorChain : InterceptorChain
 - selector : ChannelSelector
 - processEventBatch(List<Event>)
 - processEvent(Event)

PS: 猛然发现，前面其实我们已经通过channelProcessor的initialize方法初始化了**interceptorChain**，没错，interceptor就是交给它管理的。因为ChannelProcessor是处理event的实际作用点，所以对于event的过滤与处理也应该是在这里。

**event处理流程**：

* 通过interceptor
* 分类到各个channel -> 通过channelSelector找到对应关系
* 遍历channel，调用对应的transaction及其方法，进行put操作，关闭事务

展示一下事务这部分，以便联想起前面总结的事务部分内容，也可以串起来：
```java
public void processEvent(Event event) {
	...
	// Process required channels
    List<Channel> requiredChannels = selector.getRequiredChannels(event);
    for (Channel reqChannel : requiredChannels) {
      Transaction tx = reqChannel.getTransaction();
      Preconditions.checkNotNull(tx, "Transaction object must not be null");
      try {
        tx.begin(); // 开始

        // 这里就是调用前面提到BasicTransactionSemantics的API了
        reqChannel.put(event); 

        tx.commit();	//提交
      } catch (Throwable t) {
        tx.rollback();	// 回滚
        ....
      } finally {
        if (tx != null) {
          tx.close();	// 关闭
        }
      }
    }
    ...
}
```

---
### PollableSourceRunner
元素：

* runner : **PollingRunner**   // 关键就在这上面了
* runnerThread : Thread
* lifecycleState : LifecycleState
* shouldStop : AtomicBoolean

看一下它实现的start方法
```java
	PollableSource source = (PollableSource) getSource();
    ChannelProcessor cp = source.getChannelProcessor();
    cp.initialize();
    source.start();

    // 至此前面与EventDrivenSourceRunner的处理是一模一样的

    runner = new PollingRunner();

    runner.source = source;
    runner.counterGroup = counterGroup;
    runner.shouldStop = shouldStop;

    runnerThread = new Thread(runner);
    runnerThread.setName(getClass().getSimpleName() + "-" + 
        source.getClass().getSimpleName() + "-" + source.getName());
    runnerThread.start();

    lifecycleState = LifecycleState.START;
```
可以看到，后面起了一个线程，runnable就是PollingRunner。

```java
public void run() {
		....
      while (!shouldStop.get()) {
        ......
        try {
          if (source.process().equals(PollableSource.Status.BACKOFF)) { //只有一行关键
            ...
          } else {
            counterGroup.set("runner.backoffs.consecutive", 0L);
          }
        } catch (InterruptedException e) {
          ......
        }
      }
    }

```
明白了哈？就是执行了一下PollableSource里的process方法。前面提到了，这个接口的子类并非自身就是消息驱动的那种，**需要别人来抽取event**。所以前面的东西只是做准备，而PollingRunner来负责event的抽取以及将event放入channel的工作。

再次找个实现类，KafkaSource
```java
public Status process() throws EventDeliveryException {
	....
    try {
      while (eventList.size() < batchUpperLimit &&
              System.currentTimeMillis() < batchEndTime) {
        iterStatus = hasNext();
        if (iterStatus) {
          // get next message
          MessageAndMetadata<byte[], byte[]> messageAndMetadata = it.next();
          kafkaMessage = messageAndMetadata.message();
          kafkaKey = messageAndMetadata.key();

          // Add headers to event (topic, timestamp, and key)
          headers = new HashMap<String, String>();
          headers.put(KafkaSourceConstants.TIMESTAMP,
                  String.valueOf(System.currentTimeMillis()));
          headers.put(KafkaSourceConstants.TOPIC, topic);
          if (kafkaKey != null) {
            headers.put(KafkaSourceConstants.KEY, new String(kafkaKey));
          }
          if (log.isDebugEnabled()) {
            log.debug("Message: {}", new String(kafkaMessage));
          }
          event = EventBuilder.withBody(kafkaMessage, headers);
          eventList.add(event);
        }
      }
      .....
      if (eventList.size() > 0) {
        getChannelProcessor().processEventBatch(eventList);
        counter.addToEventAcceptedCount(eventList.size());
        eventList.clear();
        if (log.isDebugEnabled()) {
          log.debug("Wrote {} events to channel", eventList.size());
        }
        if (!kafkaAutoCommitEnabled) {
          // commit the read transactions to Kafka to avoid duplicates
          long commitStartTime = System.nanoTime();
          consumer.commitOffsets();
          long commitEndTime = System.nanoTime();
          counter.addToKafkaCommitTimer((commitEndTime-commitStartTime)/(1000*1000));
        }
      }
      ......
      return Status.READY;
    } catch (Exception e) {
      ......
    }
  }

```

看了这段代码后，可确定kafka的event就是由PollingRunner搞的。另外发现一个事儿，就是**ChannelProcessor专门负责处理event后，放入channel**。
