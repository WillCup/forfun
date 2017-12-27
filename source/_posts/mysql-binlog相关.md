
title: mysql-binlog相关
date: 2017-04-16 16:29:19
tags: [youdaonote]
---

binlog的记录格式有三种：
-  基于sql语句(statement-based replication,SBR)
-  基于行(row-based replication,RBR)
-  混合模式(mixed-based replication,MBR)

查看所有的binlog文件列表
```
show binary logs;
You are not using binary logging
```

代表并没有开启，需要编辑一下mysql的配置文件并重启。


