
title: slider服务发现
date: 2017-12-22 18:30:57
tags: [youdaonote]
---


#### 问题

- client可能是运行在cluster之外的
- 部署在yarn里的各个component的位置都是会变的
- 服务端口可以是固定的、或者可预测的

比如开始在client端指定了service的位置，但是如果service遇到失败自己重启了，这个位置可能就在yarn集群中变了。client自然就访问不到了。这种事情的发生几率跟集群稳定性有很大关系。


其他限制

- 降低已经存在app的变化次数，最好是一次都不要变。。这显然不现实
- 防止恶意注册service endpoint
- app和client都可以任意扩缩容

#### 可能的解决方案

###### zookeeper
client用zk来发现service，hbase就是这样的

###### DNS

###### 变化的IP地址

###### LDAP

###### 等等


我的场景就是使用slider在yarn上部署了presto，presto的coordinator的位置是会变化的，但是端口是固定的。在不修改hue的配置的前提下，怎样才能跟随presto变化呢？

个人认为官网给的点子都把问题复杂化了。

前面提到其实我们已经在hue的源码里自己创建presto的jdbc类，那么我们在建立session的时候，那个url通过爬取yarn上运行的我们指定presto app名字的位置的接口不就可以获取到当前在yarn里运行着的presto的coordinator了。聪明如我，嘎嘎。而且，hue本来就是要配置resourcemanager的地址，来抓取hive任务的执行进度的。

伪代码如下
```py
def creat_sesssion():
    url = request_yarn(options['presto_app_name'])
    session = JDBC(self.user, self.pwd, url..)

```

参考： https://slider.incubator.apache.org/design/registry/index.html
