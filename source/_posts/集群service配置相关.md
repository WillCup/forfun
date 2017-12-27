
title: 集群service配置相关
date: 2017-04-21 16:41:02
tags: [youdaonote]
---

YARN
---

服务名称 | 现有配置 | TO 
---|---| ---
resourcemanager | 1G | 4G
nodemanager | 1G    | 2G
timeline server | 1G    | 4G
resource + timline server = 2G


HDFS 
---
服务名称 | 现有配置 | TO
---|--- | ---
datanode | 1G  | 2G
namenode | 3G 
namenode * 2 =  6G


MAPRED
---
服务名称 | 现有配置 | TO
---|--- | ---
history server | 1G  | 2G

HIVE
---
服务名称 | 配置
---|---
hiveserver2 | 769M
metastore | 1G
hcatserver | 1G
3G

HBASE
---
服务名称 | 配置
---|---
regionserver | 2G
master | 1G
regionserver * 8 + master * 2 = 18G

ZK
---
服务名称 | 配置
---|---
zk server | 1G
zk * 3 = 3G

KAFKA
---
服务名称 | 配置
---|---
broker | 1G
broker * 3 =3G

SPAKR
---
服务名称 | 配置
---|---
spark history server  | 1G






需要copy的机器
---
以下机器copy完成后，全部将内存提升至32G、CPU提升到8核


##### 迁出机器

- datanode15.will.com     -> NM，DN
- 1.103-prd-datanode14.will.com   ->kafka, DN,NM
- 1.108-prd-datanode19.will.com -> regionserver, DN,NM
- 1.110-prd-datanode21.will.com -> NM, DN
- 1.99-prd-datanode10.will.com -> kafka, DB, NM
- 1.113-fengkong.data.com 风控 -> NONE
- 1.87-prd-datanode05.data.com -> DN, NM, regionserver
- 1.97-prd-datanode08.data.com -> DN, NM, regionserver
- 1.95-prd-datanode06.data.com -> kafka, DN, NM, SB HbaseMaster
- 1.96-prd-datanode07.data.com -> DN, NM, regionserver

原有机器迁出会占新资源的内存32G * 10 + CPU*80


##### 原有机器资源升级

在原有资源基础上，添加16G内存，CPU核数翻倍【把迁移出去的机器空闲出来的资源充分利用】：
- 1.107-prd-datanode18.will.com
- 1.109-prd-datanode20.will.com
- 1.102-prd-datanode13.will.com
- 1.106-prd-datanode17.will.com
- 1.100-prd-datanode11.will.com
- 1.98-prd-datanode09.will.com
- 1.110-prd-datanode21.will.com
- 1.86-prd-datanode04.data.com
- 1.82-prd-datanode01.data.com
- 1.83-prd-datanode02.data.com
- 1.84-prd-datanode03.data.com
- 1.105-prd-datanode16.will.com
- 1.101-prd-datanode12.will.com

理论上，对原有机器不会有额外资源要求，对新机器资源也不会有影响。

需要新建的机器
---


#### 新服务节点
可按照service01.will.com开始弄主机名。


单机配置： 32G内存 + 8核 + 50G硬【整机空间即可】 + CentOS release 6.8 (Final)

机器数量： 8台
共占：内存32G * 8 + CPU8 * 8 = 内存256G + CPU 64



#### 新数据节点

目前最新数据节点的主机名编号到了21，新机器可以datanode22.will.com开始。


单机配置：32G内存 + 8核CPU + 640G硬盘【/server/目录挂载空间】 + CentOS release 6.8 (Final)

机器数量： 12【如果物理机资源不够，可以暂时弄10台，以后酌情扩容】

共占：内存32G * 12 + CPU8 * 12 = 内存384G + CPU 96
