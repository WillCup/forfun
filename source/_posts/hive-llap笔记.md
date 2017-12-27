
title: hive-llap笔记
date: 2017-11-27 17:22:27
tags: [youdaonote]
---

#### 调度
- 每个coordinator一个query
- 分布式
- 所有资源共享
- 基于case亲mixing的fragment执行
- 容错性


- 基于优先级的智能调度，低优先级的fragment可以被取代(比如input数据还没有就绪就提前开始的)。LLAP 工作队列会检查dag参数来

参考：https://www.youtube.com/watch?v=n8jHOVkvNoc&t=1332s
