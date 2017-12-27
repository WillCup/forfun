
title: 使用harbor安装docker-registry
date: 2017-01-17 16:34:40
tags: [youdaonote]
---

1. 安装docker和docker compose
---
```
sudo curl -sSL https://get.docker.com/ | sh
# 设置Docker以非Root用户运行，确保安全。
sudo usermod -aG docker your-user
# 安装Compose：
curl -L https://github.com/docker/compose/releases/download/1.7.0-rc1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

2. 获取SSL证书[可选]

如果使用域名绑定私有仓库，就必须开启SSL

```
git clone https://github.com/letsencrypt/letsencrypt
cd letsencrypt
./letsencrypt-auto certonly -d docker.will.me
```

3. 安装harbor

```
wget https://github.com/vmware/harbor/releases/download/0.5.0/harbor-offline-installer-0.5.0.tgz
tar zxvf harbor-offline-installer-0.5.0.tgz
```

修改一下harbor.cfg里的hostname
```
hostname = 10.1.5.129
```
其他的都没敢改，因为安装文档也没有特别深入的说明，暂时跑个demo而已，不要太拼。

但是目前有疑问的有两点：

> 1. mysql的连接没有指定，只有一个db_password, 那元数据存在哪？
> 2. 没有指定docker image的数据文件存储位置，理论上这会是一个比较大的数据量

带着疑问，先开始安装：
```
[root@controller harbor]# ./install.sh 

[Step 0]: checking installation environment ...

Note: docker version: 1.12.6

Note: docker-compose version: 1.7.0

[Step 1]: loading Harbor images ...
dd60b611baaa: Loading layer [==================================================>] 133.2 MB/133.2 MB
e541edf73dfb: Loading layer [==================================================>] 3.072 kB/3.072 kB
Loaded image: vmware/harbor-log:0.5.0>                                          ]    512 B/3.072 kB
f96222d75c55: Loading layer [==================================================>] 128.9 MB/128.9 MB
5a3deeae1518: Loading layer [==================================================>] 4.608 kB/4.608 kB
Loaded image: vmware/harbor-db:0.5.0                                            ]    512 B/4.608 kB
7b612c5c8104: Loading layer [==================================================>] 1.536 kB/1.536 kB
5c07a2fe7c73: Loading layer [==================================================>] 20.94 MB/20.94 MB
Loaded image: vmware/harbor-jobservice:0.5.0                                    ] 229.4 kB/20.94 MB
fe4c16cbf7a4: Loading layer [==================================================>] 128.9 MB/128.9 MB
c4a8b7411af4: Loading layer [==================================================>] 60.57 MB/60.57 MB
3f117c44afbb: Loading layer [==================================================>] 3.584 kB/3.584 kB
Loaded image: nginx:1.11.5r [=======>                                           ]    512 B/3.584 kB
4fe15f8d0ae6: Loading layer [==================================================>] 5.046 MB/5.046 MB
3bb5bc5ad373: Loading layer [==================================================>] 2.048 kB/2.048 kB
Loaded image: registry:2.5.0[============>                                      ]    512 B/2.048 kB
Loaded image: photon:1.0
41be4a382617: Loading layer [==================================================>] 58.31 MB/58.31 MB
32ec73a38f19: Loading layer [==================================================>] 23.62 MB/23.62 MB
Loaded image: vmware/harbor-ui:0.5.0                                            ] 262.1 kB/23.62 MB


[Step 2]: preparing environment ...
generated and saved secret key
Generated configuration file: ./common/config/nginx/nginx.conf
Generated configuration file: ./common/config/ui/env
Generated configuration file: ./common/config/ui/app.conf
Generated configuration file: ./common/config/registry/config.yml
Generated configuration file: ./common/config/db/env
Generated configuration file: ./common/config/jobservice/env
Generated configuration file: ./common/config/jobservice/app.conf
Generated configuration file: ./common/config/ui/private_key.pem
Generated configuration file: ./common/config/registry/root.crt
The configuration files are ready, please use docker-compose to start the service.


[Step 3]: checking existing instance of Harbor ...


[Step 4]: starting Harbor ...
Creating network "harbor_default" with the default driver
Creating harbor-log
Creating registry
Creating harbor-db
Creating harbor-ui
Creating harbor-jobservice
Creating nginx

✔ ----Harbor has been installed and started successfully.----

Now you should be able to visit the admin portal at http://10.1.5.129. 
For more details, please visit https://github.com/vmware/harbor .


```
可以看到这是安装了一系列的docker iamge啊，难怪有离线和在线之分了。


闲着没事儿，看一下install.sh都干了啥
```
# 检查docker和docker-compose是不是符合版本要求
check_docker
check_dockercompose

# load harbor的压缩包到当前docker的images库里
if [ -f harbor*.tgz ]
then
        h2 "[Step $item]: loading Harbor images ..."; let item+=1
        docker load -i ./harbor*.tgz
fi
echo ""

# prepare就是读取harbor.cfg配置文件，准备环境变量
h2 "[Step $item]: preparing environment ...";  let item+=1
if [ -n "$host" ]
then
        sed "s/^hostname = .*/hostname = $host/g" -i ./harbor.cfg
fi
./prepare

# 查看有没有正在运行的harbor实例，有的话就停掉
h2 "[Step $item]: checking existing instance of Harbor ..."; let item+=1
if [ -n "$(docker-compose -f docker-compose*.yml ps -q)"  ]
then
        note "stopping existing Harbor instance ..."
        docker-compose -f docker-compose*.yml down
fi
echo ""

# 启动新的harbor实例
h2 "[Step $item]: starting Harbor ..."
docker-compose -f docker-compose*.yml up -d

```
以上是主要步骤，其他的全都是不重要的。

查看一下当前的docker进程：
```
[root@controller harbor]# docker ps
CONTAINER ID        IMAGE                            COMMAND                  CREATED             STATUS              PORTS                                      NAMES
b8ed2fce6816        nginx:1.11.5                     "nginx -g 'daemon off"   5 minutes ago       Up 4 minutes        0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp   nginx
8408e2b03a4b        vmware/harbor-jobservice:0.5.0   "/harbor/harbor_jobse"   5 minutes ago       Up 4 minutes                                                   harbor-jobservice
48b280c1caba        vmware/harbor-ui:0.5.0           "/harbor/harbor_ui"      6 minutes ago       Up 5 minutes                                                   harbor-ui
0235e55493d0        vmware/harbor-db:0.5.0           "docker-entrypoint.sh"   6 minutes ago       Up 5 minutes        3306/tcp                                   harbor-db
5bc55096ba78        library/registry:2.5.0           "/entrypoint.sh serve"   6 minutes ago       Up 5 minutes        5000/tcp                                   registry
805f744b40ae        vmware/harbor-log:0.5.0          "/bin/sh -c 'crond &&"   6 minutes ago       Up 6 minutes        0.0.0.0:1514->514/tcp                      harbor-log
```

该起来的全起来了，下面看一下harbor-db的参数，把数据弄到哪里去了？

docker inspect 看了一下
```
"Mounts": [
            {
                "Source": "/data/database",
                "Destination": "/var/lib/mysql",
                "Mode": "rw",
                "RW": true,
                "Propagation": "rprivate"
            }
        ],

```
mysql数据放到/data/database了。

再看一下registry的挂载情况：
```
"Mounts": [
            {
                "Source": "/data/registry",
                "Destination": "/storage",
                "Mode": "rw",
                "RW": true,
                "Propagation": "rprivate"
            },
            {
                "Source": "/server/harbor/common/config/registry",
                "Destination": "/etc/registry",
                "Mode": "rw",
                "RW": true,
                "Propagation": "rprivate"
            },
            {
                "Name": "e8aab221ad99604d0f660695895eec3ff425cc0847ac5f0a3a803ab83e70275a",
                "Source": "/var/lib/docker/volumes/e8aab221ad99604d0f660695895eec3ff425cc0847ac5f0a3a803ab83e70275a/_data",
                "Destination": "/var/lib/registry",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            }
        ],
```

此时以上两个目录还都是空的
```
[root@controller harbor]# ll /server/harbor/common/config/registry
total 8
-rw-r--r-- 1 root root  694 Jan 18 03:05 config.yml
-rw-r--r-- 1 root root 2151 Jan 18 03:05 root.crt
[root@controller harbor]# ll /data/database
total 110608
-rw-rw---- 1 systemd-bus-proxy ssh_keys       56 Jan 18 03:07 auto.cnf
-rw-rw---- 1 systemd-bus-proxy ssh_keys 12582912 Jan 18 03:25 ibdata1
-rw-rw---- 1 systemd-bus-proxy ssh_keys 50331648 Jan 18 03:25 ib_logfile0
-rw-rw---- 1 systemd-bus-proxy ssh_keys 50331648 Jan 18 03:06 ib_logfile1
drwx------ 2 systemd-bus-proxy ssh_keys     4096 Jan 18 03:07 mysql
drwx------ 2 systemd-bus-proxy ssh_keys     4096 Jan 18 03:07 performance_schema
drwx------ 2 systemd-bus-proxy ssh_keys     4096 Jan 18 03:07 registry
```



参考：
https://github.com/vmware/harbor/blob/master/docs/installation_guide.md

4. 使用
---
netstat -ntlp 发现80端口被启动了，应该就是它。下面就登陆一下harbor看一下


docker-compose.yml配置文件
```yml
version: '2'
services:
  log:
    image: vmware/harbor-log:0.5.0
    container_name: harbor-log
    restart: always
    volumes:
      - /var/log/harbor/:/var/log/docker/
    ports:
      - 1514:514
  registry:
    image: library/registry:2.5.0
    container_name: registry
    restart: always
    volumes:
      - /data/registry:/storage
      - ./common/config/registry/:/etc/registry/
    environment:
      - GODEBUG=netdns=cgo
    command:
      ["serve", "/etc/registry/config.yml"]
    depends_on:
      - log
    logging:
      driver: "syslog"
      options:
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "registry"
  mysql:
    image: vmware/harbor-db:0.5.0
    container_name: harbor-db
    restart: always
    volumes:
      - /data/database:/var/lib/mysql
    env_file:
      - ./common/config/db/env
    depends_on:
      - log
    logging:
      driver: "syslog"
      options:
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "mysql"
  ui:
    image: vmware/harbor-ui:0.5.0
    container_name: harbor-ui
    env_file:
      - ./common/config/ui/env
    restart: always
    volumes:
      - ./common/config/ui/app.conf:/etc/ui/app.conf
      - ./common/config/ui/private_key.pem:/etc/ui/private_key.pem
      - /data:/harbor_storage
    depends_on:
      - log
    logging:
      driver: "syslog"
      options:
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "ui"
  jobservice:
    image: vmware/harbor-jobservice:0.5.0
    container_name: harbor-jobservice
    env_file:
      - ./common/config/jobservice/env
    restart: always
    volumes:
      - /data/job_logs:/var/log/jobs
      - ./common/config/jobservice/app.conf:/etc/jobservice/app.conf
    depends_on:
      - ui
    logging:
      driver: "syslog"
      options:
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "jobservice"
  proxy:
    image: nginx:1.11.5
    container_name: nginx
    restart: always
    volumes:
      - ./common/config/nginx:/etc/nginx
    ports:
      - 80:80
      - 443:443
    depends_on:
      - mysql
      - registry
      - ui
      - log
    logging:
      driver: "syslog"
      options:
        syslog-address: "tcp://127.0.0.1:1514"
        tag: "proxy"
```


5. 上传镜像
找一个安装了docker的机器，在/etc/docker/daemon.json文件中指定insecure-registries【因为我们上面并没有启用HTTPS协议】
```json

{
  "insecure-registries": ["10.1.5.129"],
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn","http://3a35aff6.m.daocloud.io","http://ethanzhu.m.tenxcloud.net"]
}

```

把本地的镜像上传到
```
[root@dnode2 ~]# docker tag willr 10.1.5.129/library/willr:0.1
[root@dnode2 ~]# docker login 10.1.5.129
Username: admin
Password: 
Login Succeeded
[root@dnode2 ~]# docker images
REPOSITORY                   TAG                 IMAGE ID            CREATED             SIZE
10.1.5.129/library/willr     0.1                 2a33c792d4aa        5 hours ago         718.7 MB
willr                        latest              2a33c792d4aa        5 hours ago         718.7 MB
<none>                       <none>              3dabf60e7d97        17 hours ago        714.7 MB
uhopper/hadoop-nodemanager   2.7.2               046a3f819f63        41 hours ago        715.4 MB
r-base                       3.3.2               dbee2cbef542        4 weeks ago         682.8 MB
r-base                       latest              dbee2cbef542        4 weeks ago         682.8 MB
[root@dnode2 ~]# docker push 10.1.5.129/library/willr:0.1
The push refers to a repository [10.1.5.129/library/willr]
0748650e84ed: Pushed 
72547fe7a086: Pushing [==================================================>] 509.8 MB
2cd05b2f5db4: Pushed 
72547fe7a086: Pushed 
c610e92140d8: Pushed 
3b04b5a81444: Pushed 
29d255cec883: Pushed 
29d255cec883: Preparing 

0.1: digest: sha256:9eb5d67aa235a1d7724b659310e8947c7e6cf967bef7ae02b955225fc7eb193a size: 1791
[root@dnode2 ~]# 

```

页面中的显示
[]()

参考：
- http://www.jianshu.com/p/141855241f2d
- https://vmware.github.io/harbor/index_cn.html
- https://my.oschina.net/u/1540325/blog/702260
- http://www.zimug.com/317.html
