title: "Flume 测试笔记（8）"
date: 2015-05-06 00:04:52
tags:
------

已经打通了经脉，还需要考虑负载均衡以及容灾问题。鉴于现在log的数据压力不是很大，而且投入的机器也不是很多，暂时只考虑容
灾，暂时不考虑负载均衡。

## 1.flume代码逻辑
flume里原生一个Failover Sink Processor，维护failedSinks和liveSinks两个东西来进行工作。首先会测试配置的所有sink，>工作时只取liveSink里的一个作为activeSink，进行实质性的工作。
    
FailoverSinkProcessor中的逻辑如下：
```
@Override
  public Status process() throws EventDeliveryException {
    // Retry any failed sinks that have gone through their "cooldown" period
    Long now = System.currentTimeMillis();
    while(!failedSinks.isEmpty() && failedSinks.peek().getRefresh() < now) {
      FailedSink cur = failedSinks.poll();
      Status s;
      try {
        s = cur.getSink().process();
        if (s  == Status.READY) {
          liveSinks.put(cur.getPriority(), cur.getSink());
          activeSink = liveSinks.get(liveSinks.lastKey());
          logger.debug("Sink {} was recovered from the fail list",
                  cur.getSink().getName());
        } else {
          // if it's a backoff it needn't be penalized.
          failedSinks.add(cur);
        }
        return s;
      } catch (Exception e) {
        cur.incFails();
        failedSinks.add(cur);
      }
    }

    Status ret = null;
    while(activeSink != null) {
      try {
        ret = activeSink.process();
        return ret;
      } catch (Exception e) {
        logger.warn("Sink {} failed and has been sent to failover list",
                activeSink.getName(), e);
        activeSink = moveActiveToDeadAndGetNext();
      }
    }

    throw new EventDeliveryException("All sinks failed to process, " +
        "nothing left to failover to");
  }
```
分析一下执行逻辑，在初始化的时候，failedSink是空的【可以参见configure方法】。在执行第二个while循环的时候，当有activeSink处理失败，就会把这个sink加到failedSinks里去，然后获取liveSinks里的下一个sink作为activeSink。但是这个process方法并没有返回执行的逻辑，那么就应该被重复调用，才可能执行到上面的第一个while循环。
SinkRunner中的逻辑：
```java
@Override
    public void run() {
      logger.debug("Polling sink runner starting");

      while (!shouldStop.get()) { //只要shouldStop为false便会一次次的执行
        try {
          if (policy.process().equals(Sink.Status.BACKOFF)) { //这里的policy就是SinkProcessor
            counterGroup.incrementAndGet("runner.backoffs");

            Thread.sleep(Math.min(
                counterGroup.incrementAndGet("runner.backoffs.consecutive")
                * backoffSleepIncrement, maxBackoffSleep));
          } else {
            counterGroup.set("runner.backoffs.consecutive", 0L);
          }
        } catch (InterruptedException e) {
          logger.debug("Interrupted while processing an event. Exiting.");
          counterGroup.incrementAndGet("runner.interruptions");
        } catch (Exception e) {
          logger.error("Unable to deliver event. Exception follows.", e);
          if (e instanceof EventDeliveryException) {
            counterGroup.incrementAndGet("runner.deliveryErrors");
          } else {
            counterGroup.incrementAndGet("runner.errors");
          }
          try {
            Thread.sleep(maxBackoffSleep);
          } catch (InterruptedException ex) {
            Thread.currentThread().interrupt();
          }
        }
      }
      logger.debug("Polling runner exiting. Metrics:{}", counterGroup);
    }
```
这样看来，其实flume的FailoverSinkProcessor也是会自动发现恢复了的Sink的。
## 2. flume官网文档
> Failover Sink Processor maintains **a prioritized list of sinks**, guaranteeing that so long as one is available events will be processed (delivered).The failover mechanism works by relegating failed sinks to a pool where they are assigned a cool down period, increasing with sequential failures before they are retried. Once a sink successfully sends an event, it is restored to the live pool.

>To configure, set a **sink groups** processor to failover and set priorities for all individual sinks. All specified priorities must be **unique**. Furthermore, upper limit to failover time can be set (in milliseconds) using maxpenalty property.

>Example for agent named a1:
a1.sinkgroups = g1
a1.sinkgroups.g1.sinks = k1 k2
a1.sinkgroups.g1.processor.type = failover
a1.sinkgroups.g1.processor.priority.k1 = 5
a1.sinkgroups.g1.processor.priority.k2 = 10
a1.sinkgroups.g1.processor.maxpenalty = 10000

我们看到，这里又涉及到了sink group这个东西，其实主要是为了做负载均衡和容灾时更加方便一些，也很好理解，就是把sink分个组而已，然后负载均衡和容灾可以指定一个组就可以了，也就针对这个组里的sink进行对应的策略实施。找了一个比较好的例子
```
# channels
agent.channels = mem_channel
agent.channels.mem_channel.type = memory

# sources
agent.sources = event_source
agent.sources.event_source.type = avro
agent.sources.event_source.bind = 127.0.0.1
agent.sources.event_source.port = 10000
agent.sources.event_source.channels = mem_channel

# sinks
agent.sinks = main_sink backup_sink

agent.sinks.main_sink.type = avro
agent.sinks.main_sink.hostname = 127.0.0.1
agent.sinks.main_sink.port = 10001
agent.sinks.main_sink.channel = mem_channel

agent.sinks.backup_sink.type = avro
agent.sinks.backup_sink.hostname = 127.0.0.1
agent.sinks.backup_sink.port = 10002
agent.sinks.backup_sink.channel = mem_channel

# sink groups    
agent.sinkgroups = failover_group
agent.sinkgroups.failover_group.sinks = main_sink backup_sink
agent.sinkgroups.failover_group.processor.type = failover
agent.sinkgroups.failover_group.processor.priority.main_sink = 10
agent.sinkgroups.failover_group.processor.priority.backup_sink = 5
```
