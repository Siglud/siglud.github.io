---
layout: post
title:  "Ubuntu本地K8s环境搭建"
date:   2019-05-08 11:54:00 +0800
author: Siglud
categories:
  - DevOps
  - Kubernetes
tags:
  - DevOps
comment: true
share: true
---

机器上本身就装了Docker-CE，这一步就可以略过了，具体流程在这里： https://kubernetes.io/docs/setup/cri/

先安装阿里云的Kubernetes镜像：https://opsx.alibaba.com/mirror （参见Kubernetes相关的帮助文件）

## 安装K8s
安装必要组件

```bash
apt-get install -y kubelet kubeadm kubectl
```

关闭SWAP空间
```bash
sudo swapoff -a
sed -i 's/.\\*swap.\\*/#&/' /etc/fstab
```

因为k8s.gcr.io和gcr.io被墙，只能选择绕一绕，先下载coredns的对应版本，给它打上标

```bash
docker pull coredns/coredns:1.3.1
docker tag coredns/coredns:1.3.1 mirrorgooglecontainers/coredns:1.3.1
 ```

初始化环境
```bash
sudo kubeadm init --kubernetes-version=v1.14.1 --pod-network-cidr=10.244.0.0/16 --image-repository index.docker.io/mirrorgooglecontainers
```

kubeadm的参数解释
```
--apiserver-advertise-address string
API Server将要广播的监听地址。如指定为 `0.0.0.0` 将使用缺省的网卡地址。
--apiserver-bind-port int32     缺省值: 6443
API Server绑定的端口
--apiserver-cert-extra-sans stringSlice
可选的额外提供的证书主题别名（SANs）用于指定API Server的服务器证书。可以是IP地址也可以是DNS名称。
--cert-dir string     缺省值: "/etc/kubernetes/pki"
证书的存储路径。
--config string
kubeadm配置文件的路径。警告：配置文件的功能是实验性的。
--cri-socket string     缺省值: "/var/run/dockershim.sock"
指明要连接的CRI socket文件
--dry-run
不会应用任何改变；只会输出将要执行的操作。
--feature-gates string
键值对的集合，用来控制各种功能的开关。可选项有:
Auditing=true|false (当前为ALPHA状态 - 缺省值=false)
CoreDNS=true|false (缺省值=true)
DynamicKubeletConfig=true|false (当前为BETA状态 - 缺省值=false)
-h, --help
获取init命令的帮助信息
--ignore-preflight-errors stringSlice
忽视检查项错误列表，列表中的每一个检查项如发生错误将被展示输出为警告，而非错误。 例如: 'IsPrivilegedUser,Swap'. 如填写为 'all' 则将忽视所有的检查项错误。
--kubernetes-version string     缺省值: "stable-1"
为control plane选择一个特定的Kubernetes版本。
--node-name string
指定节点的名称。
--pod-network-cidr string
指明pod网络可以使用的IP地址段。 如果设置了这个参数，control plane将会为每一个节点自动分配CIDRs。
--service-cidr string     缺省值: "10.96.0.0/12"
为service的虚拟IP地址另外指定IP地址段
--service-dns-domain string     缺省值: "cluster.local"
为services另外指定域名, 例如： "myorg.internal".
--skip-token-print
不打印出由 `kubeadm init` 命令生成的默认令牌。
--token string
这个令牌用于建立主从节点间的双向受信链接。格式为 [a-z0-9]{6}\.[a-z0-9]{16} - 示例： abcdef.0123456789abcdef
--token-ttl duration     缺省值: 24h0m0s
令牌被自动删除前的可用时长 (示例： 1s, 2m, 3h). 如果设置为 '0', 令牌将永不过期。
```

安装成功之后提示如下：
```
our Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.0.18:6443 --token 3up2p8.28s3ff159mzvaa79 \
    --discovery-token-ca-cert-hash sha256:162610463547f13f16a18e225f4f8d1cb254c3f3caf13c6c90ed41896c42442a
```

按照提示的，开始创建用户空间
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

安装flannel
```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

允许master节点上部署pod
```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

## 安装dashboard

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

不出所料的ImagePullBackOff，describe一下它的需求

```
Failed to pull image "k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1": rpc error: code = Unknown desc = Error response from daemon: Get https://k8s.gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
```

从阿里云上下然后打上标签
```bash
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64:v1.10.1
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64:v1.10.1 k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1
```

把出问题的Pod delete掉，它重启就没问题了

## 安装Helm

安装它的最大障碍还是被墙的问题，本来应该这么装：

```bash
curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh
sh get_helm.sh
```
但是它会需要 kubernetes-helm.storage.googleapis.com 下载文件，很明显下不下来，只能手动安装，去github抓它的[最新版](https://github.com/helm/helm/releases)

```bash
wget https://storage.googleapis.com/kubernetes-helm/helm-v2.13.1-linux-amd64.tar.gz
tar xvf helm-v2.13.1-linux-amd64.tar.gz
sudo cp linux-amd64/helm /usr/local/bin
helm init
```

检查安装是否成功
```bash
kubectl -n kube-system get pods | grep tiller
```

我这里发现它ImagePullBackOff了，说明镜像拉不下来，换成阿里云镜像

```bash
helm init --upgrade -i registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.13.1 --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
```
注意，这个2.13.1的这个版本是PullBackOff之后通过descirbe错误的pod拿到的版本号

为了以绝后患，把tag改一下
```bash
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.13.1 gcr.io/kubernetes-helm/tiller:v2.13.1
```

## 安装istio
参考[官方中文教程](https://istio.io/zh/docs/setup/kubernetes/install/helm/)

首先安装
```bash
curl -L https://git.io/getLatestIstio | sh -
```

通过helm安装istio
```bash
kubectl apply -f install/kubernetes/helm/helm-service-account.yaml
helm init --upgrade --service-account tiller
helm install install/kubernetes/helm/istio-init --name istio-init --namespace istio-system
```

确认一下是否安装了足够的CRD
```bash
kubectl get crds | grep 'istio.io\|certmanager.k8s.io'|wc -l
```
结果应该是53~54个

最后的安装程序
```
helm install install/kubernetes/helm/istio --name istio --namespace istio-system
```

查看一下istio的状况
```bash
helm status istio
```

为 default 名空间开启自动 sidecar 注入
```bash
kubectl label namespace default istio-injection=enabled
kubectl get namespace -L istio-injection
```

安装 Book Info 示例
```
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

删除示例应用
```bash
samples/bookinfo/platform/kube/cleanup.sh
```