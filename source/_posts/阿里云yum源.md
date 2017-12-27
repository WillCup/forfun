
title: 阿里云yum源
date: 2017-09-14 15:34:14
tags: [youdaonote]
---

[root@centos7-base-ok]# cat /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0



参考： http://www.cnblogs.com/liangDream/p/7358847.html
