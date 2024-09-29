---
title: 在k8s上对WeatherForecast进行组合条件路由
date: 2024-09-17 20:39:00
tags:
description: 一些复杂的灰度发布场景需要使用前面两种路由规则的组合形式
cover: https://raw.gitmirror.com/ByteQuestor/picture/main/WeatherForecast.png
---

# 实战目标

> 在生产环境中同时上线了`frontend`服务的v1和v2版本,**v1版本的按钮颜色是绿色的，v2版本的按钮颜色是蓝色的**。运维人员期望使用`Android`操作系统的一半用户看到的是v1版本，另一半用户看到的是v2版本;使用其他操作系统的用户看到的总是1版本

# 实战演练

+ 切换到目标目录
  ```shell
  cd istioWeather/cloud-native-istio/10_canary-release/10.4/
  ```

+ 部署`frontend`服务的v2版本
  ```shell
  k apply -f frontend-v2-deployment.yaml -n weather
  ```

  验证
  ```shell
  k get po -n weather
  ```

  ![部署验证](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/graypro/001.png)

+ 更新`frontend`服务的`DestinationRule`
  ```shell
  k apply -f frontend-v2-destination.yaml -n weather
  ```

  查看下发的`DestinationRule`，发现新增了v2版本的subset的定义
  ```shell
  k get dr frontend-dr -o yaml -n weather
  ```

  ![部署验证](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/graypro/002.png)

+ 配置`frontend`服务的路由策略
  ```shell
  k apply -f vs-frontend-combined-condition.yaml -n weather
  ```

+ 效果
  用安卓手机多次查询前台页面，一半概率是绿色按钮，一半概率是蓝色按钮。在win上查询前台界面始终是绿色按钮
  **小改一下：**用Edg浏览器一半绿色一半蓝色，用谷歌始终是绿色

  ```shell
  k edit vs frontend-route -n weather
  ```

  ![部署验证](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/graypro/003.png)

  

+ 访问

  + Edg会有概率出现两种颜色
    ![部署验证](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/graypro/004.png)
    ![部署验证](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/graypro/005.png)
  +  谷歌始终是绿色
    ![部署验证](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/graypro/006.png)

  
