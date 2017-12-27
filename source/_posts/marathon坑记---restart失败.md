
title: marathon坑记---restart失败
date: 2017-09-08 17:35:04
tags: [youdaonote]
---

接上一篇的场景。

我们通过marathon的REST API修改某个app的json配置，然后会触发restart操作。

因为json里指定了host的port，所以会导致mesos agent的31111端口被占用，然后新的app就起不来，然后就一直处于waiting状态了。

这个被占用的信息可以从mesos master的/state接口里used_resources看到。

尝试把hostPort解放开，重新设置成0.....果然就成功了。
```
{
  "id": "basic-3",
  "cmd": "python3 -m http.server 8080",
  "cpus": 0.5,
  "mem": 32.0,
  "container": {
    "type": "DOCKER",
    "docker": {
      "image": "python:3",
      "network": "BRIDGE",
      "portMappings": [
        { "containerPort": 8080, "hostPort": 0 }
      ]
    }
  }
}
```

这样的话，对于服务的使用有以下几个方案
- HAProxy。需要在HAProxy注册每个service的端口就行了。hostPort不用开，都通过HAProxy去访问。
- 继续单个服务指定端口。需要每次都先将app缩容到0，再扩容回来。
- 使用smartstack或者consul等服务发现工具，对HAProxy进行动态修改。


我们的应用是监控平台，每个监控app只需要有一个实例，而且并不要求每个实例有严格高可用。所以暂时使用上面最简单的方案2.


