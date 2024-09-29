---
title: 一篇学习k8s的记录
date: 2024-09-21 08:29:14
tags:
    - k8s
description: 加深对k8s的理解
cover: https://raw.gitmirror.com/ByteQuestor/picture/main/kubernetes/k8s.jpg
---

原文链接：[📚 Kubernetes（K8S）简介 - K8S 教程 - 易文档 (easydoc.net)](https://k8s.easydoc.net/docs/dRiQjyTY/28366845/6GiNOzyZ/9EX8Cp45)

# k8s简介

## 部署应用的方案

+ 传统部署
  直接在物理机上部署，机器资源分配不好控制，如果出现bug，可能机器的大部分资源被某个应用占用，导致其他应用无法正常运行，无法做到应用隔离
+ 虚拟机部署
  在物理机上运行多个虚拟机，每个虚拟机都是完整独立的系统，性能损耗大
+ 容器部署
  所有容器共享主机，轻量级的虚拟机，性能损耗小，资源隔离，CPU和内存可按需分配

## 什么时候需要k8s

如果应用只是跑在一台机器，`docker + docker-compose`

如果应用跑在3,4台机器上，每台机器单独配置运行环境 + 负载均衡其；

当应用访问数不断增加，机器逐渐增加到**几十上千台**，进行**加机器、软件更新、版本回滚**都会很麻烦

k8s就是可以一个命令搞定这些麻烦事，不停机的灰度更新，确保高可用、高性能、高扩展

## k8s集群架构

最少要两台机器

+ 主机点(master)
  **控制平台，不需要很高性能**，不跑任务，通常一个就可以了，也可以多开提高集群可用度
+ 工作节点(worker)
  可以说虚拟机或物理机，任务都在这里跑，机器要性能好，通常有多个，每个工作节点都由主节点管理
  《插入手绘图》



## 重要概念Pod（豆荚）

k8s调度，管理的最小单位，主节点负责考量负载自动调度Pod到哪个**节点(worker)**运行

+ 一个**Pod可以包含一个或多个容器**
+ 每个Pod都有自己的虚拟IP
+ 一个工作节点有多个Pod

> 总结：``worker`包含`pod`，`pod`包含容器

## k8s组件

+ `kube-apiserver`API服务器，公开了`kubernetes API`
+ `etcd`键值数据库，可以作为保存k8s所有集群数据的后台数据库
+ `kube-scheduler`调度`Pod`到哪个节点运行
+ `kube-controller`集群控制器
+ `cloud-controller`与云服务商交互

# 安装k8s集群

## 主机名映射

## 关闭SElinux & 防护墙

## 添加安装源

建议安装版本不超过`1.24`，因为`1.24`以后默认不支持`docker`了，得用`podman`或单独安装插件

```shell
# Centos的
# 添加 k8s 安装源
cat <<EOF > kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
mv kubernetes.repo /etc/yum.repos.d/

# 添加 Docker 安装源
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

## 安装所需组件（所有节点）

```shell
yum install -y kubelet-1.22.4 kubectl-1.22.4 kubeadm-1.22.4 docker-ce
```

## 启动 kubelet、docker，并设置开机启动（所有节点）

```shell
systemctl enable kubelet
systemctl start kubelet
systemctl enable docker
systemctl start docker
```

## 修改 docker 配置（所有节点）

```shell
# kubernetes 官方推荐 docker 等使用 systemd 作为 cgroupdriver，否则 kubelet 启动不了
cat <<EOF > daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://ud6340vz.mirror.aliyuncs.com"]
}
EOF
mv daemon.json /etc/docker/

# 重启生效
systemctl daemon-reload
systemctl restart docker
```

## 用 [kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/) 初始化集群（仅在主节点跑）

`token`是这条`init`命令生产的，在`log`里有

```shell
# 初始化集群控制台 Control plane
# 失败了可以用 kubeadm reset 重置
kubeadm init --image-repository=registry.aliyuncs.com/google_containers

# 记得把 kubeadm join xxx 保存起来
# 忘记了重新获取：kubeadm token create --print-join-command

# 复制授权文件，以便 kubectl 可以有权限访问集群
# 如果你其他节点需要访问集群，需要从主节点复制这个文件过去其他节点
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# 在其他机器上创建 ~/.kube/config 文件也能通过 kubectl 访问到集群
```



## 把工作节点加入集群（只在工作节点跑）

下面这条命令，在master安装成功后会提供，只需要直接复制即可

```shell
kubeadm join 172.16.32.10:6443 --token xxx --discovery-token-ca-cert-hash xxx
```



## 安装网络插件，否则 node 是 NotReady 状态（主节点跑）

```shell
# 很有可能国内网络访问不到这个资源，你可以网上找找国内的源安装 flannel
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

# 如果上面的插件安装失败，可以选用 Weave，下面的命令二选一就可以了。
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
kubectl apply -f http://static.corecore.cn/weave.v2.8.1.yaml

# 更多其他网路插件查看下面介绍，自行网上找 yaml 安装
https://blog.csdn.net/ChaITSimpleLove/article/details/117809007
```



---

# 部署应用到集群中

## 简单部署

```shell
kubectl run testApp --image=$镜像
```

## Pod

将YAML转为JSON可以更清楚地看到，每个缩进都是一个大括号，他们就是一组资源

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod	# pod的名字
spec:
  # 定义容器，可以多个
  containers:
    - name: test-k8s # 容器名字，容器是运行在pod里面的
      image: ccr.ccs.tencentyun.com/k8s-tutorial/test-k8s:v1 # 镜像
```

部署

```shell
kubectl apply -f $YAML文件
```

## Deployment

原文参考图
![参考图](https://cos.easydoc.net/46901064/files/kwpt8p8o.png)

每次部署一个Pod很不方便

```yaml
apiVersion: apps/v1
kind: Deployment	# 这个可以立即为直接建立一个组，就是这个组将部署多少个资源
metadata:
  # 部署名字
  name: test-k8s
spec:
  replicas: 2	# 运行的pod数量
  # 用来查找关联的 Pod，所有标签都匹配才行
  selector:
    matchLabels:
      app: test-k8s
  # 定义 Pod 相关数据
  template:
    metadata:
      labels:
        app: test-k8s	 # 注意这个标签，一定要跟selector 里的标签对应上
    spec:
      # 定义容器，可以多个
      containers:
      - name: test-k8s # 容器名字，这个不需要跟其他的一样，因为这是pod内部的容器
        image: ccr.ccs.tencentyun.com/k8s-tutorial/test-k8s:v1 # 镜像
```

注意：Pod部署我们起什么名字就是什么名字，Deployment部署，会在我们起的名字后加一段随机字符以避免重复

`Pod`部署和`Deployment`部署的结果都是一样的

**唯一的区别是**，`Deployment`部署的将会有此`yaml`文件统一管理，如需变化，只需要更改`yaml`文件再次`apply`即可

这样创建的资源，可以以下命令查看

```shell
# 这是所有的pod，包括直接创建的 pod 和 deployment部署 
kubectl get pod
kubectl get pod -o wide

# 也可以直接看Deployment(部署)
kubectl get deployment # 这样就看到了我们所有的Deployment及其pod概览
```

查看pod详细状态

```shell
kubectl describe $podname
```

查看pod日志

```shell
kubectl logs/$podname
kubectl logs/$podname -f	# 持续查看
```

进入pod

```shell
kubectl exec -it $podname
kubectl exec -it $podname -c $容器名字
```

删除Deployment

```shell
kubectl delete deployment $DeploymentName
```

重新Deployment

```shell
kubectl rollout restart deployment $DeploymentName
```

跟`docker`一样，只不过把`docker`换成了`kubectl`

## 一些好用的命令

### 命令行调度pod

```shell
kubectl scale deployment $DeployName --replicas=10
```

### 命令行绑定端口

```shell
kubectl port-forward $PodName 8888:80 # 将Pod的 80 端口绑定到本机 8888 
```

### 命令行修改镜像  

`--record` 表示把这个命令记录到操作历史中，就可以回退了

```shell
kubectl set image deployment $DeployMent $ContainerName=$Image --record
```

### 回滚操作

直接修改我们的镜像，不会杀掉以前的Pod，直到我们的新镜像完全部署才会杀掉以前的Pod

```shell
# 查看我们有多少可以回滚
kubectl rollout history deployment $DeploymentName
# 回退上一个版本（无需指定）,不会杀掉当前运行的pod，直到我们回滚的pod全部成功运行
kubectl rollout undo deployment $DeploymentName

# 回退后，同样会出现一个版本，这样如果再不指定版本就容易回滚容器回滚到错误的

# 回退到 1 版本（注意这个版本是history的版本号，而不是DeploymentName的版本号）
kubectl rollout undo deployment $DeploymentName --to-revision=1
```

### 暂停和恢复【冷门】

感觉太冷门了，用不上，记录一下

```shell
# 暂停运行，暂停后，对 deployment 的修改不会立刻生效，恢复后才应用设置
# 比如上面的命令行修改镜像，先暂停的话，直到恢复后才会生效
kubectl rollout pause deployment $DeploymentName

# 在这两天命令中间做操作，不会生效，等恢复后才会生效

# 恢复
kubectl rollout resume deployment $DeploymentName
```

### 输出到文件【冷门】

```shell
# 输出到文件
kubectl get deployment $DeploymentName -o yaml >> app2.yaml
```

### 删除全部资源【冷门】

> `注意：`删的是全部资源

```shell
# 删除全部资源
kubectl delete all --all
```

## YAML指定某个节点运行

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd
```

可以更清楚地看到，这个缩进的影响范围，

## 下节预告

### 工作负载分类【了解】

- Deployment
  适合无状态应用，所有pod等价，可替代
- StatefulSet
  有状态的应用，适合数据库这种类型。
- DaemonSet
  在每个节点上跑一个 Pod，可以用来做节点监控、节点日志收集等
- Job & CronJob
  Job 用来表达的是一次性的任务，而 CronJob 会根据其时间规划反复运行。

[文档](https://kubernetes.io/zh/docs/concepts/workloads/)

### 现存问题

- 每次只能访问一个 pod，没有负载均衡自动转发到不同 pod
- 访问还需要端口转发
- Pod 重创后 IP 变了，名字也变了

这就是`Service`要做的事情

---

# Service

## 特性

+ `Service`通过`label`关联对应的`Pod`
+ `Service`生命周期不限`Pod`绑定,不会因为`Pod`重创改变`IP`
+ 提供了负载均衡功能，自动转发流量到不同`Pod`
+ 可对集群外部提供访问端口
+ 集群内部可通过服务名字访问

![service特性](https://cos.easydoc.net/46901064/files/kwpuoh0h.png)

## 创建Service

关于服务类型的介绍

+ `ClusterIP`【集群内部访问】（可以进入到 Pod 里面访问）

+ `NodePort`【节点可访问】（换成这个可暴露端口后访问）
+ `LoadBalancer`【负载均衡模式】（需要负载均衡器才能使用）

服务的默认类型是`ClusterIP`

```shell
kubectl exec -it pod-name -- bash
curl http://test-k8s:8080
```

如果要在集群外部访问，可以通过端口转发实现（只适合临时测试用）

```shell
kubectl port-forward service/test-k8s 8888:8080
```

`YAML`文件解析

```yaml
apiVersion: v1
kind: Service
metadata:
  name: test-k8s # 本 Service 的名字
spec:
  selector:
    app: test-k8s	 # 本 Service 的要集结哪些 pod
  type: ClusterIP	# 服务类型
  ports:
    - port: 8080        # 本 Service 的端口,就是外部将要访问的端口
      targetPort: 8080  # 容器端口，被转发过去的端口
```

应用服务

```shell
kubectl apply -f service.yaml
```

查看服务

```shell
kubectl get svc
```

### 对外暴露服务 & 负载均衡

上面是通过端口转发的方式可以在外面访问到集群里的服务

如果想要直接把集群服务暴露出来，我们可以使用`NodePort` 和 `Loadbalancer` 类型的 Service，此时也具有了负载均衡的效果

`Loadbalancer` 也可以对外提供服务，这需要一个负载均衡器的支持，因为它需要生成一个新的 IP 对外服务，否则状态就一直是 pendding，这个很少用了，后面会有**更高端的 Ingress 来代替它**。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: test-k8s
spec:
  selector:
    app: test-k8s
  # 默认 ClusterIP 集群内可访问，NodePort 节点可访问，LoadBalancer 负载均衡模式（需要负载均衡器才可用）
  type: NodePort	# 需要新增节点端口
  ports:
    - port: 8080        # 本 Service 的端口
      targetPort: 8080  # 容器端口
      nodePort: 31000   # 节点端口，范围固定 30000 ~ 32767
```

## 多端口

对于某些 Service，需要公开多个端口。k8s允许为 Service 对象配置多个端口定义。

 为 Service 使用多个端口时，必须为所有端口提供名称，以使它们无歧义

```yaml
apiVersion: v1
kind: Service
metadata:
  name: test-k8s
spec:
  selector:
    app: test-k8s
  type: NodePort
  ports:
    - port: 8080        # 本 Service 的端口
      name: test-k8s    # 必须配置
      targetPort: 8080  # 容器端口
      nodePort: 31000   # 节点端口，范围固定 30000 ~ 32767
    - port: 8090
      name: test-other
      targetPort: 8090
      nodePort: 32000
```



## 本节内容

### ClusterIP

默认的，仅在集群内可用

### NodePort

暴露端口到节点，提供了集群外部访问的入口
端口范围固定 30000 ~ 32767

### LoadBalancer

需要负载均衡器（通常都需要云服务商提供，裸机可以安装 [METALLB](https://metallb.universe.tf/) 测试）
会额外生成一个 IP 对外服务，后续会有更高级的`Ingress`
K8S 支持的负载均衡器：[负载均衡器](https://kubernetes.io/zh/docs/concepts/services-networking/service/#internal-load-balancer)

### Headless

适合数据库
clusterIp 设置为 None 就变成 Headless 了，不会再分配 IP，后面会再讲到具体用法

---

# StatefuSet

## 什么是StatefuSet

`StatefulSet `是用来**管理有状态的应用**，例如**数据库**。

前面部署的应用，都是**不需要存储数据，不需要记住状态**的，可以**随意扩充副本**，每个副本都是一样的，可替代的。

而像**数据库、Redis**这类有状态的，不能随意扩充副本的，`StatefulSet `会固定每个 Pod 的名字。

## StatefulSet 特性

- Service 的 `CLUSTER-IP` 是空的，Pod 名字也是固定的。
- Pod 创建和销毁是有序的，创建是顺序的，销毁是逆序的。
- Pod 重建不会改变名字，除了IP，所以不要用IP直连

## 部署 StatefulSet 类型的 Mongodb

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  serviceName: mongodb
  replicas: 3
  selector:
    matchLabels:
      app: mongodb	# 用来匹配 POd
# 下面就是要匹配的pod（是会起的）
  template:
    metadata:
      labels:
        app: mongodb # pod名字
    spec:
      containers:
        - name: mongo # pod里容器的名字
          image: mongo:4.4
          # IfNotPresent 仅本地没有镜像时才远程拉，Always 永远都是从远程拉，Never 永远只用本地镜像，本地没有则报错
          imagePullPolicy: IfNotPresent
---
# 下面用一个 Service 统一一个入口来访问
apiVersion: v1
kind: Service
metadata:
  name: mongodb
spec:
  selector:
    app: mongodb	# 这个选择的是 pod的标签，所有拥有此标签的会被 Service 统一管理
  type: ClusterIP
  # HeadLess
  clusterIP: None	# 这样就会通过服务名字来访问
  ports:
    - port: 27017
      targetPort: 27017
```

创建

```shell
kubectl apply -f mongo.yaml
```

查看

```shell
kubectl get pod
kubectl get svc
kubectl get statefulset
```

`Endpoint`s 会多一个 `hostname`

```shell
kubectl get endpoints mongodb -o yaml
```

访问时，如果直接使用 `Service `名字连接，会随机转发请求
要连接指定 Pod，可以这样`pod-name.service-name`,运行一个临时 Pod 连接数据测试下

```shell
kubectl run mongodb-client --rm --tty -i --restart='Never' --image docker.io/bitnami/mongodb:4.4.10-debian-10-r20 --command -- bash
```

### Web 应用连接 Mongodb

在集群内部，可以通过服务名字访问到不同的服务，

进入我们临时Pod，指定连接第一个

```shell
mongo --host mongodb-0.mongodb
```

第二个第三个都是这样，之所以通过名字，是因为IP会变，但是名字不会变



## 问题

+ 起的三个服务各自独立
+ 重建后数据丢失

下一节**数据持久化**解决

# 数据持久化

## 前言

kubernetes 集群不会处理数据的存储，可以为数据库挂载一个磁盘来确保数据的安全。可以选择云存储、本地磁盘、NFS。

- 本地磁盘：可以挂载某个节点上的目录，但是这需要限定 pod 在这个节点上运行
- 云存储：不限定节点，不受集群影响，安全稳定；需要云服务商提供，裸机集群是没有的。
- NFS：不限定节点，不受集群影响

> 综上所述，还是买云存储最好

## hostPath 挂载

把节点上的一个目录挂载到 Pod，但是已经不推荐使用了。配置方式简单，但是**需要手动指定 Pod 跑在某个固定的节点**。不适用于多节点集群。

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  serviceName: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongo
          image: mongo:4.4
          # IfNotPresent 仅本地没有镜像时才远程拉，Always 永远都是从远程拉，Never 永远只用本地镜像，本地没有则报错
          imagePullPolicy: IfNotPresent
# ---------------- 以下是新加入的部分 ---------------------- #        
          volumeMounts:
            - mountPath: /data/db # 容器里面的挂载路径
              name: mongo-data    # 卷名字，必须跟下面定义的名字一致
      volumes:
        - name: mongo-data              # 卷名字
          hostPath:
            path: /data/mongo-data      # 节点上的路径
            type: DirectoryOrCreate     # 指向一个目录，不存在时自动创建
```

## 存储介绍

下图有一个错误

![存储抽象](https://cos.easydoc.net/46901064/files/kwrmidne.png)

### Persistent Volume Claim (PVC)

对存储需求的一个申明，可以理解为一个申请单，系统根据这个申请单去找一个合适的 PV，还可以根据 PVC 自动创建 PV。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodata
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: "local-storage"
  resources:
    requests:
      storage: 2Gi
```

### Persistent Volume (PV)

描述卷的具体信息，例如磁盘大小，[访问模式](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/#access-modes)。[文档](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/)，[类型](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/#types-of-persistent-volumes)，[Local 示例](https://kubernetes.io/zh/docs/concepts/storage/volumes/#local)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodata
spec:
  capacity:
    storage: 2Gi
  volumeMode: Filesystem  # Filesystem（文件系统） Block（块）
  accessModes:
    - ReadWriteOnce       # 卷可以被一个节点以读写方式挂载
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /root/data
  nodeAffinity:
    required:
      # 通过 hostname 限定在某个节点创建存储卷
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - node2
```

### Storage Class (SC)

将存储卷划分为不同的种类，例如：SSD，普通磁盘，本地磁盘，按需使用。[文档](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/aws-ebs
parameters:
  type: io1
  iopsPerGB: "10"
  fsType: ext4
```

## 本地存储

所需文件汇总

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  replicas: 1
  serviceName: mongodb
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        image: mongo:5.0
        imagePullPolicy: IfNotPresent
        name: mongo
        volumeMounts:
          - mountPath: /data/db
            name: mongo-data
      volumes:
        - name: mongo-data
          persistentVolumeClaim:
             claimName: mongodata
---
apiVersion: v1
kind: Service
metadata:
  name: mongodb
spec:
  clusterIP: None
  ports:
  - port: 27017
    protocol: TCP
    targetPort: 27017
  selector:
    app: mongodb
  type: ClusterIP
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodata
spec:
  capacity:
    storage: 2Gi
  volumeMode: Filesystem  # Filesystem（文件系统） Block（块）
  accessModes:
    - ReadWriteOnce       # 卷可以被一个节点以读写方式挂载
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /root/data
  nodeAffinity:
    required:
      # 通过 hostname 限定在某个节点创建存储卷
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - node2
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodata
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: "local-storage"
  resources:
    requests:
      storage: 2Gi
```



## 问题

+ 当前数据库的连接地址是写死在代码里的，另外还有数据库的密码需要配置。

本地太麻烦了，还是买云存储

# ConfigMap & Secret

## ConfigMap

比如数据库连接地址，这种可能会根据环境变化的，我们不应该写死在代码里

`configmap.yaml`示例

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongo-config
data:
  mongoHost: mongodb-0.mongodb
```

```shell
# 应用
kubectl apply -f configmap.yaml
# 查看
kubectl get configmap mongo-config -o yaml
```



## Secret

一些重要数据，例如密码、`TOKEN`，可以放到`Secret `中

`secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongo-secret
# Opaque 用户定义的任意数据，更多类型介绍 https://kubernetes.io/zh/docs/concepts/configuration/secret/#secret-types
type: Opaque
data:
  # 数据要 base64。https://tools.fun/base64.html
  mongo-username: bW9uZ291c2Vy
  mongo-password: bW9uZ29wYXNz
```

```shell
# 应用
kubectl apply -f secret.yaml
# 查看
kubectl get secret mongo-secret -o yaml
```

## 使用方法

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongo
          image: mongo:4.4
          # IfNotPresent 仅本地没有镜像时才远程拉，Always 永远都是从远程拉，Never 永远只用本地镜像，本地没有则报错
          imagePullPolicy: IfNotPresent
# -------------------- 以下是新加入的部分 -------------------------------
          env:
          - name: MONGO_INITDB_ROOT_USERNAME
            valueFrom:
              secretKeyRef:
                name: mongo-secret
                key: mongo-username
          - name: MONGO_INITDB_ROOT_PASSWORD
            valueFrom:
              secretKeyRef:
                name: mongo-secret
                key: mongo-password
          # Secret 的所有数据定义为容器的环境变量，Secret 中的键名称为 Pod 中的环境变量名称
          # envFrom:
          # - secretRef:
          #     name: mongo-secret
```

##### 挂载为文件（更适合证书文件）【了解】

挂载后，会在容器中对应路径生成文件，一个 key 一个文件，内容就是 value，[文档](https://kubernetes.io/zh/docs/concepts/configuration/secret/#using-secrets-as-files-from-a-pod)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
```

# Helm & 命名空间

## 介绍

`Helm`类似 npm，pip，docker hub, 可以理解为是一个软件库，可以方便快速的为集群安装一些第三方软件。
使用 `Helm`可以非常方便的就搭建出来 `MongoDB `/ `MySQL `副本集群，`YAML `文件别人都给写好了，直接使用。[官网](https://helm.sh/zh/)，[应用中心](https://artifacthub.io/)

## 安装 Helm

安装 [文档](https://helm.sh/zh/docs/intro/install/)
`curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash`

本地有安装包

## Helm安装 MongoDB 示例

```yaml
# 安装
# 将 Bitnami 的 Helm 仓库添加到本地 Helm 配置中，方便后续安装 MongoDB
helm repo add bitnami https://charts.bitnami.com/bitnami
# Helm 安装一个名为 my-mongo 的 MongoDB 集群。默认情况下，这会安装一个单实例 MongoDB
helm install my-mongo bitnami/mongodb
# 安装mysql（命令记录）
helm install my-mysql bitnami/mysql --set architecture="replication",auth.rootPassword="yourpassword",auth.replicationPassword="replicapassword"

# 指定密码和架构
# 将 MongoDB 部署为副本集（replica set）架构，并设置 root 用户密码为 mongopass
helm install my-mongo bitnami/mongodb --set architecture="replicaset",auth.rootPassword="mongopass"

# 删除
# 使用 helm ls 查看已安装的 Helm 发行版，通过 helm delete 删除 my-mongo 这个实例
helm ls
helm delete my-mongo

# 查看密码
# 查看存储在 Kubernetes Secret 中的 MongoDB root 密码，可以以 JSON 或 YAML 格式输出并保存
kubectl get secret my-mongo-mongodb -o json
kubectl get secret my-mongo-mongodb -o yaml > secret.yaml

# 临时运行一个包含 mongo client 的 debian 系统
# 运行一个临时容器进入 MongoDB 客户端环境，使用 bitnami/mongodb 镜像来访问 MongoDB
kubectl run mongodb-client --rm --tty -i --restart='Never' --image docker.io/bitnami/mongodb:4.4.10-debian-10-r20 --command -- bash

# 进去 mongodb
# 使用 MongoDB 客户端登录到 MongoDB 实例，连接地址为 my-mongo-mongodb，使用 root 用户名和密码
mongo --host "my-mongo-mongodb" -u root -p mongopass

# 也可以转发集群里的端口到宿主机访问 mongodb
# 将集群中的 MongoDB 服务端口 27018 转发到本地主机的 27017，以便可以在本地通过 localhost:27017 访问 MongoDB
kubectl port-forward svc/my-mongo-mongodb 27017:27018
```

好处是，可以直接安装高可用，不需要我们手动操作

## 命名空间

如果一个集群中部署了多个应用，所有应用都在一起，就不太好管理，也可以导致名字冲突等。
我们可以使用 `namespace `把应用划分到不同的命名空间，跟代码里的 namespace 是一个概念，只是为了划分空间。

```shell
# 创建命名空间
kubectl create namespace testapp
# 部署应用到指定的命名空间
kubectl apply -f app.yml --namespace testapp
# 查询
kubectl get pod --namespace kube-system
```

可以用 `kubens`快速切换 `namespace`

```shell
# 切换命名空间
kubens kube-system
# 回到上个命名空间
kubens -
# 切换集群
kubectx minikube
```



# Ingress

## 介绍

Ingress 为外部访问集群提供了一个 **统一** 入口，避免了对外暴露集群端口；
功能类似 Nginx，可以根据域名、路径把请求转发到不同的 Service。
可以配置 https

## **跟 LoadBalancer 有什么区别？**

+ `LoadBalancer `需要对外暴露端口，不安全

+ 无法根据域名、路径转发流量到不同 `Service`，多个 `Service `则需要开多个 `LoadBalancer`；

+ 功能单一，无法配置 `https`

## 使用

要使用 `Ingress`，需要一个负载均衡器 + `Ingress Controller`

如果是裸机（bare metal) 搭建的集群，需要自己安装一个负载均衡插件，可以安装 [METALLB](https://metallb.universe.tf/)

如果是云服务商，会自动给配置，否则外部 IP 会是 “pending” 状态，无法使用。

文档：[Ingress](https://kubernetes.io/zh/docs/concepts/services-networking/ingress/)

Minikube 中部署 Ingress Controller：[nginx](https://kubernetes.io/zh/docs/tasks/access-application-cluster/ingress-minikube/)

Helm 安装： [Nginx](https://kubernetes.github.io/ingress-nginx/deploy/#quick-start)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-example
spec:
  ingressClassName: nginx
  rules:
  - host: tools.fun
    http:
      paths:
      - path: /easydoc
        pathType: Prefix
        backend:
          service:
            name: service1
            port:
              number: 4200
      - path: /svnbucket
        pathType: Prefix
        backend:
          service:
            name: service2
            port:
              number: 8080
```

# 训练方案





