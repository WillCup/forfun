title: azkaban源码追踪（二）-单次调度flow
date: 2016-06-15 11:09:03
tags: [azkaban,源码]
---
对应的操作是进入到某个project中，然后单次执行下面的某个flow。会触发web-server这边与exec-server的远程调用去执行flow。

#### 相关类
- Web-server
    - ExecutorServlet.doGet
    - ExecutorManager
- exec-server
    - ExecutorServlet.doGet
    - FlowRunnerManager
    - FlowRunner
    - ExecutableFlowBase
    - ExecutableNode
    - JobRunner
    - JobTypeManager
    - Job
    - AzkabanProcessBuilder

#### 
