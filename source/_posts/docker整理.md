
title: docker整理
date: 2017-01-17 15:41:46
tags: [youdaonote]
---

centos7的yum源
---
- 清华的docker源
https://mirrors.tuna.tsinghua.edu.cn/help/docker/
- 阿里的源
http://mirrors.aliyun.com/

registry mirror
---
编辑或添加文件 /etc/docker/daemon.json
```
{
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn","http://3a35aff6.m.daocloud.io","http://ethanzhu.m.tenxcloud.net"]
}
```

之后重启docker-daemon就可以了。

参考：
- http://ethanzhu.leanote.com/post/docker-%E6%9B%B4%E6%94%B9


修改docker daemon配置
---

查了好多资料，都说直接修改/etc/default/docker里的DOCKER_OPTS就可以制定registry mirror、镜像存储路径等各种docker daemon的启动参数，但是试了N次都没有成功，

其实细想一下，从启动脚本入手/etc/systemd/system/docker.service，官方说要在/etc/systemd/system/docker.service.d目录下创建自己的conf文件，修改启动命令的参数。我创建了一个will.conf
```
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd --graph="/server/dspace"
```
之后reload一下daemon的配置。供参考的[参数信息](http://docs-stage.docker.com/v1.10/engine/reference/commandline/daemon/)

要修改的主要参数在于**启动service里的ExecStart**

```
systemctl daemon-reload
```
再去看指定的目录和docker info信息，发现已经修改过来了。
```
[root@dnode2 ~]# ll /server/dspace/
total 32
drwx------ 2 root root 4096 Jan 18 23:48 containers
drwx------ 4 root root 4096 Jan 18 23:48 devicemapper
drwx------ 3 root root 4096 Jan 18 23:48 image
drwxr-x--- 3 root root 4096 Jan 18 23:48 network
drwx------ 2 root root 4096 Jan 18 23:48 swarm
drwx------ 2 root root 4096 Jan 18 23:48 tmp
drwx------ 2 root root 4096 Jan 18 23:48 trust
drwx------ 2 root root 4096 Jan 18 23:48 volumes
[root@dnode2 ~]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE

```
刚才的images也都查不到了，停掉docker daemon，手动把原来的镜像全都copy过来，启动docker daemon就可以了。


综上，其实已经有两种方式配置docker daemon的启动参数了，一个是在service.docker.d目录里添加自己id配置文件will.conf，直接修改启动命令并指定参数。另一个就是编辑/etc/docker/daemon.json文件。

应该是两种方式都可以行得通，但是命令行和json的参数并不完全对应，需要找一下json的mapping key，现在觉得json 的方式会更加友好一些。

参考
- 官网 https://docs.docker.com/engine/admin/systemd/
- http://blog.csdn.net/l6807718/article/details/51325431



杂记
=
1. docker容器的cpu和memory是可以调整集群资源参数的，有limit设置；
2. Dockerfile里声明volume的目录是对于任何的volumes命令参数不可变的，会自动生成一个挂载点，目的是不让host直接修改，作为一个数据容器为其他镜像提供volume挂载服务。【--volumes-from】
https://docs.docker.com/engine/reference/builder/#/volume
http://www.cnblogs.com/51kata/p/5266626.html


docker swarm
=

#### 1. 启动manager，初始化集群
manager机器上执行：
```
[root@controller wil]# docker swarm init --advertise-addr 10.1.5.129
Swarm initialized: current node (efpood09fyouiyib1f78y5oa6) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-50wivhmqel8u7pnorhfpu5a8qrfvged69b6r8v06jzscsysds2-3yllw8v8deku690gd8a8pfio1 \
    10.1.5.129:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
[root@controller wil]# docker info
.....
.....
Swarm: active
 NodeID: efpood09fyouiyib1f78y5oa6
 Is Manager: true
 ClusterID: 4wnicl4f2vvtlpmpa2pyj28ji
 Managers: 1
 Nodes: 1
 Orchestration:
  Task History Retention Limit: 5
 Raft:
  Snapshot Interval: 10000
  Heartbeat Tick: 1
  Election Tick: 3
 Dispatcher:
  Heartbeat Period: 5 seconds
 CA Configuration:
  Expiry Duration: 3 months
 Node Address: 10.1.5.129
Runtimes: runc
.....
.....

```

可以看到初始化后，docker info里显示了swarm相关的信息。

查看当前swarm集群中的机器：
```
[root@controller wil]#  docker node ls
ID                           HOSTNAME    STATUS  AVAILABILITY  MANAGER STATUS
efpood09fyouiyib1f78y5oa6 *  controller  Ready   Active        Leader

```
\* 代表的是当前机器

#### 2.添加node

可以在manager节点上确认一下加入集群需要的token：
```
[root@controller wil]# docker swarm join-token worker
To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-50wivhmqel8u7pnorhfpu5a8qrfvged69b6r8v06jzscsysds2-3yllw8v8deku690gd8a8pfio1 \
    10.1.5.129:2377

```
然后分别在dnode1、dnode2上运行此命令：
```
[root@dnode2 willdjango]# docker swarm join \
>     --token SWMTKN-1-50wivhmqel8u7pnorhfpu5a8qrfvged69b6r8v06jzscsysds2-3yllw8v8deku690gd8a8pfio1 \
>     10.1.5.129:2377

This node joined a swarm as a worker.
```


```
[root@dnode1 server]# docker swarm join \
>     --token SWMTKN-1-50wivhmqel8u7pnorhfpu5a8qrfvged69b6r8v06jzscsysds2-3yllw8v8deku690gd8a8pfio1 \
>     10.1.5.129:2377

This node joined a swarm as a worker.
```

再查看当前集群node状态, 此命令只能在manager所在的节点上执行，worker节点没有权限：
```
[root@controller wil]# docker node ls
ID                           HOSTNAME    STATUS  AVAILABILITY  MANAGER STATUS
0c67km60csfft6paxdsodp8ss    dnode2      Ready   Active        
759vbg6zxxzffj4i3pw21249y    dnode1      Ready   Active        
efpood09fyouiyib1f78y5oa6 *  controller  Ready   Active        Leader

```

#### 3. 部署服务

创建一个service：
```
[root@controller wil]# docker service create --replicas 1 --name resourcemanager_swarm willcup/yarn:0.1 /bin/bash
0nnc9cpf1ebhbw6ixbro9q8wn
[root@controller wil]# docker service ls
ID            NAME                   REPLICAS  IMAGE             COMMAND
0nnc9cpf1ebh  resourcemanager_swarm  0/1       willcup/yarn:0.1  /bin/bash
```

查看这个service的元信息：
```
[root@controller wil]# docker service inspect --pretty resourcemanager_swarm
ID:		0nnc9cpf1ebhbw6ixbro9q8wn
Name:		resourcemanager_swarm
Mode:		Replicated
 Replicas:	1
Placement:
UpdateConfig:
 Parallelism:	1
 On failure:	pause
ContainerSpec:
 Image:		willcup/yarn:0.1
 Args:		/bin/bash
Resources:

```

查看这个service的进程所在：
```
[root@controller wil]# docker service ps resourcemanager_swarm
ID                         NAME                         IMAGE             NODE        DESIRED STATE  CURRENT STATE            ERROR
dss32jnwz7aaxtqemvxpke4gg  resourcemanager_swarm.1      willcup/yarn:0.1  dnode2      Running        Preparing 5 minutes ago  
1az4a1ctb9y3io2oswew0zab9   \_ resourcemanager_swarm.1  willcup/yarn:0.1  controller  Shutdown       Complete 2 minutes ago   

```

像这个我们的例子中就是，在controller中启动未成功，然后转到dnode1上运行了。但是两次执行的DESIRED STATE和CURRENT STATE并不吻合。看了一下是我们的manager机器【也就是上面的controller】并不认识dnode2，配置一下hosts。再查看一下：
```
[root@controller wil]# docker service ps resourcemanager_swarm
ID                         NAME                         IMAGE             NODE        DESIRED STATE  CURRENT STATE            ERROR
bvrtqkl8iybt5rr5kqlj6mio2  resourcemanager_swarm.1      willcup/yarn:0.1  controller  Running        Running 10 seconds ago   
dss32jnwz7aaxtqemvxpke4gg   \_ resourcemanager_swarm.1  willcup/yarn:0.1  dnode2      Shutdown       Complete 4 minutes ago   
1az4a1ctb9y3io2oswew0zab9   \_ resourcemanager_swarm.1  willcup/yarn:0.1  controller  Shutdown       Complete 10 minutes ago  
[root@controller wil]# docker service ps resourcemanager_swarm
ID                         NAME                         IMAGE             NODE        DESIRED STATE  CURRENT STATE                ERROR
c2btlmupl7r9ss3dents28928  resourcemanager_swarm.1      willcup/yarn:0.1  controller  Running        Preparing 22 seconds ago     
99hilqo2an723j9k7n2grkf0o   \_ resourcemanager_swarm.1  willcup/yarn:0.1  dnode2      Shutdown       Complete 3 minutes ago       
a16ktljr241g9zn6qluunx84l   \_ resourcemanager_swarm.1  willcup/yarn:0.1  controller  Shutdown       Complete about a minute ago  
c52kek07d8uve9kqjsf3vcgh3   \_ resourcemanager_swarm.1  willcup/yarn:0.1  dnode2      Shutdown       Complete 6 minutes ago       
bvrtqkl8iybt5rr5kqlj6mio2   \_ resourcemanager_swarm.1  willcup/yarn:0.1  controller  Shutdown       Complete 4 minutes ago 

```

总是出问题。。。

还是用官网的吧：
```
[root@controller wil]# docker service create --replicas 1 --name helloworld alpine ping docker.com
2q5p0ahn92aa8be9el23j93yx
[root@controller wil]# docker service ps helloworld
ID                         NAME          IMAGE   NODE        DESIRED STATE  CURRENT STATE            ERROR
dvjcrjztbqkj3x5e8fggraw0f  helloworld.1  alpine  controller  Running        Preparing 6 seconds ago  
[root@controller wil]# docker service ps helloworld
ID                         NAME          IMAGE   NODE        DESIRED STATE  CURRENT STATE           ERROR
dvjcrjztbqkj3x5e8fggraw0f  helloworld.1  alpine  controller  Running        Starting 4 seconds ago  
[root@controller wil]# docker service ps helloworld
ID                         NAME          IMAGE   NODE        DESIRED STATE  CURRENT STATE           ERROR
dvjcrjztbqkj3x5e8fggraw0f  helloworld.1  alpine  controller  Running        Starting 8 seconds ago  
[root@controller wil]# docker service ps helloworld
ID                         NAME          IMAGE   NODE        DESIRED STATE  CURRENT STATE               ERROR
dvjcrjztbqkj3x5e8fggraw0f  helloworld.1  alpine  controller  Running        Running about a minute ago
```

说是在controller上运行着的，我们就到controller机器上ps一下：
```
[root@controller wil]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
5653c87be989        alpine:latest       "ping docker.com"   4 minutes ago       Up 3 minutes                            helloworld.1.dvjcrjztbqkj3x5e8fggraw0f
```


#### 4. 服务扩容

对于后台服务，很多时候我们是部署多个，然后再前面部署负载均衡的。我们可以通过swarm的命令轻松的将我们的服务进行扩容：
```
[root@controller wil]# docker service scale helloworld=5
helloworld scaled to 5
[root@controller wil]# docker service ps helloworld
ID                         NAME          IMAGE   NODE        DESIRED STATE  CURRENT STATE                     ERROR
dvjcrjztbqkj3x5e8fggraw0f  helloworld.1  alpine  controller  Running        Running 5 minutes ago             
5x3rsxhw31w5prpwhvh5bzn8m  helloworld.2  alpine  dnode2      Running        Preparing 2 minutes ago           
1ymr6qzfzvpjc86lwx7c7p938  helloworld.3  alpine  dnode1      Running        Preparing 16 seconds ago          
3h36sr8m8c4f7utei50ss9tav  helloworld.4  alpine  dnode1      Running        Preparing 16 seconds ago          
eneop6ooju630tzpxt8vlftjw  helloworld.5  alpine  controller  Running        Preparing less than a second ago  
[root@controller wil]# docker service ps helloworld
ID                         NAME          IMAGE   NODE        DESIRED STATE  CURRENT STATE               ERROR
dvjcrjztbqkj3x5e8fggraw0f  helloworld.1  alpine  controller  Running        Running 7 minutes ago       
5x3rsxhw31w5prpwhvh5bzn8m  helloworld.2  alpine  dnode2      Running        Running 3 minutes ago       
1ymr6qzfzvpjc86lwx7c7p938  helloworld.3  alpine  dnode1      Running        Running about a minute ago  
3h36sr8m8c4f7utei50ss9tav  helloworld.4  alpine  dnode1      Running        Running about a minute ago  
eneop6ooju630tzpxt8vlftjw  helloworld.5  alpine  controller  Running        Running about a minute ago 
```

可以看到，实现了扩容，自动将container重命名，并且分布到了不同的节点之上。

#### 5. 删除服务

```
[root@controller wil]# docker service rm helloworld
helloworld
[root@controller wil]# docker service ps helloworld
Error: No such service: helloworld
```

虽然service很快就被删除了，但是其实container是需要一些时间才能清理干净的。开始的时候docker ps还能看到部分，稍等一下就没有了：
```
[root@controller wil]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                     PORTS               NAMES
5653c87be989        alpine:latest       "ping docker.com"        10 minutes ago      Removal In Progress                            helloworld.1.dvjcrjztbqkj3x5e8fggraw0f
fb3c81306678        a6a1f5eb98f2        "bash /entrypoint.sh "   3 hours ago         Exited (1) 3 hours ago                         composetest_nodemanager_1
c191fc0db545        a6a1f5eb98f2        "bash /entrypoint.sh "   19 hours ago        Exited (137) 3 hours ago                       composetest_resourcemanager_1
[root@controller wil]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                     PORTS               NAMES
fb3c81306678        a6a1f5eb98f2        "bash /entrypoint.sh "   3 hours ago         Exited (1) 3 hours ago                         composetest_nodemanager_1
c191fc0db545        a6a1f5eb98f2        "bash /entrypoint.sh "   19 hours ago        Exited (137) 3 hours ago                       composetest_resourcemanager_1
```

#### 5. 节点维护
有的时候我们要临时维护几台节点，就需要把这个节点标记下线，为了不影响服务质量，需要把正在运行的服务转移到其他节点上。swarm会自动为我们做这些工作，我们只需要标记某个节点为drain就可以了。
```
[root@controller wil]# docker service ps helloworld
ID                         NAME          IMAGE   NODE        DESIRED STATE  CURRENT STATE               ERROR
ciw4f1ibuwuqjv2hvn6s1mqs9  helloworld.1  alpine  dnode1      Running        Running about a minute ago  
0ezba4uegtfmajlkw6kbw66q5  helloworld.2  alpine  dnode2      Running        Running 4 minutes ago       
dnb7d4cjwes3ubywn6iv2vpiu  helloworld.3  alpine  controller  Running        Running about a minute ago  
dyidqdhh7n04m0sk3k0fy74yp  helloworld.4  alpine  controller  Running        Running about a minute ago  
66lmmu30fywqqey7lpirz5o6s  helloworld.5  alpine  dnode1      Running        Running about a minute ago  
[root@controller wil]# docker node update --availability drain dnode1
dnode1
[root@controller wil]# docker node ls
ID                           HOSTNAME    STATUS  AVAILABILITY  MANAGER STATUS
0c67km60csfft6paxdsodp8ss    dnode2      Ready   Active        
759vbg6zxxzffj4i3pw21249y    dnode1      Ready   Drain         
efpood09fyouiyib1f78y5oa6 *  controller  Ready   Active        Leader
[root@controller wil]# docker service ps helloworld
ID                         NAME              IMAGE   NODE        DESIRED STATE  CURRENT STATE               ERROR
9sxv9bi0j7yy341rtp6hkrm0e  helloworld.1      alpine  dnode2      Ready          Preparing 3 minutes ago     
ciw4f1ibuwuqjv2hvn6s1mqs9   \_ helloworld.1  alpine  dnode1      Shutdown       Running 2 minutes ago       
0ezba4uegtfmajlkw6kbw66q5  helloworld.2      alpine  dnode2      Running        Running 4 minutes ago       
dnb7d4cjwes3ubywn6iv2vpiu  helloworld.3      alpine  controller  Running        Running about a minute ago  
dyidqdhh7n04m0sk3k0fy74yp  helloworld.4      alpine  controller  Running        Running about a minute ago  
8ozxlg29d4ynko3plauupuuu5  helloworld.5      alpine  dnode2      Ready          Preparing 3 minutes ago     
66lmmu30fywqqey7lpirz5o6s   \_ helloworld.5  alpine  dnode1      Shutdown       Running 2 minutes ago       
[root@controller wil]# docker service ps helloworld
ID                         NAME              IMAGE   NODE        DESIRED STATE  CURRENT STATE               ERROR
9sxv9bi0j7yy341rtp6hkrm0e  helloworld.1      alpine  dnode2      Running        Preparing 3 minutes ago     
ciw4f1ibuwuqjv2hvn6s1mqs9   \_ helloworld.1  alpine  dnode1      Shutdown       Shutdown 19 seconds ago     
0ezba4uegtfmajlkw6kbw66q5  helloworld.2      alpine  dnode2      Running        Running 5 minutes ago       
dnb7d4cjwes3ubywn6iv2vpiu  helloworld.3      alpine  controller  Running        Running about a minute ago  
dyidqdhh7n04m0sk3k0fy74yp  helloworld.4      alpine  controller  Running        Running about a minute ago  
8ozxlg29d4ynko3plauupuuu5  helloworld.5      alpine  dnode2      Running        Preparing 3 minutes ago     
66lmmu30fywqqey7lpirz5o6s   \_ helloworld.5  alpine  dnode1      Shutdown       Shutdown 20 seconds ago
```

当维护完成之后，再把node状态更新回来：
```
[root@controller wil]# docker node update --availability active dnode1
dnode1
[root@controller wil]# docker service ps helloworld
ID                         NAME              IMAGE   NODE        DESIRED STATE  CURRENT STATE           ERROR
9sxv9bi0j7yy341rtp6hkrm0e  helloworld.1      alpine  dnode2      Running        Running 4 minutes ago   
ciw4f1ibuwuqjv2hvn6s1mqs9   \_ helloworld.1  alpine  dnode1      Shutdown       Shutdown 2 minutes ago  
0ezba4uegtfmajlkw6kbw66q5  helloworld.2      alpine  dnode2      Running        Running 6 minutes ago   
dnb7d4cjwes3ubywn6iv2vpiu  helloworld.3      alpine  controller  Running        Running 3 minutes ago   
dyidqdhh7n04m0sk3k0fy74yp  helloworld.4      alpine  controller  Running        Running 3 minutes ago   
8ozxlg29d4ynko3plauupuuu5  helloworld.5      alpine  dnode2      Running        Running 4 minutes ago   
66lmmu30fywqqey7lpirz5o6s   \_ helloworld.5  alpine  dnode1      Shutdown       Shutdown 2 minutes ago  
[root@controller wil]# docker node ls
ID                           HOSTNAME    STATUS  AVAILABILITY  MANAGER STATUS
0c67km60csfft6paxdsodp8ss    dnode2      Ready   Active        
759vbg6zxxzffj4i3pw21249y    dnode1      Ready   Active        
efpood09fyouiyib1f78y5oa6 *  controller  Ready   Active        Leader
```

但是服务如果不出问题的话，不会自动迁移回dnode1了。

通常我们可以让manager节点是drain的，以保证它负载低，集群能正常运行。

#### 6. 路由

所有的swarm节点都加入了一个ingress的routing mesh。

```
[root@controller wil]# docker network ls
NETWORK ID          NAME                  DRIVER              SCOPE
6461c2e9e64e        bridge                bridge              local               
5d5d0be2f5a6        docker_gwbridge       bridge              local               
e2bbfd5c0b62        host                  host                local               
0r5dk7ja8vw7        ingress               overlay             swarm               
01c12cd7bba7        none                  null                local 
```
routing mesh会把所有对于public port的请求转发到所有对应service的节点的container上。

先搞一个有public接口的service:
```
[root@controller wil]# docker service create \
>   --name my-web \
>   --publish 8080:80 \
>   --replicas 2 \
>   nginx
2lsi4py6pbcu96ttnjyqj7u4i
[root@controller wil]# docker service ps my-web
ID                         NAME      IMAGE  NODE    DESIRED STATE  CURRENT STATE             ERROR
84xjxx4kth0idnzuob4186cn7  my-web.1  nginx  dnode1  Running        Preparing 20 seconds ago  
0fqk2zwyp9u2ohi9iu91qli50  my-web.2  nginx  dnode2  Running        Preparing 3 minutes ago
```
这个命令会在所有的node上都开放8080端口，swarm的load balancer会把对于任何一个node的8080接口的请求路由给active的container【这里就是dnode1和dnode2】。

也可以为正在运行的服务添加开放的端口映射：
```
$ docker service update \
  --publish-add <PUBLISHED-PORT>:<TARGET-PORT> \
  <SERVICE>
```

查看某service开放的端口：
```
[root@controller wil]#  docker service inspect --format="{{json .Endpoint.Spec.Ports}}" my-web
[{"Protocol":"tcp","TargetPort":80,"PublishedPort":8080}]

```

单独指定开放tcp/udp端口，默认就是TCP：
```
$ docker service create --name dns-cache -p 53:53/tcp dns-cache
$ docker service create --name dns-cache -p 53:53/udp dns-cache
```

因为目前是有三个node的8080都可以提供服务，所以我们也可以在他们之外再加个HAProxy做负载均衡。
其实如果swarm自己的调度器能够均匀分发请求给不同的container的话，也没有必要弄外部的LB了。



#### 7. 其他

另外，创建服务的时候和compose类似，也是有很多参数可以指定的
```
[root@controller wil]# docker service create --help

Usage:	docker service create [OPTIONS] IMAGE [COMMAND] [ARG...]

Create a new service

Options:
      --constraint value               Placement constraints (default [])
      --container-label value          Container labels (default [])
      --endpoint-mode string           Endpoint mode (vip or dnsrr)
  -e, --env value                      Set environment variables (default [])
      --help                           Print usage
  -l, --label value                    Service labels (default [])
      --limit-cpu value                Limit CPUs (default 0.000)
      --limit-memory value             Limit Memory (default 0 B)
      --log-driver string              Logging driver for service
      --log-opt value                  Logging driver options (default [])
      --mode string                    Service mode (replicated or global) (default "replicated")
      --mount value                    Attach a mount to the service
      --name string                    Service name
      --network value                  Network attachments (default [])
  -p, --publish value                  Publish a port as a node port (default [])
      --replicas value                 Number of tasks (default none)
      --reserve-cpu value              Reserve CPUs (default 0.000)
      --reserve-memory value           Reserve Memory (default 0 B)
      --restart-condition string       Restart when condition is met (none, on-failure, or any)
      --restart-delay value            Delay between restart attempts (default none)
      --restart-max-attempts value     Maximum number of restarts before giving up (default none)
      --restart-window value           Window used to evaluate the restart policy (default none)
      --stop-grace-period value        Time to wait before force killing a container (default none)
      --update-delay duration          Delay between updates
      --update-failure-action string   Action on update failure (pause|continue) (default "pause")
      --update-parallelism uint        Maximum number of tasks updated simultaneously (0 to update all at once) (default 1)
  -u, --user string                    Username or UID
      --with-registry-auth             Send registry authentication details to swarm agents
  -w, --workdir string                 Working directory inside the container
```

需要注意的是：慎重使用mount。原因有以下三点：
 - 如果mount到host path上，那么需要每个node都有同样的path和文件。
 - 需要确保所有node上的这个文件一直是可用的状态
 - 这种方式完全就是反设计的。因为不能保证生产和测试的mount文件是一致的，从而导致运行出问题。

参考：https://docs.docker.com/engine/swarm/services/


可以通过demote和promote改变node的角色。

**重大发现**： docker是有python的API的，可以用API管理集群： https://docker-py.readthedocs.io/en/stable/services.html。

#### 坑

1.12版本的swarm模式里并不能获取已经失败了的service的container的日志：

```
[root@controller wil]# docker service ps yarn
ID                         NAME        IMAGE             NODE        DESIRED STATE  CURRENT STATE              ERROR
c317x0ovq833c6g83hi4iwzz4  yarn.1      willcup/yarn:0.1  controller  Running        Running 2 seconds ago      
2w5k9c4efh8h188ompgnfvh57   \_ yarn.1  willcup/yarn:0.1  dnode1      Shutdown       Failed about a minute ago  "task: non-zero exit (127)"
4jvdu7nz8kxvnd9wa9qzmy1pw   \_ yarn.1  willcup/yarn:0.1  controller  Shutdown       Failed about a minute ago  "task: non-zero exit (127)"
5yu69ugxmndhfdetvd9kthel2   \_ yarn.1  willcup/yarn:0.1  dnode1      Shutdown       Failed 3 minutes ago       "task: non-zero exit (127)"
7qyx81ptxouv6883h1o1pbiti   \_ yarn.1  willcup/yarn:0.1  controller  Shutdown       Failed 5 minutes ago       "task: non-zero exit (127)"
```

上面失败的就都找不到了。而且rm service以后，那些container都会被自动清理掉。

对于某些功能支持的不完善.

参考：
- http://www.jianshu.com/p/3efee6927833
- https://github.com/docker/docker/issues/24812
