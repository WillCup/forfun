
title: ambari个别component扩张
date: 2017-05-09 11:08:30
tags: [youdaonote]
---

需要扩展20个NodeManager，ambari只能手动到每个datanode界面上去安装NodeManager，并启动。啰嗦，麻烦。

看了一下ajax请求，整理了一个脚本。
```
#!/bin/bash

for i in `seq 26 33`;
do

echo handing $i

# 安装service
curl -u admin:will@bj15. -H "X-Requested-By: ambari" -X POST -d '
{"RequestInfo":{"context":"Install NodeManager"},"Body":{"host_components":[{"HostRoles":{"component_name":"NODEMANAGER"}}]}}' http://namenode01.will.com:8080/api/v1/clusters/datacenter/hosts?Hosts/host_name=datanode${i}.will.com




# 安装component
curl -u admin:will@bj15. -H "X-Requested-By: ambari" -X PUT -d '
{"RequestInfo":{"context":"Install NodeManager","operation_level":{"level":"HOST_COMPONENT","cluster_name":"datacenter","host_name":"datanode${i}.will.com","service_name":"YARN"}},"Body":{"HostRoles":{"state":"INSTALLED"}}}' http://namenode01.will.com:8080/api/v1/clusters/datacenter/hosts/datanode${i}.will.com/host_components/NODEMANAGER?HostRoles/state=INIT


# 启动
curl -u admin:will@bj15. -H "X-Requested-By: ambari" -X PUT -d '{"RequestInfo":{"context":"Start NodeManager","operation_level":{"level":"HOST_COMPONENT","cluster_name":"datacenter","host_name":"datanode${i}.will.com","service_name":"YARN"}},"Body":{"HostRoles":{"state":"STARTED"}}}' http://namenode01.will.com:8080/api/v1/clusters/datacenter/hosts/datanode${i}.will.com/host_components/NODEMANAGER
done


```
