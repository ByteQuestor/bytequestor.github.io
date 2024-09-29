---
title: docker常用命令
date: 2024-09-19 17:06:21
tags:
    - docker
cover: https://raw.gitmirror.com/ByteQuestor/picture/main/docker/docker.jpg
---

# docker 常用命令

## 强制删除

```shell
docker rm -f $id # 强制删除
```

## 从docker中复制文件

如果需要拿docker中的文件

先执行下列命令，更换其当前执行的指令，然后就可以复制

```shell
docker run -it --rm --entrypoint bash todo-backend:v2
```

复制文件

`docker cp $容器名字:$容器内文件位置 $本机位置`

```shell
docker cp confident_wozniak:/app/main ./main
```

## 挂载命令`-v `

**外有 覆盖里面，外无 把容器里的挂到外面，永远不会覆盖物理机的文件，挂载多个文件就用多个-v**

如果镜像内和本机上都没有这个文件，那么会挂一个目录出来

挂载应当先`touch $filename`一个出来挂载，本地无文件挂载适用于镜像中已经存在要挂的文件

```shell
docker run -itd -v $本机文件:$映射到容器里的文件 --name $起的名字 $镜像 bash

# 下面 的是将 本地 的 bash_log 和 容器的 /root/.bash_history 同步
docker run -itd -v bash_log:/root/.bash_history --name centos1 centos:7 bash
```

**实时**查看某个文件

```shell
tail -f $路径
```

