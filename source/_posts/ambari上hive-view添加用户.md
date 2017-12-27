
title: ambari上hive-view添加用户
date: 2016-12-05 17:09:45
tags: [youdaonote]
---

#### 问题描述
在ambari 2.2.0.0中添加完用户和用户组之后，需要为这个用户单独给定一次ambari dashboard的只读权限以后才能正常访问hive view，否则就会报错：
```
E090 RA040 I/O error while requesting Ambari [AmbariApiException]
```
还有一个问题是，hive view里的每个用户都需要在/user/hive下有自己的文件夹才行，并且把文件夹的owner设置生当前用户才行。否则报错
```
E090 HDFS020 Could not write file query.hql [HdfsApiException]
```

#### 以前总结
    a. 给新用户开通dashboard的只读权限和指定view的使用权限【解决ambari的IO问题AmbariApiException】;
    b. 到hdfs上创建/user/xxx目录，并将此目录的owner设置为xxx【解决hql写入问题HdfsApiException】
    c. 使用没有问题的话，就可以把这个xxx用户的dashboard权限去掉了。

---

#### 回想
上面给用户readonly权限感觉就跟跑堂的似的，那么它实际是执行了什么操作呢？我怀疑它会在hdfs上创建一个user，但是并不存在于Linux中。

问题来了：
1. hdfs和linux到底用的是不是同一套用户呢？还是说只是用的Linux的POSIX体系？
```
[root@datanode04 ~]# groups will
will : will hdfs
[root@datanode04 ~]# hdfs groups will
will :
[will@datanode04 root]$ hdfs dfs -ls /apps/hive
drwxrwxrw-   - hive hdfs          0 2016-05-30 10:47 /apps/hive/warehouse
[will@datanode04 root]$ hdfs dfs -ls /apps/hive/warehouse
ls: Permission denied: user=will, access=READ_EXECUTE, inode="/apps/hive/warehouse":hive:hdfs:drwxrwxrw-

```
看这样子，no！hdfs并没有使用Linux上已有的用户组和用户的关系。可以看到hdfs上的文件夹/apps/hive/warehouse对于all并没有x权限，也就是说不能进去。/apps/hive/warehouse这个文件夹是属于hive用户，hdfs组。而will不能访问，代表他在hdfs系统里并不属于hdfs组。

找了hdfs的官网命令，没有找到关于在hdfs里添加用户的命令。回头儿想了一下，有个失误的地方：我只在一个node上添加了will用户，其他node上并没有。参考http://dongxicheng.org/mapreduce/hadoop-permission-management/，hdfs就是使用的Linux用户和用户组，但是需要每个node分别创建。

```
scp -v /etc/passwd /etc/group datanode05.will.com:/etc
```
每个机器都scp过之后，再次查看hdfs里的效果：
```
[root@datanode04 ~]# hdfs groups will
will : will hdfs
```



好了。那么为了验证上面给新用户dashboard的readonly权限是为了添加用户，我们再去ambari里添加普通用户will。还是报错，老套路，添加用户对集群的read权限，发现以下几个借口：

##### 为用户组添加dashboard的只读权限
PUT: http://namenode01.will.com:8080/api/v1/clusters/datacenter/privileges
```json
[
    {
        "PrivilegeInfo": {
            "permission_name": "CLUSTER.READ",
            "principal_name": "analyst",
            "principal_type": "GROUP"
        }
    }
]
```
> curl  -u admin:admin -H 'X-Requested-By:ambari' -X PUT -d '[{"PrivilegeInfo":{"permission_name":"CLUSTER.READ","principal_name":"testgroup","principal_type":"GROUP"}}]' http://namenode01.will.com:8080/api/v1/clusters/datacenter/privileges

##### 为用户添加dashboard的只读权限
PUT：http://namenode01.will.com:8080/api/v1/clusters/datacenter/privileges
```json
[
    {
        "PrivilegeInfo": {
            "permission_name": "CLUSTER.READ",
            "principal_name": "will",
            "principal_type": "USER"
        }
    }
]
```

##### 收回用户/用户组dashboard的只读权限
PUT：http://namenode01.will.com:8080/api/v1/clusters/datacenter/privileges
```json
[]
```
更新成最新的权限组合就行了
【**这种就存在并发问题了，如果操作的是group会有相互影响，user就好些**】

##### 为用户/用户组添加hive view的只读权限
PUT：http://namenode01.will.com:8080/api/v1/views/HIVE/versions/1.0.0/instances/AUTO_HIVE_INSTANCE/privileges
```json
[
    {
        "PrivilegeInfo": {
            "permission_name": "VIEW.USE",
            "principal_name": "will",
            "principal_type": "USER"
        }
    }
]
```
> curl  -u admin:admin -H 'X-Requested-By:ambari' -X PUT -d '[{"PrivilegeInfo":{"permission_name":"VIEW.USE","principal_name":"testgroup","principal_type":"GROUP"}}]' http://namenode01.will.com:8080/api/v1/views/HIVE/versions/1.0.0/instances/AUTO_HIVE_INSTANCE/privileges

* 以上接口移除权限的时候同样适用PUT方法，只是json为空：[]。

##### ambari删除用户
DELETE: http://namenode01.will.com:8080/api/v1/users/will
```json
{"id":"will"}
```

##### ambari添加用户
POST: http://namenode01.will.com:8080/api/v1/users
```json
{
    "Users/user_name": "will",
    "Users/password": "1234qwer",
    "Users/active": true,
    "Users/admin": false
}
```
> curl -u admin:admin -H 'X-Requested-By:ambari' -X POST -d '{"Users/user_name":"testuser","Users/password":"1234qwer","Users/active":true,"Users/admin":false}' http://namenode01.will.com:8080/api/v1/users


##### ambari添加用户组
POST: http://namenode01.will.com:8080/api/v1/groups
```json
{"Groups/group_name":"tt"}
```
> curl  -u admin:admin -H 'X-Requested-By:ambari' -X POST -d '{"Groups/group_name":"testgroup"}' http://namenode01.will.com:8080/api/v1/groups

##### ambari删除用户组
DELETE: http://namenode01.will.com:8080/api/v1/groups/tt
啥都不用传


##### ambari为用户指定用户组
POST: http://namenode01.will.com:8080/api/v1/groups/**analyst**/members/**will**
> curl  -u admin:admin -H 'X-Requested-By:ambari' -X POST http://namenode01.will.com:8080/api/v1/groups/testgroup/members/testuser

这个参数在url里

---


综上总结过程如下：
```bash
#! /bin/bash

user=$1
group=$2
pri_grp=$3

# 1.在namenode上执行添加用户以及指定用户组的Linux命令
useradd -G $group $user
groups $user

# 2.检查hdfs上的用户组
hdfs groups $user

# 3.ambari上添加用户
curl -u admin:admin -H 'X-Requested-By:ambari' -X POST -d '{"Users/user_name":"$user","Users/password":"1234qwer","Users/active":true,"Users/admin":false}' http://namenode01.will.com:8080/api/v1/users

# 4.ambari 调整用户到用户组
curl  -u admin:admin -H 'X-Requested-By:ambari' -X POST http://namenode01.will.com:8080/api/v1/groups/$group/members/$user

# 为用户添加dashboard的只读权限
curl  -u admin:admin -H 'X-Requested-By:ambari' -X PUT -d '[{"PrivilegeInfo":{"permission_name":"CLUSTER.READ","principal_name":"$user","principal_type":"USER"}}]' http://namenode01.will.com:8080/api/v1/clusters/datacenter/privileges

# 模拟用户登录，访问一下hive view登录，这个过程会自动创建/user/$user文件夹
# 用chrome找了半天，么有找到登录的API，看看源码吧

# 模拟登录用户，访问hive view

# 删除普通用户对dashboard的所有权限
curl  -u admin:admin -H 'X-Requested-By:ambari' -X PUT -d '[]' http://namenode01.will.com:8080/api/v1/clusters/datacenter/privileges

# 6. 检查一下用户文件夹，以此作为终点
hdfs dfs -ls /user/$user
```

#### 思考
可以修改一下ambari添加用户和用户组的地方，后台自动针对namenode的Linux系统进行相应操作，就不用单独再去执行命令行了。这样ambari、hdfs、Linux的用户以及用户组就会是一致的，后面的依赖前面的。

hive view加载过程中会请求一个URL：http://namenode01.will.com:8080/api/v1/clusters/datacenter/hosts?fields=Hosts%2Fpublic_host_name%2Chost_components%2FHostRoles%2Fcomponent_name

这个URL需要对集群有可读权限，奇怪的是使用chrome看下面的网络请求里压根儿没有这个。它是内嵌在某个接口调用里的。

#### 延迟问题：
```shell
[root@namenode01 will]# useradd -G hdfs will1
[root@namenode01 will]# groups will1
will1 : will1 hdfs
[root@namenode01 will]# hdfs groups will1
will1 : will1 hdfs
[root@namenode01 will]# userdel will1
[root@namenode01 will]# groups will1
groups: will1: No such user
[root@namenode01 will]# hdfs groups will1
will1 : will1 hdfs
[root@namenode01 will]# cat /etc/group | grep will
hdfs:x:511:hdfs,will
will:x:1003:
[root@namenode01 will]# hdfs groups will1
will1 : will1 hdfs
[root@namenode01 will]# hdfs groups will2
will2 :
[root@namenode01 will]# useradd -G sqoop will1
[root@namenode01 will]# groups will1
will1 : will1 sqoop
[root@namenode01 will]# hdfs groups will1
will1 : will1 hdfs  
[root@namenode01 will]# su will1
[will1@namenode01 will]$ hdfs dfs -ls /apps/hive/warehouse
Found 2 items
drwxrwxrw-   - hive hdfs          0 2016-05-30 10:40 /apps/hive/warehouse/batting
drwxrwxrw-   - hive hdfs          0 2016-05-30 10:47 /apps/hive/warehouse/batting2
[will1@namenode01 will]$ hdfs groups will1
will1 : will1 hdfs
[will1@namenode01 will]$ groups will1
will1 : will1 sqoop
```
上面看到，在namenode上添加用户后就可以生效了。但是假设我分配组分配错了，重新修改用户组后，hdfs并没有跟着修改，will1用户还是可以访问到hdfs组权限的文件。过了一段时间后，才可以了。目测盯了一下，差不多4分钟左右才生效。

可是如果Linux删除了这个用户，hdfs能够也跟着删除么？
```
[root@namenode01 will]# hdfs groups will1
will1 : will1 hdfs
[root@namenode01 will]# grep will /etc/passwd
[root@namenode01 will]# grep will /etc/group
```
刚删除同样没有生效，过一会儿再看。
```
[root@namenode01 will]# hdfs groups will1
will1 :
```
也是会删除的。但是如果**权限给错就不能立即收回**，这是个问题。

参考：https://issues.apache.org/jira/browse/AMBARI-12732
https://docs.hortonworks.com/HDPDocuments/Ambari-2.2.0.0/bk_ambari_views_guide/content/troubleshooting.html
