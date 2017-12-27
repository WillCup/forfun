
title: prometheus：理解告警的几个延迟点
date: 2017-09-22 14:51:15
tags: [youdaonote]
---

抓取、计算、与告警
---
prometheus会在指定配置周期内进行metric的抓取，这个周期参数是`scrape_interval`，默认是1分钟。可以全局定义一个，然后每个job再自定义覆盖。每次爬取到数据之后，prometheus会计算告警规则里的表达式，并根据结果修改告警状态。

一个告警会有以下状态：
 - inactive
 - pending. 已经检验失败，但是还没有到告警规则中for指定的持续时间。
 - firing. 已经检验失败，而且已经超过了告警规则中for指定的持续时间。

告警状态之间的变化只会发生在计算阶段，主要是取决于表达式和FOR语法定义内容

for是一个可选的定义。
- 没有定义for的告警会立即触发
- 定义了for的会先到pending，时间超过后再转化到firing。所以会有至少两个计算阶段。


告警周期
---
来个例子

告警规则：
```
ALERT NODE_LOAD_1M
  IF node_load1 > 20
  FOR 1m
```

相关配置
```
global:
  scrape_interval: 20s
  evaluation_interval: 1m
```

**问题来了**：发送这个告警到底会花费多少时间？

**答案**：介于1m和20s+1m+1m之间。

下图是事件发生的时序图：
![](https://pracucci.com/assets/2016-11-16-prometheus-alert-lifecycle-612f4a8f0171d3e56c2cc2ed4bcfb90232bfbdd2d1273d11d97f52a0e3cd121d.png)

alertmanager
---
alertManager是整个流程的最后一个环节，它会接收inactive和firing的信号。它可以将相似的告警进行分组，以一个通告发出去。这种小组发送的支持可以避免大量重复信息频繁发送的问题。但是小组发送的缺点也比较明显，就是可能会有更大的延迟。

下面是相关设置：
```
group_by: [ 'a-label', 'another-label' ]
group_wait: 30s
group_interval: 5m
```
在新的告警已经被fire后，它会等待group_wait时间到了才会发送。

但是如果第一批还没发送完的时候，第二批又来了咋整？第二批就不看group_wait了，而是看group_interval。

