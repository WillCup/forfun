
title: mesos的docker化安装部署
date: 2016-12-22 17:38:06
tags: [youdaonote]
---


首先，还是从源头说起，用阿里云的yum源，下载安装都会快一些。
zookeeper
```
docker run -d  --net="host"   -e SERVER_ID=3 -e ADDITIONAL_ZOOKEEPER_1=server.1=10.1.5.129:2888:3888 -e ADDITIONAL_ZOOKEEPER_2=server.2=10.1.5.180:2888:3888 -e ADDITIONAL_ZOOKEEPER_3=server.3=10.1.5.190:2888:3888 --name zk garland/zookeeper
```

master
```
docker run --net="host" \
   -p 5050:5050 \
   -e "MESOS_HOSTNAME=10.1.5.129" \
  -e "MESOS_IP=10.1.5.129" \
  -e "MESOS_ZK=zk://10.1.5.129:2181,10.1.5.180:2181,10.1.5.190:2181/mesos" \
  -e "MESOS_PORT=5050" \
   -e "MESOS_LOG_DIR=/var/log/mesos" \
   -e "MESOS_QUORUM=1" \
   -e "MESOS_REGISTRY=in_memory" \
   -e "MESOS_WORK_DIR=/var/lib/mesos" \
   -d --name mesos-master garland/mesosphere-docker-mesos-master
   
docker run --net="host" \
   -p 5050:5050 \
   -e "MESOS_HOSTNAME=10.1.5.180" \
  -e "MESOS_IP=10.1.5.180" \
  -e "MESOS_ZK=zk://10.1.5.129:2181,10.1.5.180:2181,10.1.5.190:2181/mesos" \
  -e "MESOS_PORT=5050" \
   -e "MESOS_LOG_DIR=/var/log/mesos" \
   -e "MESOS_QUORUM=1" \
   -e "MESOS_REGISTRY=in_memory" \
   -e "MESOS_WORK_DIR=/var/lib/mesos" \
   -d --name mesos-master garland/mesosphere-docker-mesos-master
   
   
 docker run --net="host" \
   -p 5050:5050 \
   -e "MESOS_HOSTNAME=10.1.5.190" \
  -e "MESOS_IP=10.1.5.190" \
  -e "MESOS_ZK=zk://10.1.5.129:2181,10.1.5.180:2181,10.1.5.190:2181/mesos" \
  -e "MESOS_PORT=5050" \
   -e "MESOS_LOG_DIR=/var/log/mesos" \
   -e "MESOS_QUORUM=1" \
   -e "MESOS_REGISTRY=in_memory" \
   -e "MESOS_WORK_DIR=/var/lib/mesos" \
   -d --name mesos-master garland/mesosphere-docker-mesos-master
```

marathon
```
docker run \
 -d \
 -p 8080:8080 \
 garland/mesosphere-docker-marathon --master zk://10.1.5.129:2181,10.1.5.180:2181,10.1.5.190:2181/mesos --zk zk://10.1.5.129:2181,10.1.5.180:2181,10.1.5.190:2181/marathon
```


slave
```
docker run -d \
 --entrypoint="mesos-slave" \
 -e "MESOS_MASTER=zk://10.1.5.129:2181,10.1.5.180:2181,10.1.5.190:2181/mesos" \
 -e "MESOS_LOG_DIR=/var/log/mesos" \
 -e "MESOS_LOGGING_LEVEL=INFO" \
 garland/mesosphere-docker-mesos-master:latest
```

参考：
- http://www.cnblogs.com/ee900222/p/docker_2.html
- https://mesosphere.com/blog/2014/07/17/mesosphere-package-repositories/#
- http://www.jingyuyun.com/article/8062.html
- http://dockone.io/article/493
- http://www.mesoscn.cn/document/runing-Mesos/Configuration.html
