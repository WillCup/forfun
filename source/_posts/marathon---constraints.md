
title: marathon---constraints
date: 2017-09-07 14:17:50
tags: [youdaonote]
---

constraint控制app运行的位置，可以用来调优容错以及本地部署。constraint包含3个部分：fieldname， operator， 可选参数。field可以是agent node 的hostname或者agent node的其他属性。


Fields
---
#### Hostname
支持所有的marathon operator 。

#### Attribute
如果filed name不是hostname，就会被看作一个mesos agent node 的attribute。一个mesos agent attribute可以用来给agent node打tag。可以看一下mesos-slave --help.

如果制定的attribute还没有在agent node上定义，那么多数的operator都会拒绝在agent node上运行任务。目前是有UNLIKE operator会接受这个offer，其他的都会拒绝。


attribute field支持marathon所有的operator。

Marathon 支持text，scalar，range以及set类型的attribute的值。对于scalar、range、set类型的值，marathon会基于格式化的值执行一个字符串对比操作。对于range和set类型，格式分别是[begin-end,....]和{item,....}。例如: [100-200]和{a,b,c]

LIKE和UNLIKE的operator支持正则表达式。


Operator
---
#### UNIQUE operator
UNIQUE是告诉marathon对于一个app的所有task保证attribute的唯一性。例如下面的contraint保证了每个host上只运行一个app任务。

```
 curl -X POST -H "Content-type: application/json" localhost:8080/v2/apps -d '{
    "id": "sleep-unique",
    "cmd": "sleep 60",
    "instances": 3,
    "constraints": [["hostname", "UNIQUE"]]
  }'
```


#### CLUSTER operator
CLUSTER可以让app 的task运行在共享指定attribute的agent node上。对于有硬件要求的任务比较适用，或者想要把任务集中运行在同一个机栈中。
```
curl -X POST -H "Content-type: application/json" localhost:8080/v2/apps -d '{
    "id": "sleep-cluster",
    "cmd": "sleep 60",
    "instances": 3,
    "constraints": [["rack_id", "CLUSTER", "rack-1"]]
  }'
```

也可以指定一个hostname，把app绑定在指定的一个机器上。
```
curl -X POST -H "Content-type: application/json" localhost:8080/v2/apps -d '{
    "id": "sleep-cluster",
    "cmd": "sleep 60",
    "instances": 3,
    "constraints": [["hostname", "CLUSTER", "a.specific.node.com"]]
  }'
```

#### GROUP_BY operator
风来在不同的rack或者数据中心中均匀地分配任务，实现HA。

```
curl -X POST -H "Content-type: application/json" localhost:8080/v2/apps -d '{
    "id": "sleep-group-by",
    "cmd": "sleep 60",
    "instances": 3,
    "constraints": [["rack_id", "GROUP_BY"]]
  }'
```

marathon会分析mesos进来的offer，获取不同的rack_id属性。如果任务没有覆盖到所有的值，那么在constraint中指定一个数。如果不指定，可能会出现任务只被部署到一个rack的情况。
```
curl -X POST -H "Content-type: application/json" localhost:8080/v2/apps -d '{
    "id": "sleep-group-by",
    "cmd": "sleep 60",
    "instances": 3,
    "constraints": [["rack_id", "GROUP_BY", "3"]]
  }'
```

#### LIKE operator
接收一个正则表达式作为参数，允许filed值满足这个正则的所有agent node运行当前任务。
```
curl -X POST -H "Content-type: application/json" localhost:8080/v2/apps -d '{
    "id": "sleep-group-by",
    "cmd": "sleep 60",
    "instances": 3,
    "constraints": [["rack_id", "LIKE", "rack-[1-3]"]]
  }'
```

注意，参数一定要有，不然会有警告。

#### UNLIKE operator
和LIKE operator一样的道理。

```
 curl -X POST -H "Content-type: application/json" localhost:8080/v2/apps -d '{
    "id": "sleep-group-by",
    "cmd": "sleep 60",
    "instances": 3,
    "constraints": [["rack_id", "UNLIKE", "rack-[7-9]"]]
  }'
```


#### MAX_PER operator
接受一个数字参数，制定每个group的最大大小。可以用来限制跨rack或者数据中心的任务数量。


```
 curl -X POST -H "Content-type: application/json" localhost:8080/v2/apps -d '{
    "id": "sleep-group-by",
    "cmd": "sleep 60",
    "instances": 3,
    "constraints": [["rack_id", "MAX_PER", "2"]]
  }'
```


