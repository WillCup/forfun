
title: compose版本
date: 2017-09-05 14:25:40
tags: [youdaonote]
---

compose文件是一个yaml文件，定义了docker app的service, network, volume等


有三个版本

- version1， 在YAML文件中别指定version就是了。
- version2.x。在YAML里指定version: '2',或者version: '2.1'
- version3.x。最新版本，也是最推荐使用的版本，可以兼容swarm模式。通过在YAML中指定version: '3'或者version: '3.1'声明。


如果使用了多个compose文件模式，或者用了extending service，那么每个文件的version声明应该是一样的。

Version1
---
不额外指定version的compose文件被看作"version1"。所有的service都被定义在yaml文件的root上。

version1是compose 1.6.x开始支持的，未来会被放弃。

version1文件不能声明volume, network或者build arguments。

compose不能使用网络：每个container都是连接在default bridge网络上的，相互之间用IP沟通。可以使用links来启用container之间的相互发现。

例子：
```
web:
  build: .
  ports:
   - "5000:5000"
  volumes:
   - .:/code
  links:
   - redis
redis:
  image: redis
```


version 2
---
compose文件中，必须在root指定相应version，而且所有的service都要定义在services下。

version 2要求compose 1.6.0+， docker engine版本1.10。0+。

可以通过volumes指定volume，也可以通过networks指定容器的网络。

默认情况下，每个container都加入一个app级别的默认网络，而且可以通过自身的hostname被发现。也就是说links就没啥必要了，详情[参考](https://docs.docker.com/compose/compose-file/compose-file-v2/#networking/)。

简单例子：
```
version: '2'
services:
  web:
    build: .
    ports:
     - "5000:5000"
    volumes:
     - .:/code
  redis:
    image: redis
```

稍微复杂点儿的例子, 定义了volumes和network
```
version: '2'
services:
  web:
    build: .
    ports:
     - "5000:5000"
    volumes:
     - .:/code
    networks:
      - front-tier
      - back-tier
  redis:
    image: redis
    volumes:
      - redis-data:/var/lib/redis
    networks:
      - back-tier
volumes:
  redis-data:
    driver: local
networks:
  front-tier:
    driver: bridge
  back-tier:
    driver: bridge
```

还有一些其他的选项来支持网络：
- aliases
- depends_on用来指定service之间的依赖关系
```
version: '2'
services:
  web:
    build: .
    depends_on:
      - db
      - redis
  redis:
    image: redis
  db:
    image: postgres
```
- ipv4_address, ipv6_address


Version2.1
---
是version2的升级版，需要docker engine是1.12.0+， compose版本是1.9.0+。

添加了以下参数
- link_local_ips
- isolation
- volume和network的labels
- volumes的name
- userns_mode
- healthcheck
- sysctls
- pids_limit


Version2.2
---
version2.1的升级，，需要docker engine是1.14.0+， compose版本是1.13.0+。允许指定默认的scale数。
添加了下面参数：
- init
- scale


version2.3
---
version2.2的升级，，需要docker engine是17.06.0+， compose版本是1.16.0+。

添加了下面参数:

- target for build configurations
- start_period for healthchecks


version3
---
主要是解决compose和docker swarm mode的兼容性问题。

- 移除选项：volume_driver, volumes_from, cpu_shares, cpu_quota, cpuset, mem_limit, memswap_limit, extends, group_add
- 添加选项：deploy


version3.3
---
version3的升级版。添加以下参数：
- build的labels
- credential_spec
- configs
- deploy endpoint_mode


