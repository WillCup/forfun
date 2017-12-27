
title: Fabric自动化运维脚本
date: 2017-02-27 10:50:36
tags: [youdaonote]
---



综合考察了salt和Fabric两个东西，最后使用灵活的Fabric来自动化运维HDFS集群的日志文件维护。

使用前提：
- node之间的ssh互信
- 本机安装fabric

下面是fabric.py的脚本：
```py
from fabric.api import *




def get_host():
    hosts = list()
    with open('/etc/hosts', 'r') as hf:
        for line in hf.readlines():
            ss = line.split()
            if 'node' in ss[1]:
		hosts.append(ss[0])
    hosts.remove('10.0.1.111')
    hosts.remove('10.0.1.112')
    return hosts

env.hosts = get_host()

def hello():
    print("Hello world!")

def will():
    run('ifconfig')


def clean_log():
#    run("ls /var/log/hadoop/hdfs/ | awk '{if (index($0, '2017') > 0) print $0}' | exec rm -rf {} \ ; ")
    with settings(warn_only=True):    
	local('echo `date` >> ~/clean_log.log')
	if run('find /var/log/hadoop/hdfs/ -mtime +3 -name "*log-2016*" -exec rm -rvf {} \;').failed:
            print('failed for %s ' % env.host)
        if run('find /var/log/hadoop/hdfs/ -mtime +3 -name "*will.com.out.[3,4,5,6,7,8]" -exec rm -rvf {} \;').failed:
	    print('failed for %s ' % env.host)
        if run('find /var/log/hadoop/hdfs/ -mtime +3 -name "*will.com.log.[3,4,5,6,7,8]" -exec rm -rvf {} \;').failed:
            print('failed for %s ' % env.host)

```

上面例子中，是读取了当前/etc/hosts文件中的所有host的IP，之后逐一执行远程命令，删除过期的日志。


