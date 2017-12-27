
title: 一次查询hang事件
date: 2017-10-16 15:56:14
tags: [youdaonote]
---

场景
---
同事在navicat上为table A修改了了一个字段长度，然后navicati卡死了。重启后这个表就不能查询了。


排查
---
```
show processlist
```
查看慢查询发现有status为Waiting for table metadata lock的查询出现。


查看运行中的事务
```
select * from information_schema.innodb_trx
```

根据trx_started找到运行时间最长的kill掉慢查询对应的trx_mysql_thread_id，之后就恢复了。
