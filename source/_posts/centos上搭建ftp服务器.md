
title: centos上搭建ftp服务器
date: 2017-11-20 14:46:02
tags: [youdaonote]
---

首先安装服务：
```
yum install -y vsftpd
```

创建只读的dataman用户：
```
useradd -s /sbin/nologin dataman
passwd dataman
```

编辑配置文件`/etc/vsftpd/vsftpd.conf`：
```

# 修改默认端口
listen_port=6888

# 禁止匿名登陆
anonymous_enable=NO

# 指定用户单独的配置信息所在目录
user_config_dir=/var/ftp

# 不许用户登陆后切换目录
chroot_local_user=YES
```

然后到`/var/ftp`下新建dataman用户的ftp配置文件，这里只需要指定他的root目录就好了。
```
local_root=/server/files
```


参考
- https://jasonhzy.github.io/2016/03/11/linux-ftp/
- http://cn.linux.vbird.org/linux_server/0410vsftpd.php


坑

我们的网络原因导致，测试环境要连接线上的ftp服务器的话，需要通过一层运维提供的HAProxy代理。

这就引出了ftp服务器主动传输与被动传输的概念区别。

FTP协议有两种工作方式：PORT方式和PASV方式，中文意思为主动式和被动式。

PORT（主动）方式的连接过程是：客 户端向服务器的FTP端口（默认是21）发送连接请求，服务器接受连接，建立一条命令链路。当需要传送数据时，客户端在命令链路上用PORT命令告诉服务 器：“我打开了XXXX端口，你过来连接我”。于是服务器从20端口向客户端的XXXX端口发送连接请求，建立一条数据链路来传送数据。

PASV（被动）方式的连接过程是：客 户端向服务器的FTP端口（默认是21）发送连接请求，服务器接受连接，建立一条命令链路。当需要传送数据时，服务器在命令链路上用PASV命令告诉客户 端：“我打开了XXXX端口，你过来连接我”。于是客户端向服务器的XXXX端口发送连接请求，建立一条数据链路来传送数据。
概括： 
--------------------------------------------------------------------------------
主动模式：服务器向客户端敲门，然后客户端开门
被动模式：客户端向服务器敲门，然后服务器开门


所以，如果你是如果通过代理上网的话，就不能用主动模式，因为服务器敲的是上网代理服务器的门，而不是敲客户端的门
而且有时候，客户端也不是轻易就开门的，因为有防火墙阻挡，除非客户端开放大于1024的高端端口


vsftpd服务器端修改配置：
```
# passive
tcp_wrappers=YES
pasv_promiscuous=NO
# 这个比较关键，10.103.70.27是我们的haproxy所在的IP
pasv_address=10.103.70.27
port_enable=YES
port_promiscuous=NO
pasv_enable=YES
pasv_min_port=10000
pasv_max_port=10010
```

同时，HAProxy也需要配置到这些端口范围
```
listen ftp
    bind *:6888,*:10000-10010
    mode tcp
    server ftpserver 10.2.19.62 check inter 3000 port 6888
```

- http://blog.sina.com.cn/s/blog_7f1d56650102v57p.html
- http://www.cnblogs.com/exclm/archive/2009/05/08/1452893.html
- http://blog.sina.com.cn/s/blog_5cdb72780100jwjt.html
- https://serverfault.com/questions/441721/ftp-through-haproxy
