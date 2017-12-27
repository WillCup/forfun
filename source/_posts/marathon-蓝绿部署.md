
title: marathon-蓝绿部署
date: 2017-08-31 18:59:00
tags: [youdaonote]
---

蓝绿部署是一种安全部署app的方式，它会创建两个版本的app(BLUE和GREEN)。要部署一个新版本的app，应该先将所有旧版本的请求、挂起的操作等处理完，并且将流量定向到新版本app那边。蓝绿部署预估app的停掉的时间，允许我们在必要时快速从BLUE版本恢复。详细过程，参考[蓝绿部署](http://martinfowler.com/bliki/BlueGreenDeployment.html)

不错的解释：http://www.tuicool.com/articles/2Iji2ue

https://www.v2ex.com/t/344341


在生产环境中，最好将这个过程脚本化，将它集成进已经存在的部署系统中。下面，我们提供一个使用DC/OS CLI的例子。

过程
---
我们需要把BLUE，也就是当前版本的app，替换成GREEN(新版本)。

1. 在marathon上启动新版本的app。给app名字一个唯一的id，比如git的commit id。在这个例子中，我们把GREEN添加到新版本app的名字中：
```
# launch green
 dcos marathon app add green-myapp.json
```
注意：如果你使用的不是dcos的CLI，而是API，那命令会长很多。
```
curl -H "Content-Type: application/json" -X POST -d @green-myapp.json <hosturl>/marathon/v2/apps
```

2. 扩容GREEN的app。初始的时候一般是0，设置一下需要的实例数量。记住，现在还并没有流量过来：我们并没有到LB那边注册。

```
 # scale green
 dcos marathon app update /green-myapp instances=1
```

3. 等所有的GREEN app都健康部署。
```
 # wait until healthy
 dcos marathon app show /green-myapp | jq '.tasks[].healthCheckResults[] | select (.alive == false)'
```
4. 如果看到又不健康的，就中止部署，开始rollback
5. 把GREEN app加入到LB中
6. 选一个或多个BLUE app。
```
 # pick tasks from blue
 dcos marathon task list /blue-myapp
```
7. 更新LB，将这些BLUE app移除资源池
8. 等所有的BLUE app都被移除完，这个过程可以通过app的metrics终端API进行监控。
9. 一旦BLUE app的操作都完成，就杀掉BLUE app的进程，并缩容，避免被杀掉的重启.
```
 # kill and scale blue tasks
 echo "{\"ids\":[\"<task_id>\"]}" | curl -H "Content-Type: application/json" -X POST -d @- <hosturl>/marathon/v2/tasks/delete?scale=true
```
marathon会移除指定的所有的实例。上面的hosturl是master node所在的位置
10.重复步骤2-9，直到不再有BLUE app
11. 从marathon中移除BLUE app。
```
# remove blue
dcos marathon app remove /blue-myapp
```
