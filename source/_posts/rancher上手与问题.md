
title: rancher上手与问题
date: 2017-01-02 15:26:07
tags: [youdaonote]
---

1. rancher里的host能不能同时属于env1和env2，测试了一下，貌似一个host上只能启动一个agent。

先把host1和host2添加到env1后，添加了wordpress的service stack，之后将所有服务都停掉，并deactivate两个hosts。

新建env2，将host1、host2同样的方法添加到env2中，布置mesos集群。

此时切回env1，再去激活host，发现已经不行了，host上提示disconnected。查看下面的container里，是有rancher agent的。有可能这个agent是在env2的那个把，所以不能连接到env1。


那：就是不能同时属于两个env咯？


2. 遇到get ip timeout问题，一般升级这个组件的版本就可以了
3. 如果我们升级了某个组件的版本，再去调整这个组件的scale，它竟然会先自动降级，然后再调整scale。然后版本就**又恢复成低版本**的了。 Orz....
4. 
