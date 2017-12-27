
title: iptables端口转发
date: 2017-11-06 17:25:53
tags: [youdaonote]
---


先放一张iptables的处理流程图

![](http://xstarcd.github.io/wiki/img/iptables_netfilter_chains.png)


```
# 在PREROUTING中添加转发规则:把从eth0网卡进来的1234端口的访问，定向到10.103.27.171:2224。
iptables -t nat -A PREROUTING -p tcp -i eth0 --dport 1234 -j DNAT --to 10.103.27.171:2224


# j后面是操作，masquerade的意思是伪装
iptables -t nat -A POSTROUTING -j MASQUERADE

# 保存并重启
service iptables save
service iptables restart
```

查看iptables的nat表相关配置
```
[root@node83 nginxserver_new]# iptables -t nat -L -vn
Chain PREROUTING (policy ACCEPT 12 packets, 650 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            60.1.1.1            tcp dpt:80 to:10.103.70.27:2224 
   16  2578 DNAT       tcp  --  *      *       0.0.0.0/0            10.103.27.171       tcp dpt:80 to:10.103.70.27:2224 
   10   520 DNAT       tcp  --  *      *       0.0.0.0/0            10.103.27.171       tcp dpt:1234 to:10.103.70.27:2224 
    0     0 DNAT       tcp  --  eth0   *       0.0.0.0/0            0.0.0.0/0           tcp dpt:1234 to:10.103.27.171:2224 

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
   69  5738 MASQUERADE  all  --  *      *       0.0.0.0/0            0.0.0.0/0           

Chain OUTPUT (policy ACCEPT 43 packets, 2640 bytes)
 pkts bytes target     prot opt in     out     source               destination

# 删除nat表第4个规则
[root@node83 nginxserver_new]# iptables -t nat -D PREROUTING 4
# 再看一下规则
[root@node83 nginxserver_new]# iptables -t nat -L -vn
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            60.1.1.1            tcp dpt:80 to:10.103.70.27:2224 
  130  8506 DNAT       tcp  --  *      *       0.0.0.0/0            10.103.27.171       tcp dpt:80 to:10.103.70.27:2224 
   10   520 DNAT       tcp  --  *      *       0.0.0.0/0            10.103.27.171       tcp dpt:1234 to:10.103.70.27:2224 

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
  394 24326 MASQUERADE  all  --  *      *       0.0.0.0/0            0.0.0.0/0           

Chain OUTPUT (policy ACCEPT 2 packets, 120 bytes)
 pkts bytes target     prot opt in     out     source               destination         
# 清空NAT表中PREROUTTING的规则
[root@node83 nginxserver_new]# iptables -t nat -F PREROUTING 
# 再看一下规则
[root@node83 nginxserver_new]# iptables -t nat -L -vn
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
  420 25806 MASQUERADE  all  --  *      *       0.0.0.0/0            0.0.0.0/0           

Chain OUTPUT (policy ACCEPT 2 packets, 120 bytes)
 pkts bytes target     prot opt in     out     source               destination
```



参考：
- http://xstarcd.github.io/wiki/Linux/iptables_forward_internetshare.html
