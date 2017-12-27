
title: mesos-+-marathon快速启动
date: 2017-09-14 11:14:38
tags: [youdaonote]
---

master
---
mesos_master_start.sh
```
export MESOS_cluster=will
export MESOS_zk=zk://10.1.5.65:2181/wesos
export MESOS_work_dir=/var/lib/mesos/master
export MESOS_quorum=1
export MESOS_log_dir=/server/mesos_log
```

```
 . mesos_master_start.sh && mesos-master
```


agent
---
通过zk与mesos master建立关系


mesos_agent_env.sh
```
export MESOS_executor_registration_timeout=5mins
export MESOS_containerizers=docker,mesos
export MESOS_isolation=cgroups/cpu,cgroups/mem
export MESOS_work_dir=/var/lib/mesos/agent
export MESOS_master=zk://10.1.5.65:2181/wesos
```

```
. mesos_agent_env.sh && mesos-agent --resources='ports:[10000-60000]'
```


marathon
---
通过zk与mesos master建立关系
```
marathon --master zk://10.1.5.65:2181/wesos --zk zk://10.1.5.65:2181/mt
```
