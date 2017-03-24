title: iptables手记
date: 2017-03-24 14:56:16
tags: [iptables, linux]
---

先看一下网络流程图，以及所有的控制点
![网络流程图](/imgs/iptables/iptables_controls.png)
![网络流程细节图](/imgs/iptables/iptables_control_detail.png)
![控制点规则结构](/imgs/iptables/iptables_2.png)
![filter表相关流程与控制点图](/imgs/iptables/iptables_3_filter.png)
![mangle表相关流程与控制点](/imgs/iptables/iptables_3_mangle.png)
![nat表相关流程与控制点](/imgs/iptables/iptables_3_nat.png)
![语法结构](/imgs/iptables/iptables_4_command.png)
![语法结构](/imgs/iptables/iptables_4_command1.png)



查看filter表里的数据
```
[root@schedule ~]#iptables -t filter -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```

添加一条规则， 在OUTPUT上记录所有tcp的日志，默认是添加在syslog里的。
```
[root@data-test03 hadoop-client]# iptables -t filter -A OUTPUT -p tcp -j LOG
[root@data-test03 hadoop-client]# iptables -t filter -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
LOG        tcp  --  anywhere             anywhere            LOG level warning
```

找一下syslog里是否生效：
```
[root@data-test03 hadoop-client]# tail -n 10 /var/log/messages
Mar 23 18:30:35 data-test03 kernel: IN= OUT=eth0 SRC=10.1.5.81 DST=10.1.5.237 LEN=136 TOS=0x10 PREC=0x00 TTL=64 ID=32153 DF PROTO=TCP SPT=22 DPT=60206 WINDOW=2233 RES=0x00 ACK PSH URGP=0 
Mar 23 18:30:36 data-test03 kernel: IN= OUT=eth0 SRC=10.1.5.81 DST=10.1.5.237 LEN=136 TOS=0x10 PREC=0x00 TTL=64 ID=32154 DF PROTO=TCP SPT=22 DPT=60206 WINDOW=2233 RES=0x00 ACK PSH URGP=0 
Mar 23 18:30:36 data-test03 kernel: IN= OUT=eth0 SRC=10.1.5.81 DST=10.1.5.80 LEN=72 TOS=0x00 PREC=0x00 TTL=64 ID=39647 DF PROTO=TCP SPT=56998 DPT=2888 WINDOW=501 RES=0x00 ACK PSH URGP=0 
Mar 23 18:30:36 data-test03 kernel: IN= OUT=lo SRC=10.1.5.81 DST=10.1.5.81 LEN=228 TOS=0x00 PREC=0x00 TTL=64 ID=42802 DF PROTO=TCP SPT=36920 DPT=8025 WINDOW=265 RES=0x00 ACK PSH URGP=0 
Mar 23 18:30:36 data-test03 kernel: IN= OUT=lo SRC=10.1.5.81 DST=10.1.5.81 LEN=93 TOS=0x00 PREC=0x00 TTL=64 ID=14446 DF PROTO=TCP SPT=8025 DPT=36920 WINDOW=384 RES=0x00 ACK PSH URGP=0 
Mar 23 18:30:36 data-test03 kernel: IN= OUT=lo SRC=10.1.5.81 DST=10.1.5.81 LEN=52 TOS=0x00 PREC=0x00 TTL=64 ID=42803 DF PROTO=TCP SPT=36920 DPT=8025 WINDOW=265 RES=0x00 ACK URGP=0 
Mar 23 18:30:36 data-test03 kernel: IN= OUT=eth0 SRC=10.1.5.81 DST=10.1.5.237 LEN=136 TOS=0x10 PREC=0x00 TTL=64 ID=32155 DF PROTO=TCP SPT=22 DPT=60206 WINDOW=2233 RES=0x00 ACK PSH URGP=0 
Mar 23 18:30:36 data-test03 kernel: IN= OUT=eth0 SRC=10.1.5.81 DST=10.1.5.80 LEN=93 TOS=0x00 PREC=0x00 TTL=64 ID=13946 DF PROTO=TCP SPT=8025 DPT=38421 WINDOW=499 RES=0x00 ACK PSH URGP=0 
Mar 23 18:30:36 data-test03 kernel: IN= OUT=eth0 SRC=10.1.5.81 DST=10.1.5.79 LEN=93 TOS=0x00 PREC=0x00 TTL=64 ID=15862 DF PROTO=TCP SPT=8025 DPT=37458 WINDOW=499 RES=0x00 ACK PSH URGP=0 
Mar 23 18:30:36 data-test03 kernel: IN= OUT=eth0 SRC=10.1.5.81 DST=10.1.5.237 LEN=104 TOS=0x10 PREC=0x00 TTL=64 ID=32156 DF PROTO=TCP SPT=22 DPT=60206 WINDOW=2233 RES=0x00 ACK PSH URGP=0
```


为日志添加前缀
```
iptables -t filter -A OUTPUT -p tcp -j LOG --log-prefix will
```
再看log
```
[root@data-test03 hadoop-client]# tail -n 5 /var/log/messages
Mar 23 18:33:58 data-test03 kernel: willIN= OUT=eth0 SRC=10.1.5.81 DST=10.1.5.80 LEN=72 TOS=0x00 PREC=0x00 TTL=64 ID=39849 DF PROTO=TCP SPT=56998 DPT=2888 WINDOW=501 RES=0x00 ACK PSH URGP=0 
Mar 23 18:33:58 data-test03 kernel: willIN= OUT=lo SRC=10.1.5.81 DST=10.1.5.81 LEN=228 TOS=0x00 PREC=0x00 TTL=64 ID=43206 DF PROTO=TCP SPT=36920 DPT=8025 WINDOW=265 RES=0x00 ACK PSH URGP=0 
Mar 23 18:33:58 data-test03 kernel: willIN= OUT=lo SRC=10.1.5.81 DST=10.1.5.81 LEN=93 TOS=0x00 PREC=0x00 TTL=64 ID=14648 DF PROTO=TCP SPT=8025 DPT=36920 WINDOW=384 RES=0x00 ACK PSH URGP=0 
Mar 23 18:33:58 data-test03 kernel: willIN= OUT=lo SRC=10.1.5.81 DST=10.1.5.81 LEN=52 TOS=0x00 PREC=0x00 TTL=64 ID=43207 DF PROTO=TCP SPT=36920 DPT=8025 WINDOW=265 RES=0x00 ACK URGP=0 
Mar 23 18:33:58 data-test03 kernel: willIN= OUT=eth0 SRC=10.1.5.81 DST=10.1.5.237 LEN=136 TOS=0x10 PREC=0x00 TTL=64 ID=34907 DF PROTO=TCP SPT=22 DPT=60206 WINDOW=2233 RES=0x00 ACK PSH URGP=0 

```

命令规范：
```
iptables -t 表 操作 5个控制点之一 匹配规则 -j 动作 动作的选项
```


拒绝79对于8020端口的请求
```
iptables -t filter -A INPUT -p tcp --dport 8020 -s 10.1.5.79 -j REJECT
```
匹配包含：基本匹配、隐世匹配、等。


拒绝79所有请求, 第二条是给一个拒绝原因，顺序执行，如果一起都有的话，只执行第一条动作
```
iptables -t filter -A INPUT -s 10.1.5.79 -j REJECT
iptables -t filter -A INPUT -s 10.1.5.79 -j REJECT --reject-with tcp-reset
```
这样就连ping都ping不通了。




[网段知识补充](http://blog.csdn.net/czhphp/article/details/19123673)

参考：
- http://www.cnblogs.com/yi-meng/p/3213925.html
- http://www.dabu.info/iptables-based-tutorial-grammar-rules.html
- http://v.youku.com/v_show/id_XNzIxOTAxODky.html?from=s1.8-1-1.2&spm=a2h0k.8191407.0.0
