
title: saltstack自动化运维实施（一）
date: 2016-12-30 18:09:41
tags: [youdaonote]
---


安装
---
```
pip install salt
```

配置
---
###### master
配置文件/etc/salt/master, 打开注释
```
autosign_file: /etc/salt/autosign.conf
```
更多参考： https://docs.saltstack.com/en/latest/ref/configuration/master.html
###### minion
一个是/etc/salt/minion_id, 一般可以取为hostname

还有/etc/salt/minion, 指定master的host
```
master: 10.0.1.110
```
更多参考： https://docs.saltstack.com/en/latest/ref/configuration/minion.html

启动
---
###### master
salt-master restart
###### minion
salt-minion restart


认证
---
###### 手动认证
默认是没有开启自动认证的，需要人工手动判断哪些id是允许的。
```
[root@datanode21 ~]# salt-key -L    # 列出所有id
Accepted Keys:
datanode19
datanode20
datanode21
Denied Keys:
Unaccepted Keys:
Rejected Keys:
[root@datanode21 ~]# salt-key -A # 通过所有认证
[root@datanode21 ~]# salt-key -d datanode21  # 删除指定id
The following keys are going to be deleteed:
Accepted Keys:
datanode21
Proceed? [N/y] y
Key for minion datanode21 deleteed.
```

官网有说把master.pub放到minion里，但是我使用的时候并没有发现什么卵用~~~

###### 自动认证
我弄了一种并不是很安全，但是相对比较安全的方式，使用hostname来自动认证.
编辑/etc/salt/master文件, 打开注释
```
autosign_file: /etc/salt/autosign.conf
```

之后编辑/etc/salt/autosign.conf
```
datanode??.will.com
```
这里是正则匹配id的，如果minion的id能满足这里的正则表达式，就可以自动通过认证。

因为修改了/etc/salt/master文件，所以需要重启一下，如果只修改匹配规则文件是不需要重启的。

参考：http://www.it610.com/article/3325439.htm


测试
---
在master的机器上执行
```
[root@datanode21 ~]# salt '*' test.ping
datanode21.will.com:
    True
datanode19.will.com:
    True
datanode20.will.com:
    True
```


分发脚本
---
先安装[**Fabric**](http://fabric-chs.readthedocs.io/zh_CN/chs/tutorial.html).


编辑fabfile.py：
```py
#!/usr/bin/env python
# encoding: utf-8

#from fabric.api import local,cd,run,env,put
from fabric.api import *

env.hosts=['172.16.4.74']
env.password = 'Yinker.com'


@parallel
def salt_install():
#    run('/etc/init.d/iptables stop')  #远程操作用run
#    run('mkdir /server')
    run('yum -y install openssh-clients')  #远程操作用run
    run('yum -y install https://repo.saltstack.com/yum/redhat/salt-repo-latest-1.el6.noarch.rpm')  #远程操作用run
    run('yum -y install python-devel')  #远程操作用run
    run('yum -y install salt-minion')  #远程操作用run
    run('test -f /etc/salt/minion && rm -rf /etc/salt/minion')  #远程操作用run
    put('/root/qian/minion','/etc/salt/minion')
    run('echo `hostname` >/etc/salt/minion_id')
    put('/root/qian/selinux','/etc/sysconfig/selinux')
    run('setenforce 0')
    run('/etc/init.d/salt-minion restart')
```

调用
```
/usr/local/bin/fab -f fabfile.py salt_install
```
