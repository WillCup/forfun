
title: 自制RPM包并发布到自己的yum源
date: 2017-04-27 11:29:46
tags: [youdaonote]
---

使用一个普通用户执行，以免对系统造成重伤。


```
yum install -y rpmdevtools rpm-build
```


自定义工作空间
---

方案1：

```
rpmdev-setuptree
```
方案2

创建文件~/.rpmmacros
```
%_topdir        /home/will/rpmbuild
```
配置了空间之后，还得手动创建一下
```
mkdir /home/will/rpmbuild
```

创建需要的目录
```
cd ~/rpmbuild  
mkdir -pv {BUILD,BUILDROOT,RPMS,SOURCES,SPECS,SRPMS} 
```
- BUILD #编译之前，如解压包后存放的路径
- BUILDROOT #编译后存放的路径
- RPMS #打包完成后rpm包存放的路径
- SOURCES #源包所放置的路径
- SPECS #spec文档放置的路径
- SPRMS #源码rpm包放置的路径

注：一般我们都把源码打包成tar.gz格式然后存放于SOURCES路径下，而在SPECS路径下编写spec文档，通过命令打包后，默认会把打包后的rpm包放在RPMS下，而源码包会被放置在SRPMS下

源码放到SOURCES目录
---
```
cp /tmp/redis-3.2.8.tar.gz SOURCES
```

SPECS下创建配置
---
到SPECS下创建指定配置，编辑的时候会自动生成模板。编辑`SPECS/redis.spec`
```spec
Name:		redis
Version:	3.2.8	
Release:	3%{dist}
Summary:	redis from will	

Group:		System Environment/Daemons
License:	GPLv2
URL:		https://github.com/willcup
Source0:	redis-3.2.8.tar.gz

Packager:       WillCup <willcup@163.com> 
BuildRequires:	gcc,make
Requires:	openssl,chkconfig
BuildRoot:      %_topdir/BUILDROOT


%define PortFindDir /usr/local/will_redis

%description
this is build from will, just for ambari service.

%prep
%setup -q

%build
make %{?_smp_mflags}


%install
[ ! -e $RPM_BUILD_ROOT%{PortFindDir} ] && mkdir -p $RPM_BUILD_ROOT%{PortFindDir}
cp -rfv * $RPM_BUILD_ROOT%{PortFindDir}
[ ! -e $RPM_BUILD_ROOT/etc/init.d ] && mkdir -p $RPM_BUILD_ROOT/etc/init.d
[ ! -e $RPM_BUILD_ROOT/usr/local/bin ] && mkdir -p $RPM_BUILD_ROOT/usr/local/bin
cp -vf $RPM_BUILD_ROOT%{PortFindDir}/utils/redis_init_script $RPM_BUILD_ROOT/etc/init.d/redis
cp -vf $RPM_BUILD_ROOT%{PortFindDir}/src/redis-server $RPM_BUILD_ROOT/usr/local/bin
cp -vf $RPM_BUILD_ROOT%{PortFindDir}/src/redis-cli $RPM_BUILD_ROOT/usr/local/bin
#mkdir -p "$RPM_BUILD_ROOT"
#cp -rvf * "$RPM_BUILD_ROOT"
#ls -l "$RPM_BUILD_ROOT"
echo "no reason to install redis, just move it"



%files
%defattr (-,root,root,0755)
%dir %{PortFindDir}
%attr(0755, root, root) %{PortFindDir}/*
%attr(0755, root, root)	/etc/init.d/redis
%attr(0755, root, root) /usr/local/bin
%doc



%postun	
rm -fr %(PortFindDir}
rm -vf /usr/local/bin/redis-*
rm -vf /etc/init.d/redis

%changelog
* Fri Dec 29 2016 willcup <willcup@163.com> - 3.2.8-1 
- Initial version just changelog

```

其实上面的buildroot就相当于是后面rpm安装时候的系统根目录。所以一般都需要指定一个子目录。不然是行不通的，会发现弄完以后RPM包里没有任何文件。

开始构建
---
```
rpmbuild -ba redis.spec
```

可以看到已经有了结果

这个是构建过程中的源码目录：
```
[will@datanode08 rpmbuild]$ ll BUILD/redis-3.2.8/
total 204
-rw-r--r--.  1 will will 85775 Feb 12 23:14 00-RELEASENOTES
-rw-r--r--.  1 will will    53 Feb 12 23:14 BUGS
-rw-r--r--.  1 will will  1805 Feb 12 23:14 CONTRIBUTING
-rw-r--r--.  1 will will  1487 Feb 12 23:14 COPYING
-rw-r--r--.  1 will will     0 Apr 27 12:25 debugfiles.list
-rw-r--r--.  1 will will     0 Apr 27 12:25 debuglinks.list
-rw-r--r--.  1 will will     0 Apr 27 12:25 debugsources.list
drwxr-xr-x.  7 will will  4096 Apr 27 12:25 deps
-rw-r--r--.  1 will will    11 Feb 12 23:14 INSTALL
-rw-r--r--.  1 will will   151 Feb 12 23:14 Makefile
-rw-r--r--.  1 will will  4223 Feb 12 23:14 MANIFESTO
-rw-r--r--.  1 will will  6834 Feb 12 23:14 README.md
-rw-r--r--.  1 will will 46695 Feb 12 23:14 redis.conf
-rwxr-xr-x.  1 will will   271 Feb 12 23:14 runtest
-rwxr-xr-x.  1 will will   280 Feb 12 23:14 runtest-cluster
-rwxr-xr-x.  1 will will   281 Feb 12 23:14 runtest-sentinel
-rw-r--r--.  1 will will  7606 Feb 12 23:14 sentinel.conf
drwxr-xr-x.  2 will will  4096 Apr 27 12:25 src
drwxr-xr-x. 10 will will  4096 Feb 12 23:14 tests
drwxr-xr-x.  7 will will  4096 Feb 12 23:14 utils
```
打包完成后rpm包存放的路径：
```
[will@datanode08 rpmbuild]$ ll RPMS/x86_64/
total 8
-rw-rw-r--. 1 will will 1752 Apr 27 12:25 redis-3.2.8-1.el6.x86_64.rpm
-rw-rw-r--. 1 will will 1880 Apr 27 12:25 redis-debuginfo-3.2.8-1.el6.x86_64.rpm
```
源码rpm包放置的路径
```
[will@datanode08 rpmbuild]$ ll SRPMS/
total 1516
-rw-rw-r--. 1 will will 1549338 Apr 27 12:25 redis-3.2.8-1.el6.src.rpm
```

然而查看rpm包的文件，竟然什么都没有。
```
[will@datanode08 rpmbuild]$ rpm -qlp RPMS/x86_64/redis-3.2.8-1.el6.x86_64.rpm 
(contains no files)

```


修改之后，能够正常安装与使用了。

使用 rpm -ivh xx.rpm成功测试。


下一步，将RPM发布至我们自有的yum源上。

按照其他的类似hadoop之类，创建一个redis目录，然后把我们的rpm包放上去

目录样子
```
[root@datanode21 2.4.2.0]# ll
total 2146284
.....
drwxr-xr-x  2 ambari-qa users       4096 Apr 25  2016 hadoop
drwxr-xr-x  2 ambari-qa users       4096 Apr 25  2016 hadooplzo
drwxr-xr-x  2 ambari-qa users       4096 Apr 25  2016 hbase
.......
drwxr-xr-x  2 ambari-qa users       4096 Apr 25  2016 oozie
drwxr-xr-x  2 ambari-qa users       4096 Apr 25  2016 phoenix
drwxr-xr-x  2 ambari-qa users       4096 Apr 25  2016 pig
drwxr-xr-x  2 ambari-qa users       4096 Apr 25  2016 ranger
drwxr-xr-x  2 root      root        4096 Apr 27 16:45 redis
You have new mail in /var/spool/mail/root
[root@datanode21 2.4.2.0]# pwd
/server/www/html/hdp/HDP/centos6/2.x/updates/2.4.2.0
```

先看下我们的HDP.repo
```
[HDP-2.4]
name=HDP-2.4
baseurl=http://10.2.19.110/hdp/HDP/centos6/2.x/updates/2.4.2.0

path=/
enabled=1
gpgcheck=0
```

可以看到是同一个目录，我们需要在这个目录下重新执行`createrepo`
```
[root@datanode21 2.4.2.0]# createrepo .
Spawning worker 0 with 182 pkgs
Workers Finished
Gathering worker results

Saving Primary metadata
Saving file lists metadata
Saving other metadata
Generating sqlite DBs
Sqlite DBs complete
[root@datanode21 2.4.2.0]# ll repodata
total 856
-rw-r--r-- 1 root root 373992 Apr 27 17:59 458b198ae4b61c02b0f7bb03ae9421365dcb2076ffb6087e6808e7915ec31778-filelists.sqlite.bz2
-rw-r--r-- 1 root root   9092 Apr 27 17:59 9a59871d45121b0be43f188ae7ffe788e2eceb07d03648717323dedd26faf567-other.xml.gz
-rw-r--r-- 1 root root  12162 Apr 27 17:59 bb3c2dc4c1f7ae17b172f104e3df5171d0d09db797a5f5a9227ab4ae424a45b7-other.sqlite.bz2
-rw-r--r-- 1 root root  37245 Apr 27 17:59 c23cec21ae4b3e9b89a029b593362ca1e2483316e1d07843a9035ad8e3c93053-primary.xml.gz
-rw-r--r-- 1 root root 363468 Apr 27 17:59 c521ec682137ff19e61f7ec91e86c725d6d6fa7553be01c783a67ffb68b30afe-filelists.xml.gz
-rw-r--r-- 1 root root  65483 Apr 27 17:59 eb98320b3ad040203135b91f26caee107900f2e9bdc86a4be6c5218fcc505036-primary.sqlite.bz2
-rw-r--r-- 1 root root   2996 Apr 27 17:59 repomd.xml

```

然后更新客户机上，也就是要安装redis 的机器上的repo信息。
```
yum clean all
yum update
yum search redis
```
能够搜到，至此结束。



问题
---
我们要构建redis，主要是为了适应既有ambari的redis service，它执行安装的命令是`yum install redis.3.2.8`。当然我们是可以修改一下他的install代码的，但是考虑到以后其他组件很有可能还是通过yum安装，那还是现在就把自己动手构建yum的rpm包这个问题解决了吧，以绝后患。

问题来了，上面我们搜到的只是`redis.x86_64`,实际安装的时候请求的是3.2.8，需要版本号加进去才行。看到spec里有version关键字。。。折腾了好半天，都不是，构建出来search的时候都还是只有redis。

后来参考既有的hadoop、flume等yum源，成功搞定。
```
flume_2_4_2_0_258-1.5.2.2.4.2.0-258.el6.noarch.rpm	25-Apr-2016 19:54	40M	 
	
flume_2_4_2_0_258-agent-1.5.2.2.4.2.0-258.el6.noarch.rpm	25-Apr-2016 19:54	6.4K	 
```

发现这个版本号貌似是跟着文件夹的。

所以，修改如下:
- 修改redis.spec为redis-3.2.8.spec
- 编辑spec文件
```
Name:           redis-3.2.8
Version:        1_0_0

URL:            https://github.com/willcup
Source0:        redis-3.2.8.tar.gz

```
关键在于Name，后面都是喽啰
- 这样解压后会去redis-3.2.8-1_0_0，也就是根据name和version生成的规则，所以可能要调整一下tar.gz里的文件夹名称
- 重新构建预发布，即可
```
redis-3.2.8-1_0_0-4.el6.x86_64.rpm

yum search redis
redis.x86_64 : redis from will
redis-3.2.8.x86_64 : redis 3 2 8 from will
```

参考：
- https://docs.fedoraproject.org/en-US/Fedora_Draft_Documentation/0.1/html/Packagers_Guide/chap-Packagers_Guide-Spec_File_Reference-Preamble.html
- https://fedoraproject.org/wiki/How_to_create_an_RPM_package/zh-cn#.E5.AE.9E.E4.BE.8B
