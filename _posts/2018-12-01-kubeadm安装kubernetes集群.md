---
layout: post
title: kubeadm安装kubernetes集群
---

kubeadm是kubernetes开发的安装配置工具，用来方便配置集群，简化了安装配置的步骤。

## 安装docker

centos或者fedora使用yum安装

```shell
yum install docker
```

安装之后启动

```shell
systemctl daemon-reload
systemctl enable docker.service
systemctl start docker.service
```

## 安装kubernetes组件

### 添加应用源

将阿里kubernetes应用源加入系统

```shell
vim /etc/yum.repo.d/kubernetes.repo
```

将下面的内容粘贴进repo文件

```shell
[kubernetes]

name=Kubernetes Repo

baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/

gpgcheck=0

enabled=1
```

### 安装master节点

master节点需要kubelet kubeadm 和kubectl

```shell
yum install -y kubelet kubeadm kubectl
```

安装之后启动

```shell
systemctl daemon-reload

systemctl enable kubelet 
```

### 安装node节点

子节点需要安装kubelet和kubeadm

```
yum install -y kubelet kubeadm
```

启动

```shell
systemctl daemon-reload

systemctl enable kubelet 
```

### 准备镜像

查看需要的镜像

```
kubeadm config images list
```

```
k8s.gcr.io/kube-apiserver:v1.12.2
k8s.gcr.io/kube-controller-manager:v1.12.2
k8s.gcr.io/kube-scheduler:v1.12.2
k8s.gcr.io/kube-proxy:v1.12.2
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.2.24
k8s.gcr.io/coredns:1.2.2
```

知道需要这些镜像之后

- 可以翻墙去外网下载这些镜像
- 也可以下载kubernetes官方下载文件，里边会有kube-*的镜像压缩文件，采用docker load加载之后就有了想要的镜像
- 其他的几个镜像可以去阿里云下载同名同版本的镜像然后使用docker tag打成我们需要的标签。

## 启动master节点

主节点执行

```
kubeadm init --kubernetes-version=v1.12.2 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --node-name=master
```

- --kubernetes-version=v1.12.2 这里的版本号需要与你kubeadm和kube*镜像的版本一致。
- --node-name 注册节点的名字

一些配置命令

```shell
mkdir -p $HOME/.kube

cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

chown $(id -u):$(id -g) $HOME/.kube/config

```

配置flannel

```
kubectl apply -fhttps://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

## 启动子节点

子节点上安装完之后执行

```
kubeadm join  . . .
```

这里执行的命令直接复制master节点init之后提示的命令即可，命令中会包含token、证书之类的东西，最后跟上  --node-name 修改子节点注册的名字

## 检查安装情况

在mater节点上执行

```shell
kubectl get node
```

如果节点是notready状态就要检查对应的节点的运行情况

首先检查节点环境，关闭防火墙和swap

```
systemctl stop firewalld
swapoff -a
```

做完这些之后再检查节点是否ready，还可以查看kubelet的日志

journalctl  -exfu  kubelet

## 集群重启

主节点关机之后，再次启动只要保证kubelet启动就行，可以考虑永久关闭防火墙和swap，这样的话master重新启动之后就立马启动了集群。

关闭交换区，永久生效

```shell
echo "vm.swappiness = 0">> /etc/sysctl.conf 
```

禁止开机启动防火墙

```shell
systemctl disable firewalld.service
```

子节点关机之后，再次启动也是要启动kubelet，如果使用systemctl enable kubelet之后，kubelet就会开机启动，不需要手动启动集群。

## 安装原理分析

- kubeadm只是一个安装和配置kubelet的工具，每个节点上实际运行的应用服务只有kubelet。
- kubeadm帮助kubelet配置了各个节点注册和相互访问的token和其他证书。
- kube-apiserver、kube-scheduler、etcd、flannel、kube-controller-manager、kube-coredns、kube-proxy都是作为一个pod提供服务的，kubelet启动的时候会执行manifests文件夹下的yaml文件生成服务。



