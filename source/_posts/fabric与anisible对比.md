
title: fabric与anisible对比
date: 2017-05-26 11:18:37
tags: [youdaonote]
---

主要是跟anisible做对比，搞数据集群的环境越来越大，机器越来越多，不弄点儿自动化维护的工具，真心折腾不过来，而且容易出现失误。


这两天陪媳妇去图书馆自习，我就看书，看了一本不错的小说《兄弟》：余华写的，相当不错，笑中带泪，奔放中带着坚韧。因为近期工作中想要用一下docker来部署生产环境，为算法的同学提供动态可自定义的各种语言程序运行环境。前面听过几场关于docker环境部署的，听过几个人使用anisible进行docker部署，所以就在图书馆拿了本《奔跑吧：anisible》。

吸引
---
里面提到anisible的我看中的功能性卖点：
- host分组
- 任务有顺序依赖的执行
- 获取上一个task的输出，比如docker容器mysql部署后，把这个容器的id保存给下一个容器web服务的配置文件使用。
- 配置文件模板化，接受外来变量赋值后，copy进入指定主机的指定目录。

问题
---
#### 概念
但是有一个比较恶心的问题，要记住一些概念
- inventory 主机
- task
    - module 具体执行的一些功能模块
- handler 等待被task触发
- fact 主机相关的一些信息，比如CPU、内存、ip啥的

#### 问题
- module之类的都是别人封装好的，自己要完全按照人家的规则走，不舒服
- 有些需要额外处理的东西还是要自己写python代码处理，因为有些自定义的处理逻辑，是不会存在于既有的module中的

那么恶心的事情就出现了：既要去使用module中别人给自己制定的规则， 又他么要自己写代码完成自己的逻辑。还不如fabric，完全按照自己的代码去执行逻辑。


fabric
---

今儿早上看了一下fabric
- 主机分组 【Y：有role进行分组】
- 任务 【Y】
- 任务依赖 【Y：通过定义额外的任务组合惹怒我】
- 主机信息【并没有感觉有什么用】
- **task间变量传递**【上一个任务给环境中的变量赋值，下一个任务可以直接接收到，比如dict】 —— **失败**

直接上自己的脚本文件
```
from fabric.api import *

# 不指定用户的话，就是当前用户，跟ssh一样
env.hosts = [
    'user@192.168.1.1',
    'user@192.168.1.2',
]

# 必须全面指定 ： user@ip:port
env.passwords = {
    'user@192.168.1.1:22': 'password1',
    'user@192.168.1.2:22': 'password2',
}

# 这个标识有么有都可以 1.6.3版本
@task
def echo():
    run('echo "hello,world"')
```

动态为自己的机器分组, 那么就在fabric.py里执行以下逻辑：
```
services = []
datanodes = []
hosts = list()

def get_host():
    with open('/etc/hosts', 'r') as hf:
        for line in hf.readlines():
            ss = line.split()
            if 'node' in ss[1]:
                hosts.append(ss[0])
            if 'servicenode' in ss[1]:
                services.append(ss[0])
            if 'datanode' in ss[1]:
                datanodes.append(ss[0])
    hosts.remove('10.2.19.104')

env.hosts=hosts

env.roledefs = {
    'services': services,
    'datanodes': datanodes
}

```
其实这里的/etc/hosts就相当于anisible的inventory文件了。

调用的时候指定host组：
```
fab cmd:roles=services,cmd='df -h'
```


至此：放弃fabric......不能支持依赖部署，或者容器编排。离开......【尝试在任务重使用变量赋值失败，因为每个remote机器赋值都是对于自己的，并不能对全局变量赋值。但是其实通过获取任务的output，所以不放弃了】


稻草一枚：
- http://fabric-chs.readthedocs.io/zh_CN/chs/api/core/tasks.html
- https://stackoverflow.com/questions/42946197/return-value-from-fabric-task
- 



并行任务
---
fabric现在也支持并行执行task，并且提供一次并行多少的粒度
```
from fabric.api import *

@parallel(pool_size=5)
def heavy_task():
    # lots of heavy local lifting or lots of IO here
    
```
指定一次并行5个远程机器
```
 fab -P -z 5 heavy_task
```
http://docs.fabfile.org/en/1.13/usage/parallel.html

获取任务结果
---
```
from fabric.api import task, execute, run, runs_once

@task
def workhorse():
    return run("get my infos")

@task
@runs_once
def go():
    results = execute(workhorse)
    print results
```

注意：execute是在任务在每个remote host上执行完之后获取结果，而run的stdout则是在单个任务内返回。

http://docs.fabfile.org/en/1.13/usage/execution.html#intelligently-executing-tasks-with-execute

文件传递
---
使用put放到远程机器
```
with cd('/tmp'):
    put('/path/to/local/test.txt', 'files')
    
put('bin/project.zip', '/tmp/project.zip')
put('*.py', 'cgi-bin/')
put('index.html', 'index.html', mode=0755)
```

使用get获取远程机器文件
```
get('/path/to/remote_file.txt', 'local_directory') 
```

上面的put对比anisible来说，只有一方面不足，就是对于类似nginx配置文件中的一些变量信息，我们需要额外自己引入模板，渲染之后再put出去。


