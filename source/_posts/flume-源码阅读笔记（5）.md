title: flume 源码阅读笔记（5）
date: 2015-07-12 19:52:37
tags: [flume, 源码]
---
前面分析了source的运行机制与过程，其实sink和source是很像的。

## 1. 入口
```java
for (Entry<String, SinkRunner> entry : materializedConfiguration.getSinkRunners()
    .entrySet()) {
  try{
    supervisor.supervise(entry.getValue(),
      new SupervisorPolicy.AlwaysRestartPolicy(), LifecycleState.START);
  } catch (Exception e) {
    logger.error("Error while starting {}", entry.getValue(), e);
  }
}
```
哈哈，经过上一节的流程，你是不是跟我一样信心满满啊~~ 嘻嘻—— SinkRunner

## 2. SinkRunner
同SourceRunner的身份地位是一样滴，可以理解为驱动器加管理者的角色，当channel里有event的时候就尝试消费之。不同于source的是，sink都是类似于PollableSource的这种，也就是需要自己对channel里的event进行抽取消费，因为channel里的东西不会自己sink，也不知道要sink到何处去。

- runner : **PollingRunner**	//我还需要说什么吗
- runnerThread : Thread
- lifecycleState : LifecycleState
- policy : SinkProcessor

入口方法start()

```java
public void start() {
	SinkProcessor policy = getPolicy();

	policy.start();

	runner = new PollingRunner();

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

好熟悉，完全就是PollableSourceRunner的同样思路。虽然猜到了结果，但还是要求证一下。上这个内部类PollingRunner：
```java
public void run() {
	....
  while (!shouldStop.get()) {
    try {
      if (policy.process().equals(Sink.Status.BACKOFF)) {	//
        ....
      } else {
        counterGroup.set("runner.backoffs.consecutive", 0L);
      }
    } catch (InterruptedException e) {
      .....
    }
  }
}
```
好吧，还是稍微有一点点区别，我列举一下这两个PollingRunner的成员。
PollableSourceRunner：

- source : **PollableSource**	//内置getProcessor()接口
- shouldStop : AtomicBoolean
- counterGroup : CounterGroup

SinkRunner:

- policy : **SinkProcessor**
- shouldStop : AtomicBoolean
- counterGroup : CounterGroup

差别就在policy和source上了，而这两者又有同样的接口API process方法来对channel里的event进行各自的处理。前面我们也分析过了source是通过channelProcessor进行实际的针对channel的操作的。


---
这一节营养实在太少。顺便说一声CounterGroup吧，内置一个HashMap<String, AtomicLong>，做各种计数与指标统计的工作，主要当然是针对event和各个组件。
