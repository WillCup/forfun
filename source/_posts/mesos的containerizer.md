
title: mesos的containerizer
date: 2017-07-24 18:17:14
tags: [youdaonote]
---

Docker Containerizer
---
Docker containerizer使用dockers engine管理container。

#### container启动过程
- Docker containerizer 在ContainerInfo::type为DOCKER的时候尝试在docker里启动task
- Docker containerizer先pull image
- 调用pre-launch的hook
- executor以下面两种方式启动起来
    - mesos agent运行在一个docker container中
        - 这种方式需要提供--docker_mesos_image的参数。使用指定的docker image启动mesos agent
        - 如果task包含自定义的executor，那么这个executor就在docker container中被启动
        - 如果task并不包含executor。就是说它定义了一个command，默认的mesos-docker-executor就在docker container中通过docker Cli执行这个command。
    - mesos agent不运行在docker container中
        - 如果task中包含自定义的executor，那么在docker container中执行
        - 如果task不博阿涵自定义的executor，也就是定义了一个command，就fork一个子进程来执行默认的mesos-docker-executor. 然后mesos-docker-executor产生一个shell通过docker cli执行这个command。

Mesos Containerizer
---
Mesos Containerizer是原生的。Mesos Containerizer会处理所有没有制定ContainerInfo::DockerInfo的executor/task。

#### container启动过程
- 在每个isolator上调用prepare
- 使用Launcher fork出executor。在isolated之前，这个被fork出来的executor是不能执行的
- Isolate这个executor。通过pid调用每个isolator的isolate方法
- 抽取executor
- 执行executor。就是上面的executor会获得执行的信号。先执行一些准备的命令，然后再执行executor的内容。

#### Launcher
Launcher负责产生和销毁container。
- 在containerizerd context中fork新的进程。这个子进程会执行给定路径的可执行文件，接受参数、flag和环境变量等。
- 子进程的IO会根据指定的方式进行重定向

###### linux launcher

- 为container创建一个freezer 的cgroup
- 创建一个posix的pipe为host和container进程提供通信通道
- 使用系统的clone方法生成子进程(也就是container进程)
- 把新生成的container放进freezer的对应层级中
- 通过上面的pip给子进程一个signal，让它继续执行


###### Posix launcher (TBD)

#### Isolators
Isolator负责创建container中的运行环境【CPU, NETWORK, 存储，内存等】

Containerizer states
---

Docker
- FETCHING
- PULLING
- RUNNING
- DESTROYING

Mesos
- PREPARING
- ISOLATING
- FETCHING
- RUNNING
- DESTROYING
