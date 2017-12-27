
title: docker内OS的相关事宜
date: 2017-07-20 14:59:20
tags: [youdaonote]
---

问题
---

假设宿主机是centos7，然后docker镜像使用ubuntu16.04。

那么dockers容器在启动的时候是怎样运行ubuntu的，这种情境咋总是感觉跟启动虚拟机一样呢？

各种cgroup什么的还好理解，但是这些os内核它是怎样运作起来的呢？



答案
---
容器是依赖宿主机内核的，如果内核不能满足容器需求，就不能run。 这么看来，其实docker也不是那么“万能”的。

甚至如果容器里需要使用高版本内核的时候，宿主机不能支持，那么这个容器就run不起来.............理论上是这样，用的时候就要注意了。不知道这种docker是不是会自动下载这个内核，然后完全单独运行的话，其实本质上，真的和虚拟机差不多了


都独立内核了，ns和cg都不能用宿主机的了，那就是彻底的虚拟机了。

docker 和 传统的虚拟机的区别就是 ， 传统虚拟机假装有一套新的硬件，系统跑在硬件上就行了， docker 假装有一个系统，进程跑在系统里就行了，其实个人认为准确说，应该是docker假装有一个内核。


翻译
---
docker最开始使用的是LXC，后来切换到了libcontainer。这样就可以跟宿主机共享操作系统资源。docker使用一种分层的文件系统AUFS，并且管理自己的网络。

AUFS是一个分层的文件系统，将只读的和可写的部分merge在一起。OS公用的地方是只读的，每个container自己挂载的是可写的。

假设我们有一个1g的image，需要1000个节点的扩容。如果使用一个完整的VM的话，需要1000G的空间。使用Docker的话，这1000个container节点可以共享1g的OS资源。

一个完整的VM拥有自己的一组资源，基本不共享什么东西。用重量换得了更多的独立性。Docker相反，独立性相对差一些，但是很轻量。所以可以在宿主机上跑上千个container。

一个完整的VM通常要花费几分钟才能启动，一个docker container则只需要几秒。

两个东西各有优缺点。


参考：
- https://stackoverflow.com/questions/33112137/run-different-linux-os-in-docker-container
- https://stackoverflow.com/questions/33112137/run-different-linux-os-in-docker-container
- https://stackoverflow.com/questions/16047306/how-is-docker-different-from-a-normal-virtual-machine
- https://docs.docker.com/engine/faq/#what-is-different-between-a-docker-container-and-a-vm
