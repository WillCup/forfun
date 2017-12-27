
title: centos7-搭建DNS服务器
date: 2017-07-12 11:29:47
tags: [youdaonote]
---


主服务器
---

#### 安装bind9
```
yum install bind bind-utils -y
```

#### 编辑/etc/named.conf
```
options {
    listen-on port 53 { any; };
	listen-on-v6 port 53 { ::1; };
	directory 	"/var/named";
	dump-file 	"/var/named/data/cache_dump.db";
	statistics-file "/var/named/data/named_stats.txt";
	memstatistics-file "/var/named/data/named_mem_stats.txt";
	allow-query     { any; };
	allow-transfer{ localhost; 10.2.19.113; };

......
...

zone "will.com" IN {
type master;
file "forward.unixmen";
allow-update { none; };
};

zone "19.2.10.in-addr.arpa" IN {
type master;
file "reverse.unixmen";
allow-update { none; };
};



```

#### 创建上面我们配置的zone文件
vi /var/named/forward.unixmen
```
$TTL 86400
@   IN  SOA     etl02.will.com. root.will.com. (
        2017071001  ;Serial
        3600        ;Refresh
        1800        ;Retry
        604800      ;Expire
        86400       ;Minimum TTL
)
@       IN  NS          etl02.will.com.
@       IN  NS          etl03.will.com.
@       IN  NS          schedule.will.com.
@       IN  A           10.2.19.112
@       IN  A           10.2.19.113
@       IN  A           10.2.19.62
etl02       IN  A   10.2.19.112
etl03    IN  A   10.2.19.113
schedule    IN  A   10.2.19.62

```

检查配置
```
[root@etl02 ~]# named-checkzone will.com /var/named/forward.unixmen
zone will.com/IN: loaded serial 2017071001
OK
```

vi /var/named/reverse.unixmen
```
$TTL 86400
@   IN  SOA     etl02.will.com. root.will.com. (
        2017071001  ;Serial
        3600        ;Refresh
        1800        ;Retry
        604800      ;Expire
        86400       ;Minimum TTL
)
@       IN  NS          etl02.will.com.
@       IN  NS          etl03.will.com.
@       IN  NS          schedule.will.com.
@       IN  PTR         will.com.
etl02       IN  A   10.2.19.112
etl03    IN  A   10.2.19.113
schedule    IN  A   10.2.19.62
62     IN  PTR         schedule.will.com.
112     IN  PTR         etl02.will.com.
113     IN  PTR         etl03.will.com.
```

同样检查一下
```
[root@etl02 ~]# named-checkzone will.com /var/named/reverse.unixmen 
zone will.com/IN: loaded serial 2017071001
OK

```


启动DNS服务
---
```
systemctl enable named
systemctl start named
```


测试
---
1. 编辑/etc/resolv.conf文件, 修改nameserver为etl02.will.com
2. 编辑网卡配置文件/etc/sysconfig/network-scripts/ifcfg-eth0, 修改DNS为我们的DNS服务器地址10.2.19.112
3. 重启网络服务：systemctl restart network


```
[root@etl02 ~]# dig etl02.will.com

; <<>> DiG 9.9.4-RedHat-9.9.4-50.el7_3.1 <<>> etl02.will.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 48492
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 3, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;etl02.will.com.		IN	A

;; ANSWER SECTION:
etl02.will.com.	86400	IN	A	10.2.19.112

;; AUTHORITY SECTION:
will.com.		86400	IN	NS	etl02.will.com.
will.com.		86400	IN	NS	schedule.will.com.
will.com.		86400	IN	NS	etl03.will.com.

;; ADDITIONAL SECTION:
etl03.will.com.	86400	IN	A	10.2.19.113
schedule.will.com.	86400	IN	A	10.2.19.62

;; Query time: 0 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Fri Jul 14 11:52:31 EDT 2017
;; MSG SIZE  rcvd: 150

```
参考：https://www.unixmen.com/setting-dns-server-centos-7/
