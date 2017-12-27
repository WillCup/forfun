
title: centos7-微服务相关环境部署
date: 2017-09-30 16:50:34
tags: [youdaonote]
---

#### 系统环境

```
hostnamectl set-hostname xx.will.com --static

yum install -y wget git vim ntpdate

timedatectl set-timezone Asia/Shanghai

/usr/sbin/ntpdate us.pool.ntp.org


```



#### docker安装
```
tee /etc/yum.repos.d/docker.repo <<-'EOF'
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF

yum install docker-engine -y

# 获取最新版本的docker
# curl -fsSL https://get.docker.com/ | sh



systemctl enable docker.service
systemctl restart docker


docker version
```
ubuntu
```
# step 1: 安装必要的一些系统工具
sudo apt-get update
sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common
# step 2: 安装GPG证书
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
# Step 3: 写入软件源信息
sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
# Step 4: 更新并安装 Docker-CE
sudo apt-get -y update
sudo apt-get -y install docker-ce

# 安装指定版本的Docker-CE:
# Step 1: 查找Docker-CE的版本:
# apt-cache madison docker-ce
#   docker-ce | 17.03.1~ce-0~ubuntu-xenial | http://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial/stable amd64 Packages
#   docker-ce | 17.03.0~ce-0~ubuntu-xenial | http://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial/stable amd64 Packages
# Step 2: 安装指定版本的Docker-CE: (VERSION 例如上面的 17.03.1~ce-0~ubuntu-xenial)
# sudo apt-get -y install docker-ce=[VERSION]
```

centos7
```
# step 1: 安装必要的一些系统工具
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
# Step 2: 添加软件源信息
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# Step 3: 更新并安装 Docker-CE
sudo yum makecache fast
sudo yum -y install docker-ce
# Step 4: 开启Docker服务
sudo service docker start

# 注意：
# 官方软件源默认启用了最新的软件，您可以通过编辑软件源的方式获取各个版本的软件包。例如官方并没有将测试版本的软件源置为可用，你可以通过以下方式开启。同理可以开启各种测试版本等。
# vim /etc/yum.repos.d/docker-ee.repo
#   将 [docker-ce-test] 下方的 enabled=0 修改为 enabled=1
#
# 安装指定版本的Docker-CE:
# Step 1: 查找Docker-CE的版本:
# yum list docker-ce.x86_64 --showduplicates | sort -r
#   Loading mirror speeds from cached hostfile
#   Loaded plugins: branch, fastestmirror, langpacks
#   docker-ce.x86_64            17.03.1.ce-1.el7.centos            docker-ce-stable
#   docker-ce.x86_64            17.03.1.ce-1.el7.centos            @docker-ce-stable
#   docker-ce.x86_64            17.03.0.ce-1.el7.centos            docker-ce-stable
#   Available Packages
# Step2 : 安装指定版本的Docker-CE: (VERSION 例如上面的 17.03.0.ce.1-1.el7.centos)
# sudo yum -y install docker-ce-[VERSION]
```

https://yq.aliyun.com/articles/110806

配置文件 /etc/docker/daemon.json
```
{
  "graph": "/server/dspace",
  "insecure-registries": ["10.1.5.129"],
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn","http://3a35aff6.m.daocloud.io","http://ethanzhu.m.tenxcloud.net"],
  "hosts": ["tcp://0.0.0.0:4243","unix:///var/run/docker.sock"]
}

```

```
{
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn","http://3a35aff6.m.daocloud.io","http://ethanzhu.m.tenxcloud.net"],
  "metrics-addr" : "10.2.19.112:9323",
  "experimental" : true,
  "dns": ["10.2.19.112"],
  "dns-search": ["will.com"]
}

```



#### mesos + marathon

```
rpm -Uvh http://repos.mesosphere.io/el/7/noarch/RPMS/mesosphere-el-repo-7-1.noarch.rpm
yum -y install mesos marathon mesosphere-zookeeper

```


master环境脚本

mesos_master_start.sh
```bash
export MESOS_cluster=will
#export MESOS_zk=zk://10.1.5.65:2181/wesos
export MESOS_zk=zk://datanode01.will.com:2181,datanode02.will.com:2181,datanode04.will.com:2181,servicenode05.will.com:2181,servicenode06.will.com:2181/wesos
export MESOS_work_dir=/var/lib/mesos/master
export MESOS_quorum=1
export MESOS_log_dir=/server/mesos_log
export MESOS_zk_session_timeout=20secs
export MESOS_logging_level=WARNING
#export MESOS_log_dir=/var/log/mesos_master
export MESOS_hostname=10.2.19.124
```
启动脚本：
```
. /server/mesos_master_start.sh && nohup /usr/sbin/mesos-master &> mesos_master.out&
```


mesos_agent_env.sh
```
export MESOS_executor_registration_timeout=5mins
export MESOS_containerizers=docker,mesos
export MESOS_isolation=cgroups/cpu,cgroups/mem
export MESOS_work_dir=/server/mesos/agent
export MESOS_master=zk://datanode01.will.com:2181,datanode02.will.com:2181,datanode04.will.com:2181/wesos_online
export MESOS_resources='ports:[10000-60000]'
```

启动脚本
```
. /server/mesos_agent_env.sh && nohup mesos-agent &> mesos_agent.out&
```


marathon
```
nohup marathon  --master zk://datanode01.will.com:2181,datanode02.will.com:2181,datanode04.will.com:2181/wesos_online  --zk zk://datanode01.will.com:2181,datanode02.will.com:2181,datanode04.will.com:2181/mt_online  --zk_timeout 15000 &> marathon.out&
```
