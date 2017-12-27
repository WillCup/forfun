
title: docker容器的时间同步
date: 2017-12-12 16:02:33
tags: [youdaonote]
---

最近在测试环境上上了一个新的组件，发现docker容器内的时间是有问题的，就是总比真正的时间慢4分钟左右。然后就exec进去安装了ntpdate工具进行时间同步工作。


然后收到了如下信息：
```
root@1462539f1dc2:/# ntpdate -u cn.pool.ntp.org
12 Dec 15:57:35 ntpdate[502]: step-systime: Operation not permitted
```

看到是这个操作被禁止了，而且在docker里是用root执行的，竟然被禁止了。所以考虑到应该是docker对于苏初级的保护。返回宿主机，查看时间...果然与docker容器内一致，然后到宿主机安装ntpdate，进行时间同步之后，docker内时间自然就正常了。


回想一下docker原理：Docker本质上是宿主机（Linux）上的进程，通过namespace实现资源隔离，通过cgroups实现资源限制，通过写时复用机制(copy-on-write)实现高效的文件操作。归根到底还是调用的宿主机的东西，并不像虚拟机自己有一套完整的机制。



https://kasheemlew.github.io/2017/05/18/docker-linux/
