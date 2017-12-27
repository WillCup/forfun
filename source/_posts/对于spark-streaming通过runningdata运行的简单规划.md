
title: 对于spark-streaming通过runningdata运行的简单规划
date: 2017-10-26 18:41:07
tags: [youdaonote]
---



#### 提交流程
1. 添加spark jar
2. 添加参数
3. 配置调度


#### 后台流程
1. 扫描updatetime，检测新更新的spark streaming程序
2. 检查jar包修改，将修改后的jar包重新上传到HDFS/mesos集群公共访问部分
3. 创建/修改marathon app配置
4. 重启/启动marathon app


#### 日志检查

这方面其实marathon已经做得很好了。可以直接拼接marathon的日志API搞定。


marathon api
```
http://10.2.19.125:5051/files/download?path=/server/mesos/agent/slaves/5fc969e7-c2da-4435-a0cb-278240c52f25-S2/frameworks/d2f4a97d-de35-4d1b-bf6d-07b8f2ad7666-0000/executors/willsparkstreamingtest01.cfcd8dde-ba34-11e7-9beb-3e15ac098cb2/runs/1128e325-ad92-4266-8348-04f43cfaa4c4/stderr
```


mesos api
```
http://10.2.19.124:5050/#/agents/5fc969e7-c2da-4435-a0cb-278240c52f25-S2/browse?path=/server/mesos/agent/slaves/5fc969e7-c2da-4435-a0cb-278240c52f25-S2/frameworks/d2f4a97d-de35-4d1b-bf6d-07b8f2ad7666-0000/executors/willsparkstreamingtest01.cfcd8dde-ba34-11e7-9beb-3e15ac098cb2/runs/latest
```

mesos 增量

```
http://10.2.19.125:5051/files/read?path=/server/mesos/agent/slaves/5fc969e7-c2da-4435-a0cb-278240c52f25-S2/frameworks/d2f4a97d-de35-4d1b-bf6d-07b8f2ad7666-0000/executors/willsparkstreamingtest01.cfcd8dde-ba34-11e7-9beb-3e15ac098cb2/runs/latest/stdout&offset=115771&length=50000&jsonp=jQuery17109253805122640539_1509014182353&_=1509014197811
```

mesos 下载
```
http://10.2.19.125:5051/files/download?path=/server/mesos/agent/slaves/5fc969e7-c2da-4435-a0cb-278240c52f25-S2/frameworks/d2f4a97d-de35-4d1b-bf6d-07b8f2ad7666-0000/executors/willsparkstreamingtest01.cfcd8dde-ba34-11e7-9beb-3e15ac098cb2/runs/latest/stdout
```


因为有增量，所以我们会优先考虑过滤改造mesos的API。

对于上面所有的路径信息，
主要就是framework的id，mesos salve的id，task的id。

slaveid, taskid可以通过marathon的task接口获取到。
```
curl http://10.2.19.124:8080/v2/apps/willsparkstreamingtest01
```

framework的ID可以去[mesos的api](http://mesos.apache.org/documentation/latest/endpoints/master/frameworks/)里找到。其实后面这个就包含了上面的所有信息了。

另外，考虑到可能出现程序错误，导致app在marathon上不停部署，那么此时我们是需要获取前面失败的log的。失败log其实也是一次部署的log，所以路径上没有什么区别，但是可能taskid会有所变化，仔细观察上面的数据里其实是有lastTaskFailure的信息的，可以找到上个失败task的id，进而找到失败的log。
