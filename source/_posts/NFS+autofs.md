
title: NFS+autofs
date: 2016-12-05 17:05:10
tags: [youdaonote]
---

server端
---
安装服务
```
yum -y install nfs-utils rpcbind
```

设置开机启动，并启动服务
```
chkconfig nfs on
chkconfig rpcbind on

/etc/init.d/rpcbind restart
/etc/init.d/nfs restart 
```

编辑nfs主配置文件/etc/exports，添加挂载目录
```
/server/will *(ro,no_root_squash,sync)
```
这里挂载了/server/will目录，任何机器远程挂载之后都可以访问此目录，只有普通的只读权限。

重新加载主配置文件
```
[root@namenode01 will]# exportfs -arv
exporting *:/server/will
```


client端
---
安装服务
```
yum install nfs-utils autofs -y
```

启动服务
```
chkconfig nfs on
chkconfig rpcbind on

/etc/init.d/rpcbind restart
/etc/init.d/nfs restart
```

查看是否能成功连接NFS服务
```
[root@schedule ~]# showmount -e 10.0.1.72
Export list for 10.0.1.72:
/server/will *
```
这里可以看到可以成功访问。

配置autofs，在/etc/auto.master最后加入：
```
/server/will /etc/auto.nfs
```

编辑/etc/auto.nfs
```
script   -fstype=nfs                      10.0.1.72:/server/will
```

重启nfs
```
[root@schedule ~]# service autofs restart
Loading autofs4:                                           [  OK  ]
Starting automount:                                        [  OK  ]

```

测试
```
[root@datanode01 ~]# df
Filesystem           1K-blocks      Used Available Use% Mounted on
/dev/mapper/vg_template-lv_root
                      16070076   6401092   8852652  42% /
tmpfs                  8167372         0   8167372   0% /dev/shm
/dev/sda1               495844     31581    438663   7% /boot
/dev/mapper/luoji-mail
                     707004512 423592108 247504948  64% /server
10.0.1.72:/server/will
                      51606528   1049600  47935488   3% /server/will/script
```
