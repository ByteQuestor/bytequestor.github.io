---
title: 记一次配置win主机映射
date: 2024-10-22 18:03:16
tags:
    - windows
---
# 记一此配置Win主机`hosts`映射

因为主机一多起来以后，`IP`记起来就比较麻烦，所以修改一下映射

修改文件位置

```
C:\Windows\System32\drivers\etc\hosts
```

快捷键`win + R`，输入

```shell
127.0.0.1 controller
```

这样子，如果目标主机开启了`ssh`，就可以通过主机连接