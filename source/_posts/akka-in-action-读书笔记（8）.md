title: akka in action 读书笔记（8）
date: 2016-08-19 17:50:33
tags: [akka]
---

系统结构。
- pipe和filter模型
- scatter-gather模型
- 路由
- Recipient list
- Aggregator
- Become/Unbecome

如果有很多工作单元并行运行的话，那么怎样协调它们的工作？有一些可以并行执行，有的任务A需要在前置任务B完成之后才能执行。

通过实现几个经典的Enterprise Integration Pattern，Akka使我们能够充分利用好它的并发功能。

先了解以下最简单的pipe和filter模型，它是基于消息传递的系统的默认模型，经典模型是顺序执行。然后介绍 Scatter-Gather模型，这种模型提供了各种使任务并行的方式。


8.1 pipe 和 filter
---

pipe的定义就是，一个进程或者线程把它的结果交给下一个进程或线程作为输入。许多人都是从它的发源地unix了解到pipe模型的。这些pipe组件的组合，被称为pipeline。


#### 8.1.1 Enterprise integration pattern Pipes and Filters


