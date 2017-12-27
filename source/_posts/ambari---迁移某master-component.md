
title: ambari---迁移某master-component
date: 2017-03-27 14:12:55
tags: [youdaonote]
---

以AMBARI_METRICS服务的component：METRICS_COLLECTOR为例。



1. 停止对应service
```
curl -u admin:admin -H "X-Requested-By: ambari" -X PUT -d '{"RequestInfo":{"context":"Stop Service"},"Body":{"ServiceInfo":{"state":"INSTALLED"}}}' http://namenode01.will.com:8080/api/v1/clusters/datacenter/services/AMBARI_METRICS/
````

2. 确认目前node上的component已经停止
```
curl -u admin:admin -H "X-Requested-By: ambari" -X PUT -d '{"RequestInfo":{"context":"Stop Component"},"Body":{"HostRoles":{"state":"INSTALLED"}}}' http://namenode01.will.com:8080/api/v1/clusters/datacenter/hosts/datanode04.will.com/host_components/METRICS_COLLECTOR
```

3. 删除目前servcie里的component
```
curl -i -H "X-Requested-By:ambari" -u admin:admin -X DELETE http://namenode01.will.com:8080/api/v1/clusters/datacenter/services/AMBARI_METRICS/components/METRICS_COLLECTOR
```

4. 在service的元数据里再加上这个component
```
curl -i -H "X-Requested-By:ambari" -u admin:admin -X POST http://namenode01.will.com:8080/api/v1/clusters/datacenter/services/AMBARI_METRICS/components/METRICS_COLLECTOR 

```

5. 安装component到指定node上
```
curl -i -u admin:admin -H "X-Requested-By:ambari" -i -X POST -d '{"host_components" : [{"HostRoles":{"component_name":"METRICS_COLLECTOR"}}] }'  http://namenode01.will.com:8080/api/v1/clusters/datacenter/hosts?Hosts/host_name=datanode17.will.com
```

6. web端，到新的node上的component点击re-install，之后重启整个service。完成


ps: 
如果没有删除干净的话，就到指定node上删除component
```
curl -u admin:admin -H "X-Requested-By: ambari" -X DELETE http://namenode01.will.com:8080/api/v1/clusters/datacenter/hosts/datanode04.will.com/host_components/METRICS_COLLECTOR
```


参考：
- https://community.hortonworks.com/questions/82689/how-to-add-component-spark-jobhistoryserver-to-spa.html#answer-82695
- https://cwiki.apache.org/confluence/display/AMBARI/Using+APIs+to+delete+a+service+or+all+host+components+on+a+host
