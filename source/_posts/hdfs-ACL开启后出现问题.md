
title: hdfs-ACL开启后出现问题
date: 2017-04-01 16:21:15
tags: [youdaonote]
---

已经用测试环境重现了此问题。

1. 启用namenode的acl，在hdfs-site.xml中添加

```
<property>
    <name>dfs.namenode.acls.enabled</name>
    <value>true</value>
</property>

```

2. 使用root用户创建初始文件夹, 并给以同生产环境的ACL设置：
```
root@will-vm:/opt/hadoop-2.7.1# ./bin/hdfs dfs -mkdir -p /tmp/ods_tinyv_outer.db/channel=wb
root@will-vm:/opt/hadoop-2.7.1# ./bin/hdfs dfs -getfacl /tmp/ods_tinyv_outer.db/channel=wb
# file: /tmp/ods_tinyv_outer.db/channel=wb
# owner: root
# group: supergroup
user::rwx
group::r-x
other::r-x

root@will-vm:/opt/hadoop-2.7.1# ./bin/hdfs dfs -setfacl -R -m default:user::rwx /tmp/ods_tinyv_outer.db
root@will-vm:/opt/hadoop-2.7.1# ./bin/hdfs dfs -setfacl -R -m default:group:wil:rwx /tmp/ods_tinyv_outer.db

root@will-vm:/opt/hadoop-2.7.1# ./bin/hdfs dfs -getfacl /tmp/ods_tinyv_outer.db/channel=wb
# file: /tmp/ods_tinyv_outer.db/channel=wb
# owner: root
# group: supergroup
user::rwx
group::r-x
other::r-x
default:user::rwx
default:group::r-x
default:group:wil:rwx
default:mask::rwx
default:other::r-x
root@will-vm:/opt/hadoop-2.7.1# ./bin/hdfs dfs -chown -R wil:wil /tmp/ods_tinyv_outer.db/channel=wb
root@will-vm:/opt/hadoop-2.7.1# ./bin/hdfs dfs -getfacl /tmp/ods_tinyv_outer.db/channel=wb
# file: /tmp/ods_tinyv_outer.db/channel=wb
# owner: wil
# group: wil
user::rwx
group::r-x
other::r-x
default:user::rwx
default:group::r-x
default:group:wil:rwx
default:mask::rwx
default:other::r-x
root@will-vm:/opt/hadoop-2.7.1# ./bin/hdfs dfs -chmod 770 /tmp/ods_tinyv_outer.db/channel=wb
root@will-vm:/opt/hadoop-2.7.1# ./bin/hdfs dfs -getfacl /tmp/ods_tinyv_outer.db/channel=wb
# file: /tmp/ods_tinyv_outer.db/channel=wb
# owner: wil
# group: wil
user::rwx
group::rwx
other::---
default:user::rwx
default:group::r-x
default:group:wil:rwx
default:mask::rwx
default:other::r-x


```

3. 重现问题
```
wil@will-vm:/opt/hadoop-2.7.1$ ./bin/hdfs dfs -rmr /tmp/ods_tinyv_outer.db/channel=wb
rmr: DEPRECATED: Please use 'rm -r' instead.
17/04/02 23:01:41 INFO fs.TrashPolicyDefault: Namenode trash configuration: Deletion interval = 0 minutes, Emptier interval = 0 minutes.
rmr: Permission denied: user=wil, access=WRITE, inode="/tmp/ods_tinyv_outer.db/channel=wb":root:supergroup:drwxr-xr-x

```

这个文件明明是属于wil用户wil用户组的，并且从acl来看是有权限的。然而实际执行的时候，却提示没权，而且列出来的权限竟然是`:root:supergroup:drwxr-xr-x`，这个应该是刚刚创建时的权限。





4. 解决
```
root@will-vm:/opt/hadoop-2.7.1# ./bin/hdfs dfs -chown wil:wil /tmp/ods_tinyv_outer.db
```

