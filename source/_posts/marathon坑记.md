
title: marathon坑记
date: 2017-08-30 14:17:42
tags: [youdaonote]
---

资源不足
---
#### 场景

运行marathon官网提供的docker get started案例(https://mesosphere.github.io/marathon/docs/application-basics.html)可以成功。
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
我自己提了一个,屡次卡住不能deploy成功。
```
{
  "id": "/docker-tt",
  "cmd": null,
  "cpus": 1,
  "mem": 128,
  "disk": 0,
  "instances": 1,
  "acceptedResourceRoles": [
    "*"
  ],
  "container": {
    "type": "DOCKER",
    "volumes": [],
    "docker": {
      "image": "nginx:stable-alpine",
      "network": "BRIDGE",
      "portMappings": [
        {
          "containerPort": 80,
          "hostPort": 880,
          "servicePort": 10000,
          "protocol": "tcp",
          "name": "nn",
          "labels": {}
        }
      ],
      "privileged": false,
      "parameters": [],
      "forcePullImage": false
    }
  },
  "portDefinitions": [
    {
      "port": 10000,
      "protocol": "tcp",
      "name": "default",
      "labels": {}
    }
  ]
}
```


参考了官网的[这个链接](https://mesosphere.github.io/marathon/docs/troubleshooting.html#an-app-stays-in-quot-waiting-quot-forever) 的第一个，提示说：如果app一直都处于**waiting**的状态，那么很可能是由于资源不足的造成的，例如CPU，内存，磁盘等。

最后还说，如果还找不到问题，就到github提一个issue，把/state的内容贴近issue，这样就可以让社区的人帮忙分析。 


按照提示，我么有找到任何资源匮乏，当前的framework占用资源都是0.在查看slaves提供的物理资源的时候被亮瞎了狗眼：
```
"resources": {
    "disk": 12758,
    "mem": 2767,
    "gpus": 0,
    "cpus": 2,
    "ports": "[31000-32000]"
},
```

注意**ports**.....每个slave为什么要把自己的端口范围也限制啊......跪了。把上面的docker启动host端口改为31111后，测试通过，成功部署了nginx服务。


#### 其他
[官网端口映射相关](https://mesosphere.github.io/marathon/docs/ports.html)


portMapping： 对于所有需要给外部访问的docker容器，端口映射都很重要。一个端口映射是一个包含host port、container port、service port以及protocaol的tuple。有时也需要多个端口映射定义。不固定的hostPort默认是0，这样marathons会随机选一个端口。在Docker的USER模式下，hostPort是几乎不变的：hostport并不是必要的，如果没有制定，那么marathon不会自动随机选一个。这样就允许container可以在USER网络里发布，只包含containerPort和发现信息，但是不暴露这些端口给hsot网络。

portDefinitions： 一个数据定义了端口资源池。只在使用HOST网络，并且没有指定portMapping的时候是必须要有的。

servicePort：当在marathon创建新的app时，要为其指定一个或多个service port。你可以指定合理的port，也可以设为0让marathon自动分配可用端口。如果自己选择，就需要确保这个端口在所有app中是唯一的。

如果containerport和hostport同时被设置为0，那么他俩的端口虽然是会被随机分配，但是他俩是一样的，比如8888，那么就都是8888。


##### 配置实例
###### host
host网络模式是默认的网络模式，也是非docker app的唯一网络模式。注意在你的dockerfile里expose端口并不是必须的。

###### 示例
```
  "container": {
    "type": "DOCKER",
    "docker": {
      "image": "my-image:1.0",
      "network": "HOST"
    }
  },
```

指定端口的两种方式
```
"ports": [
        2001, 2002, 3000
    ],
```
```
"portDefinitions": [
        {"port": 2001}, {"port": 2002}, {"port": 3000}
    ],
```

这时环境变量$PORT0,$PORT1, $PORT2就分别是2001，2002，3000，如果都是0的话就都是随机分配的。

另外，保证一个服务发现程序例如haproxy从service port请求host port也是必须的。那么我们或许希望这个app的service ports与host ports保持一致，那么就可以设置requirePorts为true。这样marathons就只调度这个app到有这些端口的agent上了。 
```
"ports": [
        2001, 2002, 3000
    ],
    "requirePorts" : true
```
现在，service 和host的ports就都是2001， 2002， 3000了。

定义portDefinitions队列可以让我们制定protocol，每个port都有一个名字和自己的label。在启动新的task的时候，marathon会把metadata发送给mesos。mesos会在这个task的discovery字段中暴露这些信息。自定义网络发现服务可以读取这个字段，发现app服务。

```
   "portDefinitions": [
        {
            "port": 0,
            "protocol": "tcp",
            "name": "http",
            "labels": {"VIP_0": "10.0.0.1:80"}
        }
    ],
```

这是一个动态的tcp端口，name为http，并包含一个label。 port字段是必须的，protocol、name、labels是可选的。如果portDefinitions中只定义port字段的时候，其实跟定义ports数组没有分别。

注意：如果ports和portDefinitions内容不一致，那么就不能同时定义。


##### bridge 

bridge网络模型允许映射hostport到内部的container端口，而且只能用于docker container。通常在使用一个拥有固定端口的docker image的时候只用这种方式。

```
 "container": {
    "type": "DOCKER",
    "docker": {
      "image": "my-image:1.0",
      "network": "BRIDGE"
    }
  },
```

指定USER网络
```
"container": {
    "type": "DOCKER",
    "docker": {
      "image": "my-image:1.0",
      "network": "USER"
    }
  },
  "ipAddress": {
    "networkName": "someUserNetwork"
  }
```

指定端口

端口隐射与把-p参数传递给docker命令行差不多。
```
  "container": {
    "type": "DOCKER",
    "docker": {
      "image": "my-image:1.0",
      "network": "BRIDGE",
      "portMappings": [
        { "containerPort": 0, "hostPort": 0 },
        { "containerPort": 0, "hostPort": 0 },
        { "containerPort": 0, "hostPort": 0 }
      ]
    }
  },
```

上面的例子中，containerPort会和hostPort一致，都是随机的。


```
  "container": {
    "type": "DOCKER",
    "docker": {
      "image": "my-image:1.0",
      "network": "BRIDGE",
      "portMappings": [
        { "containerPort": 80, "hostPort": 0 },
        { "containerPort": 443, "hostPort": 0 },
        { "containerPort": 4000, "hostPort": 0 }
      ]
    }
  },
```

这个例子，marathon会随机分配端口来与容器的80，443，4000进行对应。

marathon会为每port都穿件service port。service port用来给服务发现程序使用，通常用一些常见的port。
```
  "container": {
    "type": "DOCKER",
    "docker": {
      "image": "my-image:1.0",
      "network": "BRIDGE",
      "portMappings": [
        { "containerPort": 80, "hostPort": 0, "protocol": "tcp", "servicePort": 2000 },
        { "containerPort": 443, "hostPort": 0, "protocol": "tcp", "servicePort": 2001 },
        { "containerPort": 4000, "hostPort": 0, "protocol": "udp", "servicePort": 3000}
      ]
    }
  },
```

host port还是随机的，但是service port固定了。HAProxy就可以配置路由从这些
service port 到host port 了。
