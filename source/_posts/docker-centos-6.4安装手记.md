
title: docker-centos-6.4安装手记
date: 2016-12-05 17:04:54
tags: [youdaonote]
---



version Base not defined in file libdevmapper.so.1.02 with link time reference
---
yum install device-mapper-event-libs -y

参考： http://qicheng0211.blog.51cto.com/3958621/1582909

can't initialize iptables table `nat'
---
编译内核的时候没有选中iptable_nat module相关的组件。
重新到内核文件夹中make 

GCC 4.8 or higher required
---
参考：https://gist.github.com/stephenturner/e3bc5cfacc2dc67eca8b

参考：
- http://blog.csdn.net/scaleqiao/article/details/46633011
- 
