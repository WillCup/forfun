
title: GIAC-全球互联网架构大会
date: 2017-12-26 13:22:56
tags: [youdaonote]
---

#### 摩拜

新上的火爆活动，需要后端扩容 —— 使用swarm支持动态扩缩容

国际化带来的问题：欧盟数据保护规范：GDPR

使用腾讯云在全球主要位置部署了多数据中心。

多集群、同城多活，也是通过腾讯云给的解决方案搞定

多集群、异地多活。可以缓存的全部引入CDN


dbproxy. mysql分表分库->TiDB

DRC 数据复制中心


Mysql -> kafka -> xxx

随着开发人员增多，代码合并后的质量很成问题。 -> 拆分spring cloud微服务 + netflix


nginx + lua [路由控制、负载均衡、服务发现] -> spring zuul [目前的问题：同步IO，占用CPU内存较高]

spring config 、 consul + vault进行加密控制，后面接git、svn、文件

feign client + consul

内部通信使用grpc

容器问题【端口协调、依赖ansible、】

docker swarm的overlay网络开销比较大，使用了腾讯的swarm服务，基于docker swarm有了一些改进。

mongo是有可能有安全漏洞的，摩拜只是使用mongo来存储地理位置信息，因为对安全不太相关，最多就车的信息被泄露出去。同时也有网络隔离等安全措施

所有的服务都应该是无状态的，可以飘移到任何地方，这在最开始的时候就已经有了这个考虑。

服务化扩展使用docker swarm实现，持久化数据的扩展使用TiDB、codis等易于扩展的数据库【分表分库对于一些很复杂的连接sql，外键关联等业务不能支持很好】。

不可能所有服务都满足CAP，所以先需要对事务进行服务定级。



