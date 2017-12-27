
title: docker问题整理
date: 2017-07-14 14:15:37
tags: [youdaonote]
---

容器内ping不同外部宿主机
---
默认使用bridge网络，应该是通畅的。通常就是简单的防火墙导致，需要注意的是：**调整防火墙之后，需要重启docker engine才能生效**，仅仅重启docker容器是不会起作用的。。。。

loopback devices的告警
---
```
WARNING: devicemapper: usage of loopback devices is strongly discouraged for production use.
         Use `--storage-opt dm.thinpooldev` to specify a custom block storage device.
WARNING: IPv4 forwarding is disabled
WARNING: bridge-nf-call-iptables is disabled
WARNING: bridge-nf-call-ip6tables is disabled

```

参考
 - http://www.dockerinfo.net/2182.html

容器名称变化
---
主要是对于代理层，比如有三个backend服务节点，只要一重启，这三个容器的host和ip有可能都会变。

方案：
 - 代理 + 动态服务发现。nginx/haproxy + consul/etcd
 - traefix，专为此种场景提供了定制接口配置。

参考： 
- http://blog.hypriot.com/post/microservices-bliss-with-docker-and-traefik/
- https://www.consul.io/intro/getting-started/install.html


devicemapper: Can't set cookie dm_task_set_cookie failed
---
```
docker: failed to register layer: devmapper: Error activating devmapper device for '24ea1e709520627a2862d2b9130b6aff77dde03fef39cc0169568cde5883b013': devicemapper: Can't set cookie dm_task_set_cookie failed.
```
问题貌似是在我重新启动

```
Error response from daemon: driver "devicemapper" failed to remove root filesystem for 1d7c4b287c27ef61fd0d9ac37c401c5da213b81b31dcc42448b1864b6b838f14: failed to remove device f45fd625ac4cdaccddfdd0ea02ac97f9e529a13594a4cfc92a883b8674d2658a: devicemapper: Can not set cookie: dm_task_set_cookie failed
```

临时方案：echo 'y' | sudo dmsetup udevcomplete_all
https://github.com/kubevirt/kubevirt/issues/321

动态配置
---
一共有三个方案可以支持
#### 使用自定义的DNS服务器
这个方案仅仅是应对容器内需要访问容器外的其他机器的host时起作用，如果还需要变化其他配置，那么可以忽略掉

#### 使用env或者env文件
1. 首先要把应用依赖的外部配置从代码里面抽出来，做成配置文件
2. 容器启动的时候，执行一个自定义脚本，脚本内容就是从容器系统环境变量里面读取环境变量，然后用sed替换配置文件里面的配置
3. 那么在docker run启动容器的阶段，通过-e选项传递你的配置值到容器的环境变量，就能修改容器内应用的配置
4. 迁移到其他环境，修改docker run -e的环境变量配置就OK


#### 直接使用-v挂载不同环境的配置文件
这种与第二种本质上没有太大区别。但是关键在于env或者env文件是在client端的，而-v挂载目录是需要在server端的。也就是是说每个server都需要有那个目录或者文件才行，而env方式的话，就只需要client端有就可以了。

鉴于server端一般是多于client端的，所以选用env的会相对多一些。


网络
---
docker attach容器后，退出的时候ctrl + q
#### docker默认的docker0
bridge是不支持服务发现的，也就在这个网络里的docker容器相互之间并不能通过名称通信。
要支持服务发现的话，需要自定义一个bridge网络。

只有swarm service可以连到overylay网络，独立的容器是连不上的。

docker是会动态修改宿主机的iptables规则的。

参考： https://docs.docker.com/engine/userguide/networking/


关闭防火墙引起的问题
---

在dockerd运行的时候，关闭防火墙的话，会清空iptables里的docker相关规则，导致docker容器启动失败。

centos7, docker ce17

```
 iptables: No chain/target/match by that name
```
解决办法，在关闭防火墙之后再启动dockerd，或者重启dockerd，它会自己根据现有docker网络和容器重建iptables相关规则。


devicemapper: Can not set cookie: dm_task_set_cookie failed
---
重命名container名称导致。

解决：
```
echo 'y' | sudo dmsetup udevcomplete_all
```

最好能执行`docker rm $(docker ps -a -q)`把现有container都移除。
