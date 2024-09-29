---
title: 在k8s上对WeatherForecast进行灰度发布
date: 2024-09-17 10:56:35
tags: 
  - k8s
  - istio
  - 灰度发布
cover: https://raw.gitmirror.com/ByteQuestor/picture/main/WeatherForecast.png
---

大公司都有自己的发布系统，对于初创公司来说，这样的系统有一定的门槛.利用`istio`提供的流量路由功能可以很方便地构建一个流量分配系统来做**灰度发布**和**AB测试**

---

# 准备工作

+ 将所有流量都路由到各个服务的v1版本（`上一篇已经部署`）
+ 用到的文件
  ![文件](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/gray/001.png)

# YAML文件解析

> 对每个服务创建各自的VirtuakService和DestinationRule资源，将访问请求路由到所有服务的V1版本

## destination-rule-v1

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: frontend-dr
spec:
  host: frontend
  subsets:
  - name: v1
    labels:
      version: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: advertisement-dr
spec:
  host: advertisement
  subsets:
  - name: v1
    labels:
      version: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: forecast-dr
spec:
  host: forecast
  subsets:
  - name: v1
    labels:
      version: v1
```

> 这个 YAML 文件定义了三个 **DestinationRule**，分别为 `frontend`、`advertisement` 和 `forecast` 三个服务。每个 **DestinationRule** 都为相应的服务定义了一个子集（subset），用于流量路由的版本控制，
>
> **DestinationRule** 用于定义服务的目标子集和流量路由规则。每个服务（`frontend`、`advertisement`、`forecast`）都通过标签（如 `version: v1`）指定了一个子集。
>
> 这些子集规则在实际应用中通常与 **VirtualService** 配合使用，可以帮助控制向服务的不同版本分发流量，支持 **A/B 测试** 或 **蓝绿部署** 等场景。
>
> 以下是详细解释

### DestinationRule for Frontend

```yaml
apiVersion: networking.istio.io/v1alpha3	# 使用 Istio 提供的 API 版本，`v1alpha3` 是与网络相关的资源的版本。
kind: DestinationRule	# 定义了一个 DestinationRule，用于指定流量规则和目标服务。
metadata:
  name: frontend-dr		# DestinationRule 的名字为 `frontend-dr`，通常用于标识与 `frontend` 服务相关的流量规则。
spec:
  host: frontend		# 定义了目标服务为 `frontend`，表示流量将路由到名为 `frontend` 的服务。
  subsets:		# 定义了服务的子集，流量可以根据标签路由到指定的子集。
  - name: v1	# 子集名称是 `v1`，通常与服务的版本相关。
    labels:		# 定义这个子集匹配的标签。
      version: v1	# 只有 Pod 拥有 `version: v1` 标签的才会被路由到这个子集。【这是我们之前部署时定义的标签】
```


### DestinationRule for Advertisement

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: advertisement-dr
spec:
  host: advertisement
  subsets:
  - name: v1
    labels:
      version: v1
```

+ **`metadata.name: advertisement-dr`**：DestinationRule 的名字为 `advertisement-dr`，与 `advertisement` 服务相关的流量规则。

+ **`host: advertisement`**：目标服务为 `advertisement`。

+ **`subsets`**：

  - **`name: v1`**：子集名称是 `v1`，用于版本 `v1` 的广告服务。

  - **`labels`**：使用标签 `version: v1` 进行匹配，只会路由到具有此标签的 `advertisement` 服务实例。

### DestinationRule for Forecast

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: forecast-dr
spec:
  host: forecast
  subsets:
  - name: v1
    labels:
      version: v1
```

+ **`metadata.name: forecast-dr`**：DestinationRule 的名字为 `forecast-dr`，与 `forecast` 服务相关的流量规则。

+ **`host: forecast`**：目标服务为 `forecast`。

+ **`subsets`**：

  - **`name: v1`**：子集名称是 `v1`，用于版本 `v1` 的 `forecast` 服务。

  - **`labels`**：使用标签 `version: v1`，只会路由到具有此标签的 `forecast` 服务实例。

## virtual-service-v1

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: frontend-route
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
        subset: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: advertisement-route
spec:
  hosts:
  - advertisement
  http:
  - route:
    - destination:
        host: advertisement
        subset: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: forecast-route
spec:
  hosts:
  - forecast
  http:
  - route:
    - destination:
        host: forecast
        subset: v1
```

> 这个 YAML 文件定义了三个 **VirtualService**，分别是 `frontend-route`、`advertisement-route` 和 `forecast-route`，它们用于将外部或内部流量路由到指定的服务及其子集（如 `v1` 版本）。
>
> **VirtualService** 是 Istio 中的一个重要资源，用来定义外部或内部的流量如何路由到服务。这些路由可以基于主机名、请求路径、HTTP 方法、端口等。
>
> 通过 **`host`** 和 **`gateways`**，Istio 控制着流量如何从外部或者内部被引导到正确的服务（如 `frontend`、`advertisement` 和 `forecast`）。
>
> **`subsets`** 参数与 **DestinationRule** 中定义的版本相结合，帮助控制流量被路由到哪个服务实例或哪个版本。
>
> 以下是详细解释：

### VirtualService for Frontend

```yaml
apiVersion: networking.istio.io/v1alpha3	# 使用 Istio 的 `v1alpha3` 网络资源版本。
kind: VirtualService	# 定义了一个 `VirtualService`，用于控制流量的路由规则。
metadata:
  name: frontend-route	# 这个 `VirtualService` 的名字是 `frontend-route`，用于定义 `frontend` 服务的流量路由规则。
spec:
  hosts:
  - "*"			# 匹配任何主机名的请求，适用于需要从外部通过任意域名访问的场景。可以根据需要替换成特定的域名。
  gateways:	# 指定使用的 Istio Gateway（在 `istio-system` 命名空间下的 `weather-gateway`）。
  - istio-system/weather-gateway	# 这个 `Gateway` 负责从外部接收 HTTP 请求并将其转发到服务网格内部。
  http:
  - match:
    - port: 80		# 匹配 HTTP 请求时使用的端口是 80。
    route:
    - destination:
        host: frontend	# 流量将被路由到 `frontend` 服务。
        subset: v1	# 流量将被路由到 `frontend` 服务的 `v1` 版本（根据之前的 `DestinationRule` 中定义的子集 `v1`）。
```

### VirtualService for Advertisement

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: advertisement-route
spec:
  hosts:
  - advertisement
  http:
  - route:
    - destination:
        host: advertisement
        subset: v1
```

**`metadata.name: advertisement-route`**：这个 `VirtualService` 的名字是 `advertisement-route`，用于定义 `advertisement` 服务的流量路由规则。

**`hosts: ["advertisement"]`**：流量将会被匹配到主机名为 `advertisement` 的请求。它通常在集群内部使用，或在外部域名解析为此服务时使用。

**`http`**：

- `route`
  - `destination`
    - **`host: advertisement`**：流量将被路由到 `advertisement` 服务。
    - **`subset: v1`**：流量将被路由到 `advertisement` 服务的 `v1` 版本。

### VirtualService for Forecast

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: forecast-route
spec:
  hosts:
  - forecast
  http:
  - route:
    - destination:
        host: forecast
        subset: v1
```

**`metadata.name: forecast-route`**：这个 `VirtualService` 的名字是 `forecast-route`，用于定义 `forecast` 服务的流量路由规则。

**`hosts: ["forecast"]`**：流量将会匹配主机名为 `forecast` 的请求，通常用于集群内部的服务间通信。

**`http`**：

- `route`
  - `destination`
    - **`host: forecast`**：流量将被路由到 `forecast` 服务。
    - **`subset: v1`**：流量将被路由到 `forecast` 服务的 `v1` 版本。

# 一个小技巧

可替换输入全部命令，使用`k`代替`kubectl`

![文件](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/gray/002.png)

# 创建各自的资源

+ 将访问请求路由到所有服务的`v1`版本（创建此资源并没有清除上一篇的gateway）

  ```shell
  k apply -f destination-rule-v1.yaml -n weather
  k apply -f virtual-service-v1.yaml  -n weather
  k get vs -n weather		
  ```

  ![文件](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/gray/003.png)

  ```shell
  k get vs -n weather forecast-route -o yaml
  ```

  ![文件](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/gray/005.png)图示（这张图片会显示的是当前请求的情况)
  ![文件](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/gray/004.png)

# 灰度发布

Istio能够提供基于百分数比例的流量控制,精确地将不同比例的流量分发给指定的版本。这种基于流量比例的路由策略用于典型的灰度发布场景

## 实战目标

用户需要软件能够根据不同的天气状况推荐合适的穿衣和运动信息。于是开发人员增加了`recommendation`新服务,并升级`forecast `服务到 v2版本来调用`recommendation` 服务。在新特性上线时，运维人员首先部署`forecast`服务的v2版本和`recommendation`服务，并对`forecast`服务的v2版本进行灰度发布

## 实战步骤

软件包位置

![文件](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/gray/006.png)

### 部署服务

+ 部署`recommendation`服务和forecast服务的v2版本

  ```shell
  k apply -f recommendation-all.yaml -n weather
  k apply -f forecast-v2-deployment.yaml -n weather
  ```

  ![文件](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/gray/007.png)

+ 验证部署成功

  ```shell
  k get po -n weather
  ```

  ![文件](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/gray/008.png)

+ 更新`forecast`服务的`DestinationRule`
  ```shell
  k apply -f forecast-v2-destination.yaml -n weather
  ```

  ![文件](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/gray/009.png)查看下发成功的配置，可以看到新增了v2版本subset的定义

  ```shell
  k get dr forecast-dr -o yaml -n weather
  ```

  ![文件](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/gray/010.png)

+ 配置`forecast`服务的路由规则
  ```shell
  k apply -f vs-forecast-weight-based-50.yaml -n weather
  ```

  ![文件](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/gray/011.png)

+ 查看`forecast`服务的`VirtualService`配置，其中的weight字段显示了相应服务的流量占比
  ```shell
  k get vs forecast-route -o yaml -n weather
  ```

  ![文件](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/gray/012.png)

+ 多次刷新即可看到流量图
  ![文件](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/gray/013.png)
  ![文件](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/gray/014.png)

+ 然后逐步增加`forecast`服务的v2版本的流量比例，直到流量全部被路由到v2版本
  此命令直接调制100

  ```shell
  k apply -f vs-forecast-weight-based-v2.yaml -n weather
  k get vs forecast-route -o yaml -n weather
  ```

  ![文件](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/gray/015.png)

+ 此时再重复刷新，则会发现每次都有推荐
  ![文件](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/gray/016.png)

> 保留forecast服务的老版本v1一段时间,在确认v2版本稳定后，删除老版本v1的所有资源，完成灰度发布



