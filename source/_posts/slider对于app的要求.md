
title: slider对于app的要求
date: 2017-12-05 17:49:35
tags: [youdaonote]
---

slider把app安装并运行在yarn上，但是app并不一定要是针对yarn定制的。

app应该是被slider发布，也就是yarn负责安装，slider负责配置，最后yarn负责执行。yarn会在销毁container的时候kill掉执行的进程，所以被部署的app应该知道这个，并能够在没有人工参与的前提下自动启动一个新的container运行自己的component。

app的component还应该能够自己动态发现其他的component，在启动和执行的时候都要能发现，因为后面server或者进程失败的话，也会导致component位置变化。


#### must
- 使用tar进行安装运行，而且运行用户不能是root
- 自包含或者所有的依赖都已经预装了
- 支持节点的动态发现，比如通过ZK
- 节点能够动态rebind自己，就是说如果即便节点被删除，app仍能继续运行
- 把kill作为通用机制处理
- 支持多个实例运行在同个集群中，不同app实例可以共享同一个server
- 当多个role实例被分配到同一个物理节点的时候，要能够正确操作。
- 安全可靠。yarn不会再sandbox中运行代码的
- 如果与HDFS交互，注意兼容性
- 持久化数据到HDFS，配置好文件位置。slider中每个app实例都需要有一个目录。
- 配置目录要可配置，最好不要是绝对路径
- log输出不要是一个固定的位置
- 不明确让它停止，就一直跑下去。slider吧app的终止作为失败处理，会自动重启container。

#### must not

- 人工的启动和关闭


#### should

- 发布一个RPC或者HTTP端口，可以通过zk或者admin API。slider会提供一个发布机制
- 可通过标准的hadoop机制进行配置：text、xml文件。不然，就需要自定义的解析器
- 支持明确的参数来指定配置目录
- 允许通过-D类似方式指定运行参数
- 不要运行复杂脚本
- 提供给slider获取集群节点和状态的方式，这样slider才能探测失败的worker节点并处理
- 启动的时候能够感知位置。
- 支持简单的探活，可以是http get
- 退出的时候返回一个可读的内容，方便针对性处理
- 支持集群大小调整
- 支持ambari类似的平台管理


#### may

- 动态分配RPC或者web端口
- 包含一个单独的进程，运行在指定位置，这个进程一挂，就引起app终止。这样的进程也能运行在slider am的container中。如果一个活着的cluster不能处理这个进程的restart或者migration，这个slider app就不能处理slider am的restart
- 汇报load/cost

#### may not

- yarn可写
- 纯java
- 支持动态配置

