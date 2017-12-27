
title: 使用portainer监控docker容器
date: 2017-07-14 14:57:22
tags: [youdaonote]
---

我们先监控一个远程的docker daemon，那么这个docker daemon先要允许远程访问才行。

修改113的配置文件/etc/docker/daemon.json
```
{
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn","http://3a35aff6.m.daocloud.io","http://ethanzhu.m.tenxcloud.net"],
  "hosts": ["tcp://0.0.0.0:2375"]
}

```
注意，启用hosts，也就是远程接口之后，本地的命令行就不能用了
```
[root@etl03 ~]# docker images
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
[root@etl03 ~]# docker -H 0.0.0.0:2375 images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
sso                 latest              f8fdedf3edca        13 hours ago        246MB
centos              7.3.1611            67591570dd29        7 months ago        192MB
```


在112上启动portainer
```
docker run -d -p 9000:9000 --rm -v /var/run/docker.sock:/var/run/docker.sock portainer/portainer
```

如果要持久化一些配置文件的话，可以参考以下命令：
```
docker run -d -p 9000:9000 -v /path/on/host/data:/data portainer/portainer
```

配置初始密码，然后

参考：
- https://docs.docker.com/engine/reference/commandline/dockerd/#extended-description
