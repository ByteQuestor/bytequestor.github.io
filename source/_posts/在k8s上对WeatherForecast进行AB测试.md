---
title: 在k8s上对WeatherForecast进行AB测试
date: 2024-09-17 17:52:49
tags:
  - k8s
  - AB测试
  - istio
cover: https://raw.gitmirror.com/ByteQuestor/picture/main/WeatherForecast.png
---

Istio可以基于不同的请求内容将流量路由到不同的版本，这种策略一方面被应用于AB测试的场景中,另一方面配合基于流量比例的规则被应用于较复杂的灰度发布场景中例如组合条件路由

# 实战目标

> 在生产环境中同时上线了forecast服务的v1和v2版本，运维人员期望让不同的终端用户访问不同的版本
> 			例如:让使用Chrome浏览器的用户看到推荐信息，但让使用其他浏览器的用户看不到推荐信息。

文件位置

```shell
cd istioWeather/cloud-native-istio/10_canary-release/10.3/
```

# 实战演练

+ 配置`forecast`服务的路由规则

  ```shell
  k apply -f vs-forecast-header-based.yaml
  k get vs -n weather forecast-route -o yaml
  ```

  ![配置规则](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/ABTest/001.png)

+ 修改配置文件

  由于本机浏览器太乱，它的请求头带了很多浏览器

  + Edge浏览器
    ![Edge浏览器请求头](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/ABTest/002.png)

  + 谷歌浏览器
    ![谷歌浏览器请求头](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/ABTest/003.png)

  经过对比，决定通过Edg来判断不同的浏览器

  ```shell
  k edit vs forecast-route -n weather
  ```

  进入后将``Chorme`修改为`Edg`

+ 访问

  + 谷歌浏览器访问（看不到推荐信息）
    ![谷歌浏览器访问](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/ABTest/004.png)
  + Edge浏览器访问（能看到推荐信息）
    ![Edge浏览器访问](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/ABTest/005.png)
