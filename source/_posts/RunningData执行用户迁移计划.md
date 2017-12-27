
title: RunningData执行用户迁移计划
date: 2017-07-03 14:41:20
tags: [youdaonote]
---

超级用户准备
---
- jlc
- wplan
- xiaov



文件权限
---
#### HDFS
- 所有数仓调整owner为所在事业部的超级用户
 ```
 hdfs dfs -chown jlc:jlc -R /log/statistics
 
 
 hdfs dfs -ls /apps/hive/warehouse/ | awk '{print("hdfs dfs -chown -R ", $4, ":",$4, $8)}'
 ```

#### 本地
名称 | 位置 | 操作q
---| --- | ---
sqoop目录权限 | /server/app/sqoop | rm -vfr /server/app/sqoop/vo
|| /server/metamap/metamap_django | /server/metamap/metamap_django/*.java


azkaban执行用户
---
- sqoop
- hive

需要修改M2H，H2H，H2M以及新版本的generate_job_file里的执行用户，包含两个地方
 - 生成执行脚本
 - 生成命令【测试执行】

在settings文件中添加参数USE_ROOT

jenkins调度
---



资产匹配问题: 62远程调用106上的脚步任务
```
Message from syslogd@datanode17 at Jul  3 15:50:52 ...
 kernel:BUG: soft lockup - CPU#7 stuck for 67s! [python:2708]
```
