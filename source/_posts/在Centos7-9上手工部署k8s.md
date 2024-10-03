---
title: 在Centos7.9上手工部署k8s
date: 2024-10-02 22:09:52
tags:
    - k8s
cover: https://raw.gitmirror.com/ByteQuestor/picture/main/k8s130.png
description: 使用3台Centos7.9手工部署k8s，以加深对k8s的理解
---

# 环境准备

<div style="color: red; font-weight: bold; font-size: 35px;">所有节点执行</div>

## 拉取dockerhub环境

> 本次实验在**新开的虚拟机**上，配置拉取镜像的脚本会清除所有原来的yum源和覆盖一些配置文件
>
> **不可在重要的机器上跑**（或者 注释掉删除的命令 和 把覆盖改为追加）
>
> **在重要的机器上**可以手动添加脚本里的配置内容

[ToDockerHub脚本地址](https://github.com/ByteQuestor/shell/blob/centos7_9/Centos7_9/ToDockerHub.sh)

## 环境架构

> 三台虚拟机，一台`master`、两台`worker`

## 配置主机名

去三台主机分别执行

```shell
hostnamectl set-hostname k8s-master && bash
hostnamectl set-hostname k8s-node1 && bash
hostnamectl set-hostname k8s-node2 && bash
```

## 配置hosts

```shell
cat >> /etc/hosts <<EOF
192.168.100.121 k8s-master
192.168.100.122 k8s-node1
192.168.100.123 k8s-node2
EOF
```

## 关闭防火墙和SElinux

```shell
systemctl stop firewalld && systemctl disable firewalld
setenforce 0
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
```

## 关闭swap分区

```shell
swapoff -a
sed -i '/swap/s/^/#/g' /etc/fstab
```

## 配置内核参数和优化

```shell
cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sysctl --system
```

## 安装ipset、ipvsadm

```shell
yum -y install conntrack ipvsadm ipset jq iptables curl sysstat libseccomp wget vim net-tools git

cat >/etc/modules-load.d/ipvs.conf <<EOF
# Load IPVS at boot
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
nf_conntrack_ipv4
EOF

systemctl enable --now systemd-modules-load.service
```

## 验证内核模块加载成功

```shell
lsmod | egrep "ip_vs|nf_conntrack_ipv4"
```

## 安装docker

### 安装依赖包

```shell
yum -y install yum-utils device-mapper-persistent-data lvm2
```

### 添加腾讯docker源

> 已在最开始的脚本中添加，记录一下添加源的命令`yum-config-manager --add-repo $URL`

### 安装docker并配置开机自启动

```shell
systemctl enable --now docker
```

### 添加阿里源docker仓库加速器（已废弃）

> 最开始的ToDockerHub脚本，已经配置了可以直接拉取`dockerhub`

## 安装Cri-Dockerd

### 安装`wget`

```shell
yum install -y wget
```

### 下载安装包并安装

> 这一个下载太慢了，可以复制链接去浏览器下载然后上传到虚拟机

```shell
wget -c https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.1/cri-dockerd-0.3.1-3.el7.x86_64.rpm
yum -y install cri-dockerd-0.3.1-3.el7.x86_64.rpm
```

配置沙盒（Pause）镜像

<div style="color: red; font-weight: bold;">     注意: 阿里源已被弃用，配置下一行的DockerHub源。 </div>

```shell
sed -i '/ExecStart/s#dockerd#& --network-plugin=cni --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.9#' /usr/lib/systemd/system/cri-docker.service
```

> 配置从DockerHub拉取基础镜像

```shell
sed -i '/^ExecStart/s#dockerd#& --network-plugin=cni --pod-infra-container-image=registry.hub.docker.com/library/pause:3.9#' /usr/lib/systemd/system/cri-docker.service
```

### 启动Cri-Dockerd并配置自启动

```shell
systemctl enable --now cri-docker
```

---

> 完成此阶段的脚本`不包含修改主机名`

```shell
#!/bin/bash
cat >> /etc/hosts <<EOF
192.168.100.121 k8s-master
192.168.100.122 k8s-node1
192.168.100.123 k8s-node2
EOF
swapoff -a
sed -i '/swap/s/^/#/g' /etc/fstab
cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sysctl --system
yum -y install conntrack ipvsadm ipset jq iptables curl sysstat libseccomp wget vim net-tools git

cat >/etc/modules-load.d/ipvs.conf <<EOF
# Load IPVS at boot
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
nf_conntrack_ipv4
EOF

systemctl enable --now systemd-modules-load.service
lsmod | egrep "ip_vs|nf_conntrack_ipv4"
yum -y install yum-utils device-mapper-persistent-data lvm2
systemctl enable --now docker
yum install -y wget
wget -c https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.1/cri-dockerd-0.3.1-3.el7.x86_64.rpm
yum -y install cri-dockerd-0.3.1-3.el7.x86_64.rpm
sed -i '/^ExecStart/s#dockerd#& --network-plugin=cni --pod-infra-container-image=registry.hub.docker.com/library/pause:3.9#' /usr/lib/systemd/system/cri-docker.service
systemctl enable --now cri-docker
```

---

# 安装kubectl、kubelet、kubeadm

<div style="color: red; font-weight: bold; font-size: 35px;">所有节点执行</div>

## 添加谷歌`kubernetes`源

```shell
cat >/etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

## 查看可用版本

```shell
yum --showduplicates list kubelet | grep 1.28
```

## 安装指定版本

```shell
yum install -y kubectl-1.28.2 kubelet-1.28.2 kubeadm-1.28.2
```

## 启动`kubelet`

```shell
systemctl enable --now kubelet
```

> 完成此阶段的脚本

```shell
#!/bin/bash
cat >/etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
yum install -y kubectl-1.28.2 kubelet-1.28.2 kubeadm-1.28.2
systemctl enable --now kubelet
```

---

# 初始化`kubernetes`集群

<div style="color: red; font-weight: bold; font-size: 35px;">只需要在主节点执行</div>

## 查看需要的镜像

```shell
kubeadm config images list --kubernetes-version=v1.28.2
```

一条命令拉取所需镜像，提前拉取可以加速初始化的过程

```shell
kubeadm config images pull --kubernetes-version=v1.28.2
```

## 初始化集群

只需要在主节点

> 由于配置了可以直接从`dockerhub`拉取镜像，所以无需再指定镜像仓库

```shell
kubeadm init --kubernetes-version=1.28.2 \
--apiserver-advertise-address=192.168.100.121 \
--service-cidr=172.15.0.0/16 --pod-network-cidr=172.16.0.0/16 \
--cri-socket unix://var/run/cri-dockerd.sock
```

`--kubernetes-version=1.28.2`：指定要安装的 Kubernetes 版本。

`--apisever-advertise-address=192.168.100.121`：设置 API 服务器的地址，节点将使用此地址进行通信。

`--service-cidr=172.15.0.0/16`：定义 Kubernetes 服务的 CIDR 范围，服务将使用此范围的 IP 地址。创建的`Service`就是这个网段，**不可与`pod`冲突**

`--pod-network-cidr=172.16.0.0/16`：指定 Pod 网络的 CIDR 范围，以便后续安装网络插件。创建的`Pod`就是这个网段，**不可与`service`冲突**

`--cri-socket unix://var/run/cri-dockerd.sock`：指定容器运行时的 socket，通常用于 Docker 或其他 CRI 兼容的运行时。



