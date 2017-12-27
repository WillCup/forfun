
title: prometheus的alertmanager部分
date: 2017-08-23 11:07:29
tags: [youdaonote]
---

概念
---

#### 分组
把近似的告警放进单个通知中。

例如：一个服务的几百台实例分布在不同的机器中，如果出现局部的网络问题，那么可能会报出部分服务大约100个报警信息，我们是需要知道哪些实例有问题的，这样才能快速定位区域。但是没有必要发送几百条通知，只需要一条通知，包含所有的实例列表就可以了。

#### inhibit
inhibition是用来避免关联报警的。

例如：如果一个集群不可用的告警已经firing了。那么Altermanager可以让其他的与集群相关的告警不发出。这样能避免此种类型的告警爆炸问题。

#### silence

一般用来指定不告警的时段。通常基于matcher进行配置。

告警规则
---
```
ALERT <alert name>
  IF <expression>
  [ FOR <duration> ]    发生持续时长
  [ LABELS <label set> ]    为告警指定label，同样label的告警会被最新的告警覆盖
  [ ANNOTATIONS <label set> ]
```

例子
```
# Alert for any instance that is unreachable for >5 minutes.
ALERT InstanceDown
  IF up == 0
  FOR 5m
  LABELS { severity = "page" }
  ANNOTATIONS {
    summary = "Instance {{ $labels.instance }} down",
    description = "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes.",
  }

# Alert for any instance that have a median request latency >1s.
ALERT APIHighRequestLatency
  IF api_http_request_latencies_second{quantile="0.5"} > 1
  FOR 1m
  ANNOTATIONS {
    summary = "High request latency on {{ $labels.instance }}",
    description = "{{ $labels.instance }} has a median request latency above 1s (current value: {{ $value }}s)",
  }
```

label存的是实例的label信息，value是IF表达式的结果。
