
title: mysql从库crash
date: 2017-10-27 17:25:14
tags: [youdaonote]
---


 mysql主从架构，从库crash掉。恢复以后，出现主从同步错误问题。
 
 ```
 2017-10-27 16:23:38 2098 [ERROR] /server/mysql/bin/mysqld: Table './mysql/proc' is marked as crashed and should be repaired
2017-10-27 16:23:38 2098 [Warning] Checking table:   './mysql/proc'
2017-10-27 16:23:47 2098 [Warning] Access denied for user 'root'@'localhost' (using password: NO)
2017-10-27 16:30:03 2098 [Warning] Access denied for user 'root'@'localhost' (using password: NO)
2017-10-27 16:31:06 2098 [Warning] Slave SQL: If a crash happens this configuration does not guarantee that the relay log info will b
e consistent, Error_code: 0
2017-10-27 16:31:06 2098 [Note] Slave SQL thread initialized, starting replication in log 'mysql-bin.004001' at position 1000175864, 
relay log './prd-mysql03-relay-bin.000994' position: 4145
2017-10-27 16:31:06 2098 [ERROR] Slave SQL: Could not execute Write_rows event on table YKX_DW.JLC_APP_MARKPOINT_REALTIME; Duplicate 
entry '212630' for key 'PRIMARY', Error_code: 1062; handler error HA_ERR_FOUND_DUPP_KEY; the event's master log mysql-bin.004001, end
_log_pos 1000176163, Error_code: 1062
2017-10-27 16:31:06 2098 [Warning] Slave: Duplicate entry '212630' for key 'PRIMARY' Error_code: 1062
2017-10-27 16:31:06 2098 [ERROR] Error running query, slave SQL thread aborted. Fix the problem, and restart the slave SQL thread wit
h "SLAVE START". We stopped at log 'mysql-bin.004001' position 1000175864

 ```


之后`show slave status`
```
Slave_IO_Running: Yes  
Slave_SQL_Running: No
```

先stop slave，然后执行了一下提示的语句，就是把重复的主键记录删除，  
再SET GLOBAL SQL_SLAVE_SKIP_COUNTER=1; START SLAVE;

参考：http://www.linuxidc.com/Linux/2017-02/141056.htm

http://blog.csdn.net/donghaixiaolongwang/article/details/74923971
