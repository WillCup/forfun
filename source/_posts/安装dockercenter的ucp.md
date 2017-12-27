
title: 安装dockercenter的ucp
date: 2017-02-17 18:53:36
tags: [youdaonote]
---




```
[root@controller wil]# docker pull docker/ucp:2.1.0
2.1.0: Pulling from docker/ucp
b7f33cc0b48e: Pulling fs layer 
8257d57cc15c: Pulling fs layer 
b7f33cc0b48e: Pull complete 
8257d57cc15c: Extracting [==================================================>] 348.8 kB/348.8 kB
8257d57cc15c: Pull complete 
f691f14109b2: Pull complete 
Digest: sha256:fd43a6560b6f5731d0e383a7b5fe6da91cea11bc6b3fd65afe08f5e4a97dbc90
Status: Downloaded newer image for docker/ucp:2.1.0
[root@controller wil]# docker run --rm -it --name ucp   -v /var/run/docker.sock:/var/run/docker.sock   docker/ucp:2.1.0 install   --host-address 10.1.5.129   --interactive

INFO[0000] Verifying your system is compatible with UCP 
INFO[0000] Your engine version 1.13.1, build 092cba3 (3.10.0-514.2.2.el7.x86_64) is compatible 
WARN[0003] Your system uses devicemapper.  We can not accurately detect available storage space.  Please make sure you have at least 3.00 GB available in /server/dspace 
Admin Username: unable to scan value: unexpected newline
FATA[0005] unable to get admin username, giving up    
[root@controller wil]# docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username (willcup): 
Password: 
Login Succeeded
[root@controller wil]# docker run --rm -it --name ucp   -v /var/run/docker.sock:/var/run/docker.sock   docker/ucp:2.1.0 install   --host-address 10.1.5.129   --interactive
INFO[0000] Verifying your system is compatible with UCP 
INFO[0000] Your engine version 1.13.1, build 092cba3 (3.10.0-514.2.2.el7.x86_64) is compatible 
WARN[0001] Your system uses devicemapper.  We can not accurately detect available storage space.  Please make sure you have at least 3.00 GB available in /server/dspace 
Admin Username: willcup
Admin Password: 
Confirm Admin Password: 
INFO[0021] Pulling required images... (this may take a while) 

```

可以看到如果不先docker login的话，启动时候就报错找不到admin name。然而启动的时候又让我们自己定义admin name和passwd。醉了。

打开debug：
```
[root@controller wil]# docker run --rm -it --name ucp   -v /var/run/docker.sock:/var/run/docker.sock   docker/ucp:2.1.0 install   --host-address 10.1.5.129   -D --interactive
DEBU[0000] New UCP Instance ID will be "XZNK:CP6T:DHHE:XJZA:OXOM:CLS6:RIRW:SAVZ:4JIL:G3Q4:DJ7F:L7P3" 
INFO[0000] Verifying your system is compatible with UCP 
DEBU[0000] Checking for compatible kernel version       
DEBU[0000] Kernel version 3.10.0-514.2.2.el7.x86_64 is compatible 
DEBU[0000] Checking for compatible engine version       
INFO[0000] Your engine version 1.13.1, build 092cba3 (3.10.0-514.2.2.el7.x86_64) is compatible 
DEBU[0000] Container ucp-phase2 not present: Error: No such container: ucp-phase2 
DEBU[0000] Detected container ucp                       
DEBU[0000] Container ucp-controller not present: Error: No such container: ucp-controller 
DEBU[0000] Container ucp-swarm-manager not present: Error: No such container: ucp-swarm-manager 
DEBU[0000] Container ucp-swarm-join not present: Error: No such container: ucp-swarm-join 
DEBU[0000] Container ucp-kv not present: Error: No such container: ucp-kv 
DEBU[0000] Container ucp-proxy not present: Error: No such container: ucp-proxy 
DEBU[0000] Container ucp-client-root-ca not present: Error: No such container: ucp-client-root-ca 
DEBU[0000] Container ucp-cluster-root-ca not present: Error: No such container: ucp-cluster-root-ca 
DEBU[0000] Container ucp-auth-store not present: Error: No such container: ucp-auth-store 
DEBU[0000] Container ucp-auth-api not present: Error: No such container: ucp-auth-api 
DEBU[0000] Container ucp-auth-worker not present: Error: No such container: ucp-auth-worker 
DEBU[0000] Container ucp-kv-backup not present: Error: No such container: ucp-kv-backup 
DEBU[0000] Container ucp-kv-restore not present: Error: No such container: ucp-kv-restore 
DEBU[0000] Container ucp-metrics not present: Error: No such container: ucp-metrics 
DEBU[0000] Validating base system meets minimum requirements 
DEBU[0000] Your system meets minimum memory requirements:  3.88 GB >= 2.00 GB 
WARN[0000] Your system uses devicemapper.  We can not accurately detect available storage space.  Please make sure you have at least 3.00 GB available in /server/dspace 
Admin Username: will
Admin Password: 
Confirm Admin Password: 
DEBU[0207] Checking for images                          
INFO[0207] All required images are present              
DEBU[0208] Local Name: controller                       
WARN[0208] None of the hostnames we'll be using in the UCP certificates [controller 127.0.0.1 172.17.0.1] contain a domain component.  Your generated certs may fail TLS validation unless you only use one of these shortnames or IPs to connect.  You can use the --san flag to add more aliases 

You may enter additional aliases (SANs) now or press enter to proceed with the above list.
Additional aliases: willalias
DEBU[0222] User entered: willalias
                     
DEBU[0222] Hostnames: [controller 127.0.0.1 172.17.0.1 willalias] 
DEBU[0245] Launching phase 2 with: [install --host-address 10.1.5.129 -D --interactive] (6f2f352ea35e614de1005fe6d93c9843c3f972dc1e7cf5191ea9ed7a0e46fb25) 
DEBU[0000] Beginning phase 2 install for instance XZNK:CP6T:DHHE:XJZA:OXOM:CLS6:RIRW:SAVZ:4JIL:G3Q4:DJ7F:L7P3 
DEBU[0000] Checking for compatible kernel version       
DEBU[0000] Kernel version 3.10.0-514.2.2.el7.x86_64 is compatible 
DEBU[0000] Checking for compatible engine version       
DEBU[0000] Detected container ucp-phase2                
DEBU[0001] Container ucp-controller not present: Error: No such container: ucp-controller 
DEBU[0001] Container ucp-swarm-manager not present: Error: No such container: ucp-swarm-manager 
DEBU[0001] Container ucp-swarm-join not present: Error: No such container: ucp-swarm-join 
DEBU[0001] Container ucp-kv not present: Error: No such container: ucp-kv 
DEBU[0001] Container ucp-proxy not present: Error: No such container: ucp-proxy 
DEBU[0001] Container ucp-client-root-ca not present: Error: No such container: ucp-client-root-ca 
DEBU[0001] Container ucp-cluster-root-ca not present: Error: No such container: ucp-cluster-root-ca 
DEBU[0001] Container ucp-auth-store not present: Error: No such container: ucp-auth-store 
DEBU[0001] Container ucp-auth-api not present: Error: No such container: ucp-auth-api 
DEBU[0001] Container ucp-auth-worker not present: Error: No such container: ucp-auth-worker 
DEBU[0001] Container ucp-kv-backup not present: Error: No such container: ucp-kv-backup 
DEBU[0001] Container ucp-kv-restore not present: Error: No such container: ucp-kv-restore 
DEBU[0001] Container ucp-metrics not present: Error: No such container: ucp-metrics 
DEBU[0001] Local Name: controller                       
DEBU[0001] Hostnames: [controller 127.0.0.1 172.17.0.1 willalias] 
DEBU[0001] This node is known as efpood09fyouiyib1f78y5oa6 
DEBU[0001] Got node IP 10.1.5.129 from swarm            
DEBU[0001] EnginePortCheck: port 2375, prpl http, err Get http://10.1.5.129:2375/info: dial tcp 10.1.5.129:2375: getsockopt: connection refused 
DEBU[0001] EnginePortCheck: port 2375, prpl https, err Get https://10.1.5.129:2375/info: dial tcp 10.1.5.129:2375: getsockopt: connection refused 
DEBU[0001] EnginePortCheck: port 2376, prpl http, err Get http://10.1.5.129:2376/info: dial tcp 10.1.5.129:2376: getsockopt: connection refused 
DEBU[0001] EnginePortCheck: port 2376, prpl https, err Get https://10.1.5.129:2376/info: dial tcp 10.1.5.129:2376: getsockopt: connection refused 
DEBU[0001] Checking for available and accessible port 12387 
DEBU[0001] Checking for available and accessible port 12380 
DEBU[0001] Checking for available and accessible port 12381 
DEBU[0001] Checking for available and accessible port 443 
DEBU[0001] Checking for available and accessible port 12382 
DEBU[0001] Checking for available and accessible port 2376 
DEBU[0001] Checking for available and accessible port 12383 
DEBU[0001] Checking for available and accessible port 12376 
DEBU[0001] Checking for available and accessible port 12384 
DEBU[0001] Checking for available and accessible port 4789 
DEBU[0001] Checking for available and accessible port 12385 
DEBU[0001] Checking for available and accessible port 12379 
DEBU[0001] Checking for available and accessible port 12386 
DEBU[0038] Checking for liveness of http://10.1.5.129:4789/ 
DEBU[0039] Connected to http://10.1.5.129:4789/         
DEBU[0063] Checking for liveness of http://10.1.5.129:12387/ 
DEBU[0063] Connected to http://10.1.5.129:12387/        
DEBU[0089] Checking for liveness of http://10.1.5.129:12382/ 
DEBU[0089] Connected to http://10.1.5.129:12382/        
DEBU[0114] Checking for liveness of http://10.1.5.129:12384/ 
DEBU[0114] Connected to http://10.1.5.129:12384/        
DEBU[0126] Checking for liveness of http://10.1.5.129:12376/ 
DEBU[0126] Connected to http://10.1.5.129:12376/        
DEBU[0158] Checking for liveness of http://10.1.5.129:12383/ 
DEBU[0161] Connected to http://10.1.5.129:12383/        
DEBU[0183] Checking for liveness of http://10.1.5.129:443/ 
DEBU[0185] Connected to http://10.1.5.129:443/          
DEBU[0199] Checking for liveness of http://10.1.5.129:2376/ 
DEBU[0201] Connected to http://10.1.5.129:2376/         
DEBU[0235] Checking for liveness of http://10.1.5.129:12386/ 
DEBU[0240] Connected to http://10.1.5.129:12386/        
DEBU[0260] Checking for liveness of http://10.1.5.129:12381/ 
DEBU[0260] Connected to http://10.1.5.129:12381/        
DEBU[0271] Checking for liveness of http://10.1.5.129:12385/ 
DEBU[0271] Connected to http://10.1.5.129:12385/        
DEBU[0295] Checking for liveness of http://10.1.5.129:12379/ 
DEBU[0295] Connected to http://10.1.5.129:12379/        
DEBU[0321] Checking for liveness of http://10.1.5.129:12380/ 
DEBU[0321] Connected to http://10.1.5.129:12380/        
DEBU[0429] All ports are open and available             
DEBU[0429] Purging old state from UCP volumes mounted in /var/lib/docker/ucp 
INFO[0429] Establishing mutual Cluster Root CA with Swarm 
INFO[0429] Installing UCP with host address 10.1.5.129 - If this is incorrect, please specify an alternative address with the '--host-address' flag 
INFO[0429] Generating UCP Client Root CA                
DEBU[0429] Writing /var/lib/docker/ucp/ucp-client-root-ca/key.pem 
INFO[0430] Deploying UCP Service                        
DEBU[0433] Local task container not found, trying again. 
DEBU[0459] Local task container not found, trying again. 
................
................
DEBU[0461] Local task container not found, trying again. 
ERRO[0461] Unable to get local task container for ucp-agent service 
FATA[0461] unable to find local task container 
```

重新仔细阅读一下官方文档，发现有些前置条件没有满足：
```
Docker Engine 1.13.0
Docker Remote API 1.25
Compose 1.9
```

刚才docker remote没有开启，compose的版本也不够。

先升级一下所有docker节点的compose：
```
[root@dnode2 willyarnbase]# curl -L "https://github.com/docker/compose/releases/download/1.11.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   600    0   600    0     0    534      0 --:--:--  0:00:01 --:--:--   534
100 8053k  100 8053k    0     0   403k      0  0:00:19  0:00:19 --:--:--  806k
[root@dnode2 willyarnbase]# docker-compose -v
docker-compose version 1.11.1, build 7c5d5e4
```

再编辑/etc/docker/daemon.json配置文件，添加下面配置，启动docker remote API:
```
"hosts": ["tcp://0.0.0.0:4243","unix:///var/run/docker.sock"]
```

然后访问此docker节点的4243接口，例如：http://10.1.5.129:4243/images/json，测试可以获取到当前节点的image信息，代表已经成功启用了docker remote api。

访问http://10.1.5.129:4243/version，查看相关版本信息：
```
{
"Version": "1.13.1",
"ApiVersion": "1.26",
"MinAPIVersion": "1.12",
"GitCommit": "092cba3",
"GoVersion": "go1.7.5",
"Os": "linux",
"Arch": "amd64",
"KernelVersion": "3.10.0-514.2.2.el7.x86_64",
"BuildTime": "2017-02-08T06:38:28.018621521+00:00"
}
```

版本是1.26，已经满足前面的各个docker组件的版本需求，把这个配置scp给其他的docker节点即可。

参考： https://docs.docker.com/engine/api/v1.26/

再去controller启动ucp：
```
[root@controller wil]# docker run --rm -it --name ucp   -v /var/run/docker.sock:/var/run/docker.sock   docker/ucp:2.1.0 install   --host-address 10.1.5.129  --interactive
INFO[0000] Verifying your system is compatible with UCP 
INFO[0000] Your engine version 1.13.1, build 092cba3 (3.10.0-514.2.2.el7.x86_64) is compatible 
FATA[0019] This swarm is already managed by UCP. Please either uninstall UCP first using the 'uninstall-ucp' operation or upgrade your cluster to a newer version 

```


