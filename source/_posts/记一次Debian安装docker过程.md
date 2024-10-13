---
title: 记一次Debian安装docker过程
date: 2024-10-13 11:02:07
tags:
- debian
- docker
description: 在物理机Debian上安装docker，记录下过程
cover: https://raw.gitmirror.com/ByteQuestor/picture/main/debian.png
---
# 完整步骤：

## 删除旧的 Docker 源（如果有）
如果之前已经添加过 Docker 源，但遇到了错误，可以先删除旧的源文件：

```shell
sudo rm /etc/apt/sources.list.d/docker.list
```

## 创建 Docker 源文件
创建并编辑 Docker 源文件` /etc/apt/sources.list.d/docker.list`，以确保使用正确的源。根据你的 Debian 版本进行配置：

+ 如果使用阿里云镜像源，并且是 `Debian 12 (bookworm)`，使用以下内容：
  ```shell
  deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.aliyun.com/docker-ce/linux/debian bookworm stable
  ```

+ 如果是 Debian 11 (bullseye)，则使用以下内容：
  ```shell
  deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.aliyun.com/docker-ce/linux/debian bullseye stable
  ```

+ 或者 Docker 官方源，可以使用：
  ```shell
  deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian bookworm stable
  ```

## 添加 Docker GPG 公钥
为了确保能够验证 Docker 仓库的签名，需要导入 Docker 的 GPG 公钥：

+ 导入公钥
  ```shell
  sudo mkdir -p /etc/apt/keyrings
  # 注意分开执行
  curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
  ```

+ 更新系统包列表
  现在更新系统包列表以确保使用最新的源：

  ```shell
  sudo apt-get update
  ```

  

+ 安装 Docker 相关组件
  最后，安装 Docker 及其依赖组件：

  ```shell
  sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
  ```

+ 验证 Docker 安装
  安装完成后，使用以下命令来验证 Docker 是否安装成功：

  ```shell
  sudo docker --version
  sudo docker run hello-world
  ```

# 总结：

这套步骤包括了配置正确的 Docker 源、导入 GPG 公钥、更新包列表并安装 Docker。通过这些步骤，能够顺利在 Debian 系统上安装 Docker。