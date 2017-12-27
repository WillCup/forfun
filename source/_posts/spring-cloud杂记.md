
title: spring-cloud杂记
date: 2016-12-05 17:05:26
tags: [youdaonote]
---

Eureka
---
云应用中，服务通常并不长久运行的。我们也不希望记录每个service的IP和port。它的功能有些类似一个DNS服务。

- 发现与定位service
- 负载均衡
- 中间件容错


看了一些文章， 跟我以前写的smart agent的master和plugin模式基本雷同，就是想master注册自己，挂掉的时候remove掉，定时发送心跳等等。

类似功能的还有Consul


https://github.com/Netflix/eureka/wiki


spring cloud配置文件
---
默认service都是连接8888端口的configserver，要是想修改的话就需要使用bootstrap.properties外部配置文件。

```
spring.cloud.config.uri: http://myconfigserver.com
```

bootstrap的优先级高于configserver的application配置。

https://cloud.spring.io/spring-cloud-config/spring-cloud-config.html

zuul
---
地狱的守门怪兽

- 安全认证
- 动态路由
- 监视
- 压测
- 降载
- 静态资源处理
- 多区域

https://github.com/Netflix/zuul/wiki


zipkin
---
分布式中的神捕，对分布式调度的追踪，分布式追踪系统。可获取各个微服务接口调用的延时。基于Google论文[dapper](http://research.google.com/pubs/pub36356.html)设计完成。

应用的执行耗时数据会保存在zipkin里。zipkin UI也会呈现出每个服务request之间的依赖关系图。特别便于最终错误或者延时问题。

本质其实跟以前我们在云智慧做的工作一样，在request上打标记。

存储可以选择内存、JDBC、Cassandra、ES。

http://zipkin.io/
http://zipkin.io/pages/architecture.html

oauth2
---
分布式系统中，client与service是交叉访问的，需要解决一个问题：哪些client有权限访问哪些service？解决的方式就是单点登录：所有的请求某个service都需要一个token，这个token由一个统一的中心认证服务提供。我们这里就使用oauth2来做这个中心认证服务。



总结
---

configserver为一切service服务，提供各个服务最新的配置信息。

eureka-service提供服务注册与发现。所有的client都来它这里找service，然后再执行RPC请求。

reservation-service是一个服务，通过configserver找到eureka，注册到eureka，提供服务。

reservation-client是一个客户端，先通过configserver找到eureka，找到要请求的reservation-service，再请求具体路径(整个请求可以通过zuul代理实现)。
