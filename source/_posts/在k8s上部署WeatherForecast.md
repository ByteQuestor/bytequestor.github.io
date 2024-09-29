---
title: 在k8s上部署WeatherForecast
date: 2024-09-16 09:22:48
tags: 
  - k8s
  - istio
description: 在k8s上部署WeatherForecast
cover: https://raw.gitmirror.com/ByteQuestor/picture/main/WeatherForecast.png
---

# 环境准备

## 运行环境准备

+ 基于centos7部署的 && 正常运行  && 能上网的k8s集群
+ 已搭建istio环境
+ 已经上传本地镜像（因为去docker hub上太慢,而且经常性会有连不上的风险，所以拉到本地反复练习）

## 资源环境准备

百度网盘里已经都准备好了，可直接从**运行加载镜像脚本**开始，当然也可以按照下面的流程走一遍

```tex
通过网盘分享的文件：k8s部署WeatherForeCast
链接: https://pan.baidu.com/s/1YNK0O0nneFtSFFdgCHFJmQ?pwd=mvsb 提取码: mvsb 
--来自百度网盘超级会员v4的分享
```

此链接永久有效

+ 下载Weather Forecast资源包

```git
git clone https://github.com/cloudnativebooks/cloud-native-istio
```

![镜像展示](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sDepolyWeatherForecast/1.png)

+ 从docker hub上拉取（太慢了）
  链接：[istioweather's Profile | Docker Hub](https://hub.docker.com/u/istioweather)

### 运行加载镜像脚本

```shell
bash load_images.sh
```

加载了本地镜像后，就不会再出现镜像拉取失败了（注意要master和worker节点都要加载，不然会有概率出现镜像拉取失败）

# 部署应用服务

现在可以直接部署应用服务了

## 文件位置

进入如下目录(`istioWeather`是自己定义的存放资源的位置)

```shell
cd istioWeather/cloud-native-istio/08_environment/8.3/
```

![图片](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sDepolyWeatherForecast/1_1.png)

## YAML扫盲

### **Service（服务）**

`Service` 的主要作用是为一组运行中的 Pod 提供**稳定的网络接口**，并进行**负载均衡**。Pod 是短暂的，可能因为扩缩容、失败重启等原因会频繁更改 IP 地址。为了确保集群内其他组件（如其他服务、外部流量）可以通过一个固定的方式访问这些动态变化的 Pod，`Service` 提供了一个稳定的 IP 地址和 DNS 名字。

具体作用：

- **负载均衡**：将客户端请求分发给多个匹配的 Pod。Service 通过 `selector` 字段找到拥有特定标签的 Pod，并自动将流量分配到这些 Pod 上。

- **服务发现**：Service 提供一个固定的 ClusterIP（集群内部 IP 地址）或者 DNS 名字，客户端可以通过这个地址或名字访问它服务的 Pod，而不需要知道 Pod 的具体 IP 地址。

- 暴露服务

  Service 可以通过多种类型暴露服务，比如：

  - **ClusterIP**（默认类型）：只在集群内部暴露服务。
  - **NodePort**：将服务暴露到每个节点的固定端口上，使得外部流量可以通过该端口访问集群中的服务。
  - **LoadBalancer**：在云环境中使用外部负载均衡器，将服务暴露到外部。

### **Deployment（部署）**

`Deployment` 的主要作用是管理应用程序的**容器化部署**，包括容器的**生命周期管理**、**副本控制**、**滚动更新**等。Deployment 定义了一个应用程序的期望状态，Kubernetes 会持续监控和维护这个状态。

具体作用：

- **定义 Pod 模板**：在 `Deployment` 中定义了 Pod 的模板（包括容器镜像、环境变量、端口等），Kubernetes 会根据这个模板启动 Pod。
- **副本控制**：通过 `replicas` 字段，可以设置运行的 Pod 副本数量。Kubernetes 会确保有正确数量的副本在运行。
- **滚动更新**：当你更新 Deployment 时，Kubernetes 会逐渐将旧的 Pod 替换为新的 Pod，而不会导致服务中断。
- **容错**：如果某个 Pod 崩溃或被手动删除，Deployment 会自动重启 Pod，以维持定义的副本数。

## weather-v1文件

> weather-v1.yaml 中包含了**为三个不同服务（Frontend、Advertisement 和 Forecast）**创建 Kubernetes `Service` 和 `Deployment` 资源的 YAML 配置。每个服务都有对应的网络暴露 (`Service`) 和 Pod 的部署 (`Deployment`) 配置

```yaml
##################################################################################################
# Frontend service
##################################################################################################
apiVersion: v1
kind: Service
metadata:
  name: frontend
  labels:
    app: frontend
    service: frontend
spec:
  ports:
  - port: 3000
    name: http
  selector:
    app: frontend
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-v1
  labels:
    app: frontend
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
      version: v1
  template:
    metadata:
      labels:
        app: frontend
        version: v1
    spec:
      containers:
      - name: frontend
        image: istioweather/frontend:v1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3000
---
##################################################################################################
# Advertisement service
##################################################################################################
apiVersion: v1
kind: Service
metadata:
  name: advertisement
  labels:
    app: advertisement
    service: advertisement
spec:
  ports:
  - port: 3003
    name: http
  selector:
    app: advertisement
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: advertisement-v1
  labels:
    app: advertisement
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: advertisement
      version: v1
  template:
    metadata:
      labels:
        app: advertisement
        version: v1
    spec:
      containers:
      - name: advertisement
        image: istioweather/advertisement:v1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3003
---
##################################################################################################
# Forecast service
##################################################################################################
apiVersion: v1
kind: Service
metadata:
  name: forecast
  labels:
    app: forecast
    service: forecast
spec:
  ports:
  - port: 3002
    name: http
  selector:
    app: forecast
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: forecast-v1
  labels:
    app: forecast
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: forecast
      version: v1
  template:
    metadata:
      labels:
        app: forecast
        version: v1
    spec:
      containers:
      - name: forecast
        image: istioweather/forecast:v1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3002
---
```

> 这个 YAML 文件定义了三个服务：Frontend、Advertisement 和 Forecast
>
> 每个服务都通过 `Service` 和 `Deployment` 资源进行定义
>
> - `Service` 用于暴露服务端口，并通过选择器匹配 Pod
> - `Deployment` 定义了每个服务的副本数量、Pod 模板和容器的配置

### Frontend 服务

+ Fronted Service
  ```yaml
  apiVersion: v1			# 指定 Kubernetes API 版本，这里用的是 v1。
  kind: Service				# 定义一个服务资源，用于暴露 Pod。
  metadata:					
    name: frontend		# metadata.name: frontend：服务的名称是 frontend，可在集群内使用这个名字访问。【唯一性】
    labels:					 # 为服务打上标签，便于管理和选择。
    
      app: frontend		# 服务发现：Service 会根据 selector 字段找到带有 app: frontend 标签的 Pod，将流量路由到这些 Pod
      							#  资源管理：用 kubectl 命令过滤带有特定标签的资源。
  									#						例如，可以通过 kubectl get pods -l app=frontend 获取所有带有 app: frontend 标签的 Pod
      
      service: frontend			# 【非必须】更进一步的标签，比如我们有多个服务的话需要进一步标记（前端、后端等）
  spec:
    ports:						# spec.ports：定义服务的端口，这里指定外部访问时使用 3000 端口。
    - port: 3000
      name: http			#给端口起的一个名字
    selector:
      app: frontend		# selector.app: frontend：指定要暴露的 Pod（Deployment 中定义的 Pod），根据标签 app: frontend 选择匹配的 Pod。
  ```

+ Frontend Deployment
  ```yaml
  apiVersion: apps/v1			# 定义使用 apps/v1 API 版本来创建一个 Deployment
  kind: Deployment				# 这是一个用于管理应用的 Deployment，定义了应用的副本和更新策略
  metadata:
    name: frontend-v1			# metadata.name: frontend-v1：Deployment 名为 frontend-v1
    labels:
      app: frontend				# 打上 app: frontend 和 version: v1 标签，用于选择和管理
      version: v1
  spec:
    replicas: 1						# 定义前端服务只需要一个副本（Pod）
    selector:
      matchLabels:
        app: frontend		# selector.matchLabels：定义 Deployment 的选择器，标签 app: frontend 和 version: v1 的 Pod 会被管理
        version: v1
    template:
      metadata:
        labels:					# template.metadata.labels：定义 Pod 的标签，和选择器相匹配
          app: frontend	
          version: v1
      spec:
        containers:				# spec.containers：定义了 Pod 中的容器，这里是 frontend 容器
        - name: frontend
          image: istioweather/frontend:v1			# image: istioweather/frontend:v1：拉取 istioweather/frontend:v1 镜像作为容器
          imagePullPolicy: IfNotPresent		
          ports:
          - containerPort: 3000			# ports.containerPort: 3000：容器暴露的端口是 3000
  ```

### Advertisement 服务

+ Advertisement Service
  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: advertisement
    labels:
      app: advertisement
      service: advertisement		# 这个就是进一步区分不同服务的
  spec:
    ports:
    - port: 3003
      name: http
    selector:
      app: advertisement			# 这个对应 metadata 里的app标签,表面将这个3003端口给它
  ```

+ Advertisement Deployment
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: advertisement-v1
    labels:
      app: advertisement
      version: v1
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: advertisement
        version: v1
    template:
      metadata:
        labels:
          app: advertisement
          version: v1
      spec:
        containers:
        - name: advertisement
          image: istioweather/advertisement:v1
          imagePullPolicy: IfNotPresent
          ports:
          - containerPort: 3003
  ```

### Forecast 服务

+ Forecast Service
  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: forecast
    labels:
      app: forecast
      service: forecast
  spec:
    ports:
    - port: 3002
      name: http
    selector:
      app: forecast
  ```

+ Forecast Deployment
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: forecast-v1
    labels:
      app: forecast
      version: v1
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: forecast
        version: v1
    template:
      metadata:
        labels:
          app: forecast
          version: v1
      spec:
        containers:
        - name: forecast
          image: istioweather/forecast:v1
          imagePullPolicy: IfNotPresent
          ports:
          - containerPort: 3002
  ```

### 部署应用服务

+ 创建命名空间

  ```shell
  kubectl create ns weather
  ```

+ 给命名空间打标签
  ```shell
  kubectl label ns weather istio-injection=enabled
  ```

+ 使用kubectl命令部署

  ```shell
  kubectl apply -f weather-v1.yaml -n weather
  ```

+ 检查是否全部正常启动

  + 正常启动
    ![正常启动](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sDepolyWeatherForecast/1_1.png)

    **【下方错误已找到原因，留下做个纪念】**是因为我忘记在worker节点上传镜像，所以当任务落在worker节点上时，就会拉取镜像失败，删除后会换节点创建，就出现了删一次就成功的现象

  + 下面有个异常的，直接删了让k8s重起一个

    ```shell
    kubectl get po -n weather
    ```

    ![异常](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sDepolyWeatherForecast/2.png)

  + 删除错误pod
    ```shell
    kubectl delete po advertisement-v1-646d667b5-xvmbg -n weather
    ```

    ![删除重启](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sDepolyWeatherForecast/3.png)

+ 此时是访问不了的
  ![正常启动](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sDepolyWeatherForecast/1_4.png)
  ![正常启动](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sDepolyWeatherForecast/1_5.png)

## weather-gateway文件

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: weather-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: frontend-dr
  namespace: weather
spec:
  host: frontend
  subsets:
  - name: v1
    labels:
      version: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: frontend-route
  namespace: weather
spec:
  hosts:
  - "*"
  gateways:
  - istio-system/weather-gateway
  http:
  - match:
    - port: 80
    route:
    - destination:
        host: frontend
        port:
          number: 3000
        subset: v1
```

> 这个 YAML 文件定义了三个 Istio 相关的资源：**Gateway**、**DestinationRule** 和 **VirtualService**。它们一起控制了流量如何从外部访问内部的服务，并定义了路由规则



### **Gateway（weather-gateway）**


Gateway 用于将外部流量引入网格（Istio 服务网格），它通过开放特定的端口和协议来接收外部请求。

```yaml
apiVersion: networking.istio.io/v1alpha3		# 使用 Istio 提供的 API 版本，用于网络相关的资源。
kind: Gateway		# 声明这个资源是一个 Gateway，它定义了如何将外部流量接入到集群内的 Istio 服务网格中。
metadata:
  name: weather-gateway	# 名字【唯一性】
  namespace: istio-system		# 该 Gateway 资源部署在 `istio-system` 命名空间中，通常是 Istio 控制面板所在的命名空间。
spec:	
  selector:	# 选择器，指定使用 `istio: ingressgateway` 作为网关控制器，通常这是 Istio 安装时的默认 ingress gateway。
    istio: ingressgateway  # 使用 Istio 的默认 ingressgateway 控制器
  servers:		# 定义服务器配置。
  - port:
      number: 80	# 开放端口 80，使用 `HTTP` 协议。
      name: http	# ！！！注意：这个是咱们之前给端口起的名字（3000，3001，3002）
      protocol: HTTP # 这个是协议
    hosts:
    - "*"		# 允许接受的主机名，这里表示所有主机
```

### **DestinationRule（frontend-dr）**

DestinationRule 用于定义目标服务的流量策略，比如负载均衡、连接池配置、版本控制等。这里，它定义了对 `frontend` 服务的路由，并根据版本控制进行流量分配。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: frontend-dr
  namespace: weather
spec:
  host: frontend
  subsets:
  - name: v1
    labels:
      version: v1
```

**`kind: DestinationRule`**：声明这个资源是一个 DestinationRule，用于定义服务的流量路由和策略。

**`namespace: weather`**：这个 DestinationRule 资源被放置在 `weather` 命名空间中。

**`host: frontend`**：指定目标服务为 `frontend`，即与 Deployment 中定义的 `frontend` 服务相关联。

**`subsets`**：定义了一组服务的子集，基于 `labels` 匹配规则。

- **`name: v1`**：子集名称为 `v1`。
- **`labels: version: v1`**：指定该子集对应标签为 `version: v1`，用于匹配 `frontend` 服务的特定版本。

### VirtualService（frontend-route）

VirtualService 用于定义请求如何在服务网格内路由，类似传统的负载均衡规则。它可以基于请求的路径、主机名等因素决定请求的目标服务。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: frontend-route
  namespace: weather
spec:
  hosts:
  - "*"
  gateways:
  - istio-system/weather-gateway
  http:
  - match:
    - port: 80
    route:
    - destination:		# 指流量最终到达的目标服务
        host: frontend
        port:
          number: 3000
        subset: v1
```

**`kind: VirtualService`**：声明这个资源是一个 VirtualService，用于控制流量在服务网格中的路由。

**`namespace: weather`**：该 VirtualService 部署在 `weather` 命名空间。

**`hosts: "\*"`**：表示 VirtualService 可以处理来自任何主机的请求。

**`gateways`**：指定使用 `istio-system/weather-gateway` 来处理外部流量。这个 Gateway 是前面定义的 `weather-gateway`。

**`http`**：定义了 HTTP 流量的路由规则。

- **`match: port: 80`**：匹配来自网关的端口 80 的请求。

- `route`

  定义了将匹配的请求路由到 `frontend` 服务。

  - **`host: frontend`**：目标服务是 `frontend`。
  - **`port: number: 3000`**：路由的目标是 `frontend` 服务的端口 3000。
  - **`subset: v1`**：将请求路由到 `DestinationRule` 中定义的 `v1` 子集，也就是 `frontend` 服务的版本 `v1`。

+ **整体工作流程**

> **Gateway**：`weather-gateway` 在端口 80 上接收外部 HTTP 请求，并允许所有主机（`*`）访问。
>
> **VirtualService**：`frontend-route` 定义了如何处理进入的流量。它通过 `weather-gateway` 接收流量，并将匹配到的流量路由到 `frontend` 服务的端口 3000。
>
> **DestinationRule**：`frontend-dr` 定义了将流量引导至 `frontend` 服务的 `v1` 子集（即服务的特定版本 `v1`）。
>
> 这个配置的最终效果是：当用户从外部访问集群时，Istio Gateway 会接收请求，并根据 VirtualService 的规则，将流量路由到 `frontend` 服务的 v1 版本

### 部署网关

```shell
kubectl apply -f weather-gateway.yaml 
```

# 成功可以访问

![image-20240916101441808](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sDepolyWeatherForecast/9.png)

![image-20240916101441808](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sDepolyWeatherForecast/1_6.png)

> 后续的部分仅是记录作用，这次练习不需要下述方法，下面是另一种访问方法

----



## 修改服务类型方法

此时还不能访问，因为我们起的是`ClusterIP`类型

![image-20240916095239955](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sDepolyWeatherForecast/4.png)

**ClusterIP** 服务只能在 Kubernetes 集群内部进行访问，无法从集群外部直接访问

### 外部访问

#### 修改服务类型为 NodePort 或 LoadBalancer

**NodePort**：允许你通过集群节点的 IP 地址和一个特定端口从外部访问服务。

**LoadBalancer**：在云环境下，LoadBalancer 会分配一个外部 IP 地址，并在云提供商创建负载均衡器。

+ **通过修改服务类型来实现**。例如，将 `frontend` 服务改为 NodePort
  进入YAML文件将 `type` 字段从 `ClusterIP` 改为 `NodePort`,保存退出后直接生效

  ```shell
  kubectl edit svc frontend -n weather
  ```

+ 通过命令直接修改
  直接生效

  ```shell
  kubectl expose deployment frontend --type=NodePort --name=frontend -n weather
  ```

#### 使用 Ingress Controller

可以通过 Ingress 配置来暴露服务，并且可以设置域名进行访问

#### 使用 kubectl port-forward

此方式只能通过本地访问,调试用

```shell
kubectl port-forward svc/frontend 8080:3000 -n weather
```

总结：生产环境，通常使用 **LoadBalancer** 或 **Ingress** 来暴露服务

## 修改服务类型

```shell
kubectl edit svc frontend -n weather
```

![image-20240916101151272](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sDepolyWeatherForecast/7.png)

![image-20240916101127712](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sDepolyWeatherForecast/6.png)

此时可以通过`49151`端口访问服务
![image-20240916101441808](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sDepolyWeatherForecast/8.png)

## 访问

可使用：http://\$IP:49151访问

**注意：一定是http而不是https，现在浏览器默认会添加https，会报以下错误**

![image-20240916101604953](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sDepolyWeatherForecast/10.png)

**使用http**

![image-20240916101441808](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sDepolyWeatherForecast/9.png)

