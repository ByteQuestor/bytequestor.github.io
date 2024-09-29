---
title: 在k8s上对WeatherForecast进行多服务灰度发布
date: 2024-09-17 21:11:13
tags:
    - k8s
    - 灰度发布
    - istio
cover: https://raw.gitmirror.com/ByteQuestor/picture/main/WeatherForecast.png
---

在一些系统中往往需要对同一应用下的多个组件同时进行灰度发布，这时需要将这些服务串联起来。例如，只有测试账号才能访问这些服务的新版本并进行功能测试;其他用户只能访问老版本，不能使用新功能

# 实战目标

运维人员对 `frontend`和`forecast`两个服务同时进行灰度发布，`frontend`服务新增v2版本，界面的按钮变为蓝色，`forecast`服务新增v2版本，增加了推荐信息。测试人员在用账号`tester`访问天气应用时会看到这两个服务的v2版本，其他用户只能看到这两个服务的v1版本，不会出现服务版本交叉调用的情况

# 实战演练

> 参照 10.2.2节在集群中部署`recommendation`服务和`forecast`服务的 v2版本，并更新`forecast` 服务的 `DestinationRule`,在 `DestinationRule` 中增加对 v2 版本 `subset `的定义`【前几篇已经完成】`
>
> 按照10.4.2节的前两个步骤在集群中部署`frontend`服务的v2版本，并更新`frontend`服务的 `DestinationRule`，增加对v2版本 `subset `的定义

+ 文件位置
  ```shell
  cd istioWeather/cloud-native-istio/10_canary-release/10.5/
  ```

+ 对**非入口服务**`forecast`使用`match`的`sourceLabels`创建`VirtualService`
  ```shell
  k apply -f vs-forecast-multiservice-release.yaml -n weather
  ```

  + 文件解析
    【只有带“version:v2”标签的Pod实例的流量，才能进入forecast服务的新版本v2实例】

    ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: forecast-route
    spec:
      hosts:
      - forecast
      http:
      - match:
        - sourceLabels:
            version: v2
        route:
        - destination:
            host: forecast
            subset: v2
      - route:
        - destination:
            host: forecast
            subset: v1
    ```

+ 对**入口服务**`frontend`设置基于访问内容的规则

  ```shell
  k apply -f vs-frontend-multiservice-release.yaml -n weather
  ```

  + 文件解析
    【对于测试账户（即cookie有“user=tester”的请求）,istio会将流量导入`frontend`服务的v2版本的pod实例；这些流量在访问`forecast`服务时会被路由到`forecast`服务的新版本v2的Pod实例】
    【对于其它用户请求,istui会将流量导入`frontend`服务的v1版本的Pod实例；这些流量在访问`forecast`服务时会被路由到`forecast`服务的老版本v1的Pod实例】

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
        - headers:
            cookie:
              regex: ^(.*?;)?(user=tester)(;.*)?$
        route:
        - destination:
            host: frontend
            subset: v2
      - route:
        - destination:
            host: frontend
            subset: v1
    ```

  

  

  # 访问效果

  + 登录位置
    ![登录](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/userTest/001.png)
  + 未登录界面
    ![未登录](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/userTest/noUser.png)
  + 登录后界面（前台）
    ![登录](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/userTest/user.png)
  + 登陆后界面（推荐）
    ![登录](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/userTest/004.png)
  + 流量走线
    ![登录](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/userTest/003.png)
  + 流量走线（先不登陆访问了后再登录的）
    ![登录](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/userTest/005.png)
