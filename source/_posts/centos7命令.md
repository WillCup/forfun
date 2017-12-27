
title: centos7命令
date: 2017-01-17 16:33:35
tags: [youdaonote]
---

修改主机名
---
```
hostnamectl set-hostname <host-name>
```

永久修改：
```
hostnamectl --static set-hostname <host-name>
```

服务相关
---
安装服务
```
systemctl enable docker.service
```

启动服务
```
systemctl start docker
```


