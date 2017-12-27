
title: openLDAP-安装配置
date: 2016-12-05 17:09:38
tags: [youdaonote]
---

## 介绍
LDAP是一个许多传统高级系统管理员喜欢的服务，但是门槛高。
这里及你进介绍一些实用的规则，最下面会有一些参考。

#### LDAP在局域网中的角色
openLDAP实现了Lightweight Directory Access Protocol协议。目录本身是一个树结构的、可读的数据库。

我们使用openLDAP提供了一个网络验证中心，每个用户登录的时候都要到这里验证身份，如果是第一次的就自动创建他们的home目录。

这篇文章会提到怎样使用openLDAP作为验证中心，并为用户提供元数据。但是，文中使用明文用户名密码在wire中传播的方式是不安全的。因此推荐大家结合Kerberos使用openLDAP。

下面介绍LDAP的安装。

技术角度来看，LDAP目录是由一系列的有层级的entry组成。每个entry都属于某个指定的Object Classes，而且每个entry都包多含个kv键值对，也就是attribute。
entry是使用DN(distinguished name)来区分彼此的。DN由一系列组件构成，这些组件之间用逗号隔开，表示从树的top节点到这个entry的全路径。例如，公司：dc=example,dc=com。员工:cn=person,ou=People,dc=example,dc=com。

entry的objectClasses决定哪些attribute必须有，哪些可以没有。

上面的cn=persion，就是键值对的形式。上面的键(cn,ou,dc)分别代表着Common Name, Organizational Unit以及Domain Component。这是一些常用的术语。

先说位一下我们安装的几个注意点：
 - LDAP与传统的系统用户或者其他数据无关。但是，我们安装过程中会存储一部分信息在/etc/paswd和/etc/group中，然后让这些信息在一个网络中心位置共享。
 - 可以配置LDAP存储用户的密码。密码主要用来验证用户是否有权限访问指定的目录，还有就是验证用户是否知道正确的密码。当某个用户打开一个LDAP client来查看目录的时候，他的DB和密码就用来验证他的权限。当LDAP用来验证用户的时候，他的DB和密码只是用来建立LDAP目录的连接。成功连接就意味着用户知道正确的密码。


#### 粘结层：将LDAP于系统软件整合起来
###### NSS
###### PAM


#### Conventions【约定】

 - Debian GNU平台，或者Ubuntu。
 - 有sudo免密命令。
 - 安装过程中会有很多提示问题，运行：sudo dpkg-reconfigure debconf。然后舒服interface=Dialog，还有priority=low
 - 查看日志：
 > cd /var/log; sudo tail -F daemon.log sulog user.log auth.log debug kern.log syslog dmesg messages kerberos/{krb5kdc,kadmin,krb5lib}.log
 - 我们的测试系统叫做monarch.spinlock.hr，ip地址是192.168.7.12。server和client都会安在同一个机器上，为了明确区分client相对于monarch.spinlock.hr，server对应ldap1.spinlock.hr。按照下面配置/etc/hosts：
 > 192.168.7.12	monarch.spinlock.hr monarch krb1.spinlock.hr krb1 ldap1.spinlock.hr ldap1
**注意**
有的机器域名会被设置成127.0.0.1，这可能会出现问题。需要保证/etc/hosts里只能是: 127.0.0.1   localhost
 - 最后，测试是不是成功了。
 ```
 ping -c1 localhost
PING localhost (127.0.0.1) 56(84) bytes of data.
....

ping -c1 monarch
PING monarch.spinlock.hr (192.168.7.12) 56(84) bytes of data.
....

ping -c1 ldap1
PING ldap1.spinlock.hr (192.168.7.12) 56(84) bytes of data.
....
 ```
 
 
 
 
 
 
 ## 2. OpenLDAP
#### 2.1 server安装
openLDAP的server叫做slapd。
> sudo apt-get install slapd ladp-tuils

Debconf如下：
```
Omit OpenLDAP server configuration? No

DNS domain name: spinlock.hr

Organization name? spinlock.hr

Administrator password: PASSWORD

Confirm password: PASSWORD

Database backend to use: HDB

Do you want the database to be removed when slapd is purged? No

Allow LDAPv2 protocol? No
```
安装完成后slapd会自动启动。

######  2.1.1初始配置
server端的配置主要包含ObjectClasses、attribute、syntax、matching rules以及其他的LDAP数据结构的细节。配置文件目录是/etc/ldap/slapd.d/，每次启动的时候都会加载这些配置文件。

这些配置文件是以LDIF的形式保存的，预期是想使用标准的LDAP数据修改工具进行修改，然后slapd就会把新的配置存储在LDIF文件中，以保证下次重启生效。其实手动修改也是可以的。这种只能在运行时修改配置的方式叫做olc(on-line configuration)，也叫cn=config以及slapd.d。

以前版本的openLDAP是使用标准的txt配置文件/etc/ladp/slapd.conf，虽然目前也还是支持的，但是并不推荐使用这个文件。因为修改之后需要重启才能生效，cn=config是当前默认的方式。

***注意***

如果你的系统上游/etc/ldap/slapd.conf文件，但是没有/etc/ldap/slapd.d目录的话，运行以下命令：
> sudo slaptest -f /etc/ldap/slapd.conf -F /etc/ldap/slapd.d

然后确认你的slapd是使用-F /etc/ldap/slapd.d启动的，而不是-f /etc/ldap/slapd.conf
> grep -e -F /etc/init.d/slapd
> ps aux | grep slapd

######  2.1.1.1 CN=CONFIG配置

cn=config是一个特殊的LDAP数据库，它保存着OpenLDAP server的配置，运行时可读可写。对它的修改直接影响当前运行的server，也会修改/etc/ldap/slapd.d/下对应的配置文件。

修改配置的时候我们需要利用已经存在的默认的cn=config配置，能让本地root用户连接到/var/run/slapd/ldapi的slapd socket修改cn=config数据库。

有了这个途径，我们先验证一下所有的LDAP schema定义(core,cosine,nis,inetorgperson)都已经加载成功。
在这一步出现重复entry问题的话不要紧张，目的在于知道我们的client以local user的身份验证身份，然后连接到slapd socket "-Y EXTERNAL -H ldapi:///":
```
sudo ldapadd -c -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/core.ldif
sudo ldapadd -c -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/cosine.ldif
sudo ldapadd -c -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/nis.ldif
sudo ldapadd -c -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/inetorgperson.ldif
```
然后我们把默认的olcLogLevel: none替换成olcLogLevel: 256:
```
echo "dn: cn=config
changetype: modify
replace: olcLogLevel
olcLogLevel: 256" > /var/tmp/loglevel.ldif

sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f /var/tmp/loglevel.ldif
```
然后为attribute uid添加索引"eq":
```
echo "
dn: olcDatabase={1}hdb,cn=config
changetype: modify
add: olcDbIndex
olcDbIndex: uid eq
" > /var/tmp/uid_eq.ldif

sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f /var/tmp/uid_eq.ldif
```

最后，以同样的方式允许LDAP管理员用户对于cn=config可读可写，dc=spinlock,dc=hr数据树就也可写了。只要用户知道正确的管理员DN和密码，就可以很方便快捷地修改slapd的配置。使用图形化的LDAP client来查看和修改配置对于初学者来说会更好上手一些：
```
echo "dn: olcDatabase={0}config,cn=config
changetype: modify
add: olcAccess
olcAccess: to * by dn="cn=admin,dc=spinlock,dc=hr" write" > /var/tmp/access.ldif

sudo ldapmodify -c -Y EXTERNAL -H ldapi:/// -f /var/tmp/access.ldif
```

###### 2.1.1.2 spapd.conf配置【失效】
###### 2.1.1.3 client端配置
server已经可以运行了。下面配置所有LDAP client的配置文件 /etc/ldap/ldap.conf。
```
BASE  dc=spinlock, dc=hr 
URI ldap://192.168.7.12/
```


#### 2.1.2 initial test 初始测试
可以测试一下我们安装的成果了。我们的openLDAP server只包含基本的读取操作。从LDAP的角度来看，读取操作被成为search。要使用命令行工具执行search，我们需要安装ldapsearch和slapcat。

ldapsearch使用LDAP协议执行在线操作。

SLAPCAT执行离线操作，直接修改本地文件系统里的文件。因此，这些命令只能在openLDAP server的本机执行，而且还需要特殊权限。向数据库写数据之前，需要先停掉openLDAP server。

在以上两个命令的输出中，一会注意到有两个LDAP entry，一个代表树的top节点，另一个代表LDAP 管理员的entry。在slapcat的输出中，不会像ldapsearch一样把额外的attributes打印出来。
```
ldapsearch -x

# extended LDIF
#
# LDAPv3
# base <dc=spinlock, dc=hr> (default) with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#

# spinlock.hr
dn: dc=spinlock,dc=hr
objectClass: top
objectClass: dcObject
objectClass: organization
o: spinlock.hr
dc: spinlock

# admin, spinlock.hr
dn: cn=admin,dc=spinlock,dc=hr
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: admin
description: LDAP administrator

# search result
search: 2
result: 0 Success

# numResponses: 3
# numEntries: 2
```
```
sudo slapcat

dn: dc=spinlock,dc=hr
objectClass: top
objectClass: dcObject
objectClass: organization
o: spinlock.hr
dc: spinlock
structuralObjectClass: organization
entryUUID: 350a2db6-87d3-102c-8c1c-1ffeac40db98
creatorsName:
modifiersName:
createTimestamp: 20080316183324Z
modifyTimestamp: 20080316183324Z
entryCSN: 20080316183324.797498Z#000000#000#000000

dn: cn=admin,dc=spinlock,dc=hr
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: admin
description: LDAP administrator
userPassword:: e2NyeXB0fVdSZDJjRFdRODluNHM=
structuralObjectClass: organizationalRole
entryUUID: 350b330a-87d3-102c-8c1d-1ffeac40db98
creatorsName:
modifiersName:
createTimestamp: 20080316183324Z
modifyTimestamp: 20080316183324Z
```


#### 创建基本树结构


