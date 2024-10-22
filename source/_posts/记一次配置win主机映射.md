---
title: 记一次配置win主机映射
date: 2024-10-22 18:03:16
tags:
    - windows
cover: https://raw.gitmirror.com/ByteQuestor/picture/main/windows.jpg
description: 最近偏向使用 ssh@连接主机，但是ip又记不住，于是修改主机映射，将ip编为我任意记的
---
# 记一此配置Win主机`hosts`映射

因为主机一多起来以后，`IP`记起来就比较麻烦，所以修改一下映射

修改文件位置，反正就是管理员运行修改工具，然后进行修改，保存到原来的位置即可

```
C:\Windows\System32\drivers\etc\hosts
```

快捷键`win + R`，输入

```shell
127.0.0.1 controller
```

这样子，如果目标主机开启了`ssh`，就可以通过主机连接