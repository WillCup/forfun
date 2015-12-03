title: springboot坑记1
date: 2015-12-09 10:18:08
tags: [spring, spring boot]
---

## 背景
业务上需要做一些定时任务的调度，最好还能对这些调度的状态进行记录与监控，可以通过一个友好的界面看到调度的执行情况是最好的。
考虑的范围：spring + quartz、dropwizard、spring boot，因为当前公司的项目是使用spring mvc写的，也就是说公司的人力资源方面偏向于spring序列。
放弃quartz的原因：
    - 只是负责调度，没有看到有对调度进度或者状态有监控的地方
    - 对于调度堆叠的处理不是很清楚
最终定位到spring boot：
    - 计划是使用spring的web来进行调度监控方面的工作
    - ApplicationRunner来启动自己写的调度器。

