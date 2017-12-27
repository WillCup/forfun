
title: slider配置相关
date: 2017-12-22 13:54:42
tags: [youdaonote]
---

#### 配置

###### component的失败处理

文档中并没有明确说这回事儿，需要自己去理解，理解了好半天。就是需要下面两个参数配置起来才行的。先配置一个window，指定一个时段长度，然后在单个时段长度内达到阈值上限才会认为app失败，否则就会一直重启。

参数 | 作用 | 默认
--- | --- | ---
yarn.container.failure.threshold | 窗口内失败次数的阈值 | 默认貌似是5次
yarn.container.failure.window.day、yarn.container.failure.window.hours、yarn.container.failure.window.minutes | 窗口时间段 | 默认是`yarn.container.failure.window.hours=6`


###### 放置策略

其实就是把app分配到哪些node上的策略。有三个值
- 0. 默认的：就是重启的时候满集群找合适的节点，除了不可用的node都有可能被选中，如果没有满足的，就触发escalation
- 1. strict： 每个node都要有一个component，不管有没有失败历史。这就不会发生escalation了。
- 2. anywhere： 不顾历史，到处请求
- 4. anti affinity required(无穷远)：目前不支持。

例子
```
"HBASE_REST": {
  "yarn.role.priority": "3",
  "yarn.component.instances": "1",
  "yarn.component.placement.policy": "1",
  "yarn.memory": "556"
},
```

这里指定了为1，那么每次启动的时候都会找到上次运行的同一个节点。

yarn.node.failure.threshold | 在某个node上执行component失败的阈值，到达这个阈值后，就不会接受component了。



一个hbase的实例配置：
```json
{
  "schema": "http://example.org/specification/v2.0.0",
  "metadata": {
  },
  "global": {
    "yarn.log.include.patterns": "hbase.*",
    "yarn.log.exclude.patterns": "hbase.*out",
    // 30分钟一个window
    "yarn.container.failure.window.hours": "0",
    "yarn.container.failure.window.minutes": "30",
    // 选择带有development label的yarnnode分配component
    "yarn.label.expression":"development"
  },
  "components": {
    "slider-appmaster": {
      "yarn.memory": "1024",
      "yarn.vcores": "1"
      // 可以随意分配到default label的node
      "yarn.label.expression":""
    },
    "HBASE_MASTER": {
      // 数字越低，优先级越高，通常用来控制启动次序
      "yarn.role.priority": "1",
      "yarn.component.instances": "1",
      "yarn.placement.escalate.seconds": "10",
      "yarn.vcores": "1",
      "yarn.memory": "1500"
    },
    "HBASE_REGIONSERVER": {
      "yarn.role.priority": "2",
      "yarn.component.instances": "1",
      "yarn.vcores": "1",
      "yarn.memory": "1500",
      // 30分钟内失败15次的话，就标记这个component部署失败
      "yarn.container.failure.threshold": "15",
      "yarn.placement.escalate.seconds": "60"
    },
    "HBASE_REST": {
      "yarn.role.priority": "3",
      "yarn.component.instances": "1",
      "yarn.component.placement.policy": "1",
      // 30分钟内失败3次的话，就标记这个component部署失败
      "yarn.container.failure.threshold": "3",
      "yarn.vcores": "1",
      "yarn.memory": "556"
    },
    "HBASE_THRIFT": {
      "yarn.role.priority": "4",
      "yarn.component.instances": "0",
      "yarn.component.placement.policy": "1",
      "yarn.vcores": "1",
      "yarn.memory": "556",
      //  选择带有stable label的yarnnode分配component
      "yarn.label.expression":"stable"
    },
    "HBASE_THRIFT2": {
      "yarn.role.priority": "5",
      "yarn.component.instances": "1",
      "yarn.component.placement.policy": "1",
      "yarn.vcores": "1",
      "yarn.memory": "556"，
      //  选择带有stable label的yarnnode分配component
      "yarn.label.expression":"stable"
    }
  }
}
```
