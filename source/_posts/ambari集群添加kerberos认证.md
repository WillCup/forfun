
title: ambari集群添加kerberos认证
date: 2017-03-22 17:07:41
tags: [youdaonote]
---

host准备
ip | 名字 | 备注
---|---|---
10.1.5.79 | data-test01 | ambari-server/KDC
10.1.5.80 | data-test02 |
10.1.5.81 | data-test03 |



1.KDC配置
=

##### a. 安装
在test01上安装KDC:
```
yum install -y krb5-server krb5-libs krb5-auth-dialog krb5-workstation
```

其他机器安装客户端
```
yum install krb5-devel krb5-workstation -y
```

##### b.编辑配置文件
/etc/krb5.conf
```
[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log

[libdefaults]
 default_realm = WILLCLUSTER.COM
 dns_lookup_realm = false
 dns_lookup_kdc = false
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true
 udp_preference_limit = 1

[realms]
 WILLCLUSTER.COM = {
  kdc = kerberos.willcluster.com
  admin_server = kerberos.willcluster.com
 }

[domain_realm]
 .willcluster.com = WILLCLUSTER.COM
 willcluster.com = WILLCLUSTER.COM
```
/var/kerberos/krb5kdc/kdc.conf
```
[kdcdefaults]
 kdc_ports = 88
 kdc_tcp_ports = 88

[realms]
 WILLCLUSTER.COM = {
  #master_key_type = aes256-cts
  acl_file = /var/kerberos/krb5kdc/kadm5.acl
  dict_file = /usr/share/dict/words
  admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
  supported_enctypes = aes256-cts:normal aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal
 }
```

##### c.分发/etc/krb5.conf到各个节点
```
scp /etc/krb5.conf data-test02:/etc/
scp /etc/krb5.conf data-test03:/etc/
```

##### d. test01上启动服务
```
chkconfig --level 35 krb5kdc on
chkconfig --level 35 kadmin on
service krb5kdc start
service kadmin start
```

2.数据库
=

###### 创建kerberos数据库
```
kdb5_util create -r WILLCLUSTER.COM -s
```

###### 在test01上创建超级用户
```
kadmin.local -q "addprinc root/admin"
```


#####  创建ambari各个服务用户与相应权限

HDP里的每个服务都必须有自己的principal。为避免每次都输入密码，我们使用keytab文件作为认证，keytab是从kerberos数据库中抽取出来的。

下面使用超级用户来创建对应service的principal
```
kadmin.local -q "addprinc -randkey yarn/data-test01@WILLCLUSTER.COM"
kadmin.local -q "addprinc -randkey yarn/data-test02@WILLCLUSTER.COM"
kadmin.local -q "addprinc -randkey yarn/data-test03@WILLCLUSTER.COM"

kadmin.local -q "addprinc -randkey zookeeper/data-test01@WILLCLUSTER.COM"
kadmin.local -q "addprinc -randkey zookeeper/data-test02@WILLCLUSTER.COM"
kadmin.local -q "addprinc -randkey zookeeper/data-test03@WILLCLUSTER.COM"

kadmin.local -q "addprinc -randkey hdfs/data-test01@WILLCLUSTER.COM"
kadmin.local -q "addprinc -randkey hdfs/data-test02@WILLCLUSTER.COM"
kadmin.local -q "addprinc -randkey hdfs/data-test03@WILLCLUSTER.COM"

kadmin.local -q "addprinc -randkey nn/data-test01@WILLCLUSTER.COM"
kadmin.local -q "addprinc -randkey nn/data-test02@WILLCLUSTER.COM"

...
```
**Tips**：**为什么要每个node都要弄一份呢？**

两个原因：
 - 如果一个node的kerberos认证通过，并不能自动扩散到所有node。kerberos认证是以node为单位的
 - 如果多个node共用同一个认证的话，万一这些node的kerberos认证同时发出去，也就是带有同样的时间戳，那么这些请求会被认为是重复的请求，后面来的会被拒绝
 

然后抽取keytab文件到本地目录
```
kadmin.local -q "xst  -k yarn.keytab  yarn/data-test01@WILLCLUSTER.COM"
kadmin.local -q "xst  -k yarn.keytab  yarn/data-test02@WILLCLUSTER.COM"
kadmin.local -q "xst  -k yarn.keytab  yarn/data-test03@WILLCLUSTER.COM"

kadmin.local -q "xst  -k zookeeper.keytab zookeeper/data-test01@WILLCLUSTER.COM"
kadmin.local -q "xst  -k zookeeper.keytab zookeeper/data-test02@WILLCLUSTER.COM"
kadmin.local -q "xst  -k zookeeper.keytab zookeeper/data-test03@WILLCLUSTER.COM"

...
```

再把keytab文件分发到每个服务对应的每个节点
```
scp yarn.keytab zookeeper.keytab data-test02:/usr/hdp/current/hadoop-client/conf
scp yarn.keytab zookeeper.keytab data-test03:/usr/hdp/current/hadoop-client/conf
```

或者也可以先将所有的keytab合并到一个里面，方便copy，但是这也带来了各个node权限控制的问题。进入ktutil，读取原有的keytab，写入到统一的一个keytab文件。
```
ktutil: rkt yarn.keytab
ktutil: rkt zookeeper.keytab
....
ktutil: wkt all_in_one.keytab
```
写完之后查看一下
```
klist -ket  all_in_one.keytab
```


3.HDP相关
=

给HDP配置kerberos包含两个部分：
- 创建unix系统里对应用户名的service principal。因为hadoop默认是使用的ShellBasedUnixGroupsMapping，它继承自GroupMappingServiceProvider接口，使用namenode的linux文件与用户权限系统。
- 在对应service的配置文件中添加kerberos相关配置。


#### 3.1 创建unix系统里对应用户名的service principal

HDP使用一个基于规则的系统来创建service principal和对应unix用户的映射。这些rule配置在core-site.xml的hadoop.security.auth_to_local属性。默认是把所有的principal对应于这个principal的第一个组件。

创建一个分层的rule来提供多种复杂的情形。每个rule由三个部分组成：
-  base
 以对应的service principal名字的的组件个数开头，跟一个冒号，然后是用户名的pattern。 有点儿类似awk的处理，$0代表realm，$1代表第一个组件，以此类推
例如：
```
[1:$1@$0] 把 myusername@APACHE.ORG 对应到 myusername@APACHE.ORG 
[2:$1] 把 myusername/admin@APACHE.ORG 对应到 myusername 
[2:$1%$2] 把 myusername/admin@APACHE.ORG 对应到 “myusername%admin
```
-  filter
一个正则表达式, 过滤rule。
-  substitution【置换】
把一个正则转化为一个固定字符串,类似sed，例如：
```
s/@ACME\.COM// 删除第一个 @ACME.DOMAIN
s/@[A-Z]*\.COM// 删除第一个后面跟着COM的@. 
s/X/Y/g 替换所有的 X成 Y
```

##### 3.1.1 例子
- 如果你的默认realm是APACHE.COM,但是你想获取所有的ACME.COM的principal，下面的rule可以做到
```
RULE:[1:$1@$0](.@ACME.COM)s/@.//
DEFAULT
```
- 要把名字对应于第二个组件，使用下面规则：
```
RULE:[1:$1@$0](.@ACME.COM)s/@.//
RULE:[2:$1@$0](.@ACME.COM)s/@.// DEFAULT
```
- 把所有带有admin的APACHE.ORG对应于admin
```
RULE[2:$1%$2@$0](.%admin@APACHE.ORG)s/./admin/
DEFAULT
```
- 把所有用户名转为小写
```
RULE:[1:$1]/L
RULE:[2:$1]/L
RULE:[2:$1;$2](^.*;admin$)s/;admin$///L
RULE:[2:$1;$2](^.*;guest$)s/;guest$//g/L
```
基于上面的rule，输入与对应输出如下：
```
"JOE@FOO.COM" to "joe"
"Joe/root@FOO.COM" to "joe"
"Joe/admin@FOO.COM" to "joe"
"Joe/guestguest@FOO.COM" to "joe"
```

#### 3.2 修改配置文件

首先，在hadoop-env.sh里配置JSVC_HOME
```
export JSVC_HOME=/usr/libexec/bigtop-utils
```

###### core-site.xml

Property Name | Property Value | Description
---|---|---
hadoop.security.authentication | kerberos
hadoop.rpc.protection | authentication; integrity; privacy | 可选
hadoop.security.authorization | true
hadoop.security.auth_to_local | 上面提到的rule

```
<property> 
     <name>hadoop.security.authentication</name> 
     <value>kerberos</value> 
     <description> Set the authentication for the cluster. 
     Valid values are: simple or kerberos.</description> 
</property> 
 
<property> 
     <name>hadoop.security.authorization</name> 
     <value>true</value> 
     <description>Enable authorization for different protocols.</description> 
</property> 
 
<property>
    <name>hadoop.security.auth_to_local</name> 
    <value> 
    RULE:[2:$1@$0]([jt]t@.*EXAMPLE.COM)s/.*/mapred/ 
    RULE:[2:$1@$0]([nd]n@.*EXAMPLE.COM)s/.*/hdfs/ 
    RULE:[2:$1@$0](hm@.*EXAMPLE.COM)s/.*/hbase/ 
    RULE:[2:$1@$0](rs@.*EXAMPLE.COM)s/.*/hbase/ 
    DEFAULT
    </value> 
    <description>The mapping from kerberos principal names
    to local OS user names.</description>
</property>
```

下面是有关hue和knox的配置
```
<property> 
     <name>hadoop.security.authentication</name> 
     <value>kerberos</value> 
     <description>Set the authentication for the cluster. 
     Valid values are: simple or kerberos.</description> 
</property> 
 
<property> 
     <name>hadoop.security.authorization</name> 
     <value>true</value> 
     <description>Enable authorization for different protocols. 
     </description> 
</property> 
 
<property>
     <name>hadoop.security.auth_to_local</name> 
     <value> 
     RULE:[2:$1@$0]([jt]t@.*EXAMPLE.COM)s/.*/mapred/ 
     RULE:[2:$1@$0]([nd]n@.*EXAMPLE.COM)s/.*/hdfs/ 
     RULE:[2:$1@$0](hm@.*EXAMPLE.COM)s/.*/hbase/ 
     RULE:[2:$1@$0](rs@.*EXAMPLE.COM)s/.*/hbase/ 
     DEFAULT
     </value> 
     <description>The mapping from kerberos principal names
     to local OS user names.</description>
</property>
 
<property>
     <name>hadoop.proxyuser.knox.groups</name>
     <value>users</value>
</property>
 
<property>
     <name>hadoop.proxyuser.knox.hosts</name>
     <value>Knox.EXAMPLE.COM</value>
</property> 
```
HTTP Cookie 持久化
默认HTTP 认证过程中，cookie是不会保留的。
```
<property>
   <name>hadoop.http.authentication.cookie.persistent</name>
   <value>true</value> 
</property>
```


###### hdfs-site.xml
```
<property> 
     <name>dfs.permissions</name> 
     <value>true</value> 
     <description> If "true", enable permission checking in
     HDFS. If "false", permission checking is turned
     off, but all other behavior is
     unchanged. Switching from one parameter value to the other does
     not change the mode, owner or group of files or
     directories. </description> 
</property> 
 
<property> 
     <name>dfs.permissions.supergroup</name> 
     <value>hdfs</value> 
     <description>The name of the group of
     super-users.</description> 
</property> 
 
<property> 
     <name>dfs.block.access.token.enable</name> 
     <value>true</value> 
     <description> If "true", access tokens are used as capabilities
     for accessing datanodes. If "false", no access tokens are checked on
     accessing datanodes. </description> 
</property> 
 
<property> 
     <name>dfs.namenode.kerberos.principal</name> 
     <value>nn/_HOST@EXAMPLE.COM</value> 
     <description> Kerberos principal name for the
     NameNode </description> 
</property> 
 
<property> 
     <name>dfs.secondary.namenode.kerberos.principal</name> 
     <value>nn/_HOST@EXAMPLE.COM</value> 
     <description>Kerberos principal name for the secondary NameNode. 
     </description> 
</property> 
 
<property> 
     <name>dfs.web.authentication.kerberos.principal</name> 
     <value>HTTP/_HOST@EXAMPLE.COM</value> 
     <description> The HTTP Kerberos principal used by Hadoop-Auth in the HTTP endpoint.
     The HTTP Kerberos principal MUST start with 'HTTP/' per Kerberos HTTP
     SPNEGO specification. 
     </description> 
</property> 
 
<property> 
     <name>dfs.web.authentication.kerberos.keytab</name> 
     <value>/etc/security/keytabs/spnego.service.keytab</value> 
     <description>The Kerberos keytab file with the credentials for the HTTP
     Kerberos principal used by Hadoop-Auth in the HTTP endpoint. 
     </description> 
</property> 
 
<property> 
     <name>dfs.datanode.kerberos.principal</name> 
     <value>dn/_HOST@EXAMPLE.COM</value> 
     <description> 
     The Kerberos principal that the DataNode runs as. "_HOST" is replaced by the real
     host name. 
     </description> 
</property> 
 
<property> 
     <name>dfs.namenode.keytab.file</name> 
     <value>/etc/security/keytabs/nn.service.keytab</value> 
     <description> 
     Combined keytab file containing the namenode service and host
     principals. 
     </description> 
</property> 
 
<property> 
     <name>dfs.secondary.namenode.keytab.file</name> 
     <value>/etc/security/keytabs/nn.service.keytab</value> 
     <description> 
     Combined keytab file containing the namenode service and host
     principals. 
     </description> 
</property> 
 
<property> 
     <name>dfs.datanode.keytab.file</name> 
     <value>/etc/security/keytabs/dn.service.keytab</value> 
     <description> 
     The filename of the keytab file for the DataNode. 
     </description> 
</property> 
 
<property> 
     <name>dfs.access.time.precision</name> 
     <value>0</value> 
     <description>The access time for HDFS file is precise upto this
     value.The default value is 1 hour. Setting a value of 0
     disables access times for HDFS. 
     </description> 
</property> 
<property> 
     <name>dfs.namenode.kerberos.internal.spnego.principal</name> 
     <value>${dfs.web.authentication.kerberos.principal}</value> 
</property> 
 
<property> 
     <name>dfs.secondary.namenode.kerberos.internal.spnego.principal</name> 
     <value>${dfs.web.authentication.kerberos.principal}</value> 
</property> 
```

另外，还需加上如下配置：

```
export HADOOP_SECURE_DN_USER=hdfs
export HADOOP_SECURE_DN_PID_DIR=/grid/0/var/run/hadoop/$HADOOP_SECURE_DN_USER

```

###### yarn-site.xml

```
<property>
     <name>yarn.resourcemanager.principal</name>
     <value>yarn/localhost@EXAMPLE.COM</value>
</property>
 
<property>
     <name>yarn.resourcemanager.keytab</name>
     <value>/etc/krb5.keytab</value>
</property>
 
<property>
     <name>yarn.nodemanager.principal</name>
     <value>yarn/localhost@EXAMPLE.COM</value>
</property>
 
<property>
     <name>yarn.nodemanager.keytab</name>
     <value>/etc/krb5.keytab</value>
</property>
 
<property>
     <name>yarn.nodemanager.container-executor.class</name>
     <value>org.apache.hadoop.yarn.server.nodemanager.LinuxContainerExecutor</value>
</property>
 
<property>
     <name>yarn.nodemanager.linux-container-executor.path</name>
     <value>hadoop-3.0.0-SNAPSHOT/bin/container-executor</value>
</property>
 
<property>
     <name>yarn.nodemanager.linux-container-executor.group</name>
     <value>hadoop</value>
</property>
 
<property>
     <name>yarn.timeline-service.principal</name>
     <value>yarn/localhost@EXAMPLE.COM</value>
</property>
 
<property>
     <name>yarn.timeline-service.keytab</name>
     <value>/etc/krb5.keytab</value>
</property>
 
<property>
     <name>yarn.resourcemanager.webapp.delegation-token-auth-filter.enabled</name>
     <value>true</value>
</property>
 
<property>
     <name>yarn.timeline-service.http-authentication.type</name>
     <value>kerberos</value>
</property>
 
<property>
     <name>yarn.timeline-service.http-authentication.kerberos.principal</name>
     <value>HTTP/localhost@EXAMPLE.COM</value>
</property>
 
<property>
     <name>yarn.timeline-service.http-authentication.kerberos.keytab</name>
     <value>/etc/krb5.keytab</value>
</property>

```

###### mapred-site.xml

```
<property>
     <name>mapreduce.jobhistory.keytab</name>
     <value>/etc/security/keytabs/jhs.service.keytab</value>
</property> 
 
<property>
     <name>mapreduce.jobhistory.principal</name>
     <value>jhs/_HOST@TODO-KERBEROS-DOMAIN</value>
</property> 
 
<property>
     <name>mapreduce.jobhistory.webapp.address</name>
     <value>TODO-JOBHISTORYNODE-HOSTNAME:19888</value>
</property> 
 
<property>
     <name>mapreduce.jobhistory.webapp.https.address</name>
     <value>TODO-JOBHISTORYNODE-HOSTNAME:19889</value>
</property> 
 
<property>
     <name>mapreduce.jobhistory.webapp.spnego-keytab-file</name>
     <value>/etc/security/keytabs/spnego.service.keytab</value>
</property> 
 
<property>
     <name>mapreduce.jobhistory.webapp.spnego-principal</name>
     <value>HTTP/_HOST@TODO-KERBEROS-DOMAIN</value>
</property> 
```


参考：
- http://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.4.2/bk_Security_Guide/content/install-kdc.html
- https://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.4.2/bk_installing_manually_book/content/ch_security_for_manual_installs_chapter.html
- http://blog.csdn.net/wulantian/article/details/42173023
