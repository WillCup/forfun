
title: centos7-安装k8s
date: 2017-03-30 18:33:52
tags: [youdaonote]
---

代理设置
=
先弄一下代理，弄一下hosts文件，参考： https://github.com/racaljk/hosts

安装
=
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://yum.kubernetes.io/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
       https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
setenforce 0
yum install -y docker kubelet kubeadm kubectl kubernetes-cni
systemctl enable docker && systemctl start docker
systemctl enable kubelet && systemctl start kubelet
```

要是安装过docker了，就不用再弄了，最好保持一致。


初始化
=
controller组件运行在master上，它包含etcd和API server。

```
root@controller kubernetes]# kubeadm init
[kubeadm] WARNING: kubeadm is in beta, please do not use it for production clusters.
[init] Using Kubernetes version: v1.6.0
[init] Using Authorization mode: RBAC
[preflight] Running pre-flight checks
[preflight] WARNING: docker version is greater than the most recently validated version. Docker version: 17.03.1-ce. Max validated version: 1.12
[certificates] Generated CA certificate and key.
[certificates] Generated API server certificate and key.
[certificates] API Server serving cert is signed for DNS names [controller kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.1.5.129]
[certificates] Generated API server kubelet client certificate and key.
[certificates] Generated service account token signing key and public key.
[certificates] Generated front-proxy CA certificate and key.
[certificates] Generated front-proxy client certificate and key.
[certificates] Valid certificates and keys now exist in "/etc/kubernetes/pki"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/controller-manager.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/scheduler.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/admin.conf"
[apiclient] Created API client, waiting for the control plane to become ready
```
这个命令会自动探查网络接口，告诉默认的网关master所在的地址。要是想指定额外的master地址，使用参数--api-advertise-addresses <ip-address>。

上面的命令也会自动下载所有的必须的镜像：
```
gcr.io/google_containers/kube-proxy-amd64                v1.5.3         
gcr.io/google_containers/kube-controller-manager-amd64   v1.5.3         
gcr.io/google_containers/kube-scheduler-amd64            v1.5.3         
gcr.io/google_containers/kube-apiserver-amd64            v1.5.3         
gcr.io/google_containers/etcd-amd64                      3.0.14-kubeadm 
gcr.io/google_containers/kube-discovery-amd64            1.0            
gcr.io/google_containers/pause-amd64                     3.0 
```

中间提示我/proc/sys/net/bridge/bridge-nf-call-iptables的内容不是1，看了一下是0，修改了之后可以了。

注意 kubeadm init 只能在你确定的master上执行一次。
