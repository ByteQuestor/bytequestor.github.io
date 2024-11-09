---
title: 记录k8s开发运维的环境准备
date: 2024-11-09 18:12:57
tags:
    - k8s
    - python
description: 记录一下，在准备运维开发环境的时候踩的坑
cover: https://raw.gitmirror.com/ByteQuestor/picture/main/kubernetes/k8s.jpg
---

# 环境准备

##  软件包

[1.png](https://raw.gitmirror.com/ByteQuestor/picture/main/k8sPython/1.png)

## 配置本地`yum`仓库

[2.png](https://raw.gitmirror.com/ByteQuestor/picture/main/k8sPython/2.png)

```shell
mkdir /opt/python && cat >> /etc/yum.repos.d/local.repo << EOF
[python]
name=python
gpgcheck=0
enabled=1
baseurl=file:///opt/python
EOF
```

```shell
tar -zxf python-3.6.8.tar.gz && cp -rf ./python-3.6.8/* /opt/python
```

## 安装`python3`

```shell
yum install -y python3
```

注意：此时安装成功后，直接按照安装提示来会报错

[3.png](https://raw.gitmirror.com/ByteQuestor/picture/main/k8sPython/3.png)

[5.png](https://raw.gitmirror.com/ByteQuestor/picture/main/k8sPython/5.png)

```shell
python3 -m pip --version
```

[4.png](https://raw.gitmirror.com/ByteQuestor/picture/main/k8sPython/4.png)

因为`pip`的版本太低，升级一下即可解决

```shell
python3 -m pip install --upgrade pip
```

[6.png](https://raw.gitmirror.com/ByteQuestor/picture/main/k8sPython/6.png)

[7.png](https://raw.gitmirror.com/ByteQuestor/picture/main/k8sPython/7.png)

# pip换国内源

## 临时更换源（单次使用）

- 使用豆瓣源

  在使用pip安装软件包时，可以在命令中指定国内源的网址。例如，使用豆瓣源安装一个包，可以使用以下命令：

  - `pip3 install -i https://pypi.doubanio.com/simple/ [package_name]`
  - 其中，`[package_name]`是要安装的软件包的名称。这样每次安装包时，就会从豆瓣源进行下载，而不是默认的国外源。

- 使用清华大学源

  和豆瓣源类似，清华大学也提供了 Python 包的镜像源。命令格式为：

  - `pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple/ [package_name]`

### 二、永久更换源（修改配置文件）

+  `Linux` 和 `macOS` 系统

   - 创建或修改`~/.pip/pip.conf`文件（如果没有这个文件就创建一个）。可以使用以下命令来编辑这个文件：

     - `vim ~/.pip/pip.conf`（如果你熟悉 `vim` 编辑器）或者 `nano ~/.pip/pip.conf`（如果你更喜欢 `nano` 编辑器）。

   - 在文件中添加以下内容（以豆瓣源为例）：

     ```ini
     [global]
     index-url = https://pypi.doubanio.com/simple/
     [install]
     trusted-host = pypi.doubanio.com
     ```
     
   - 如果使用清华大学源，则将 `index - url` 和 `trusted - host` 中的内容替换为清华大学源的信息
     ```ini
     [global]
     index-url = https://pypi.tuna.tsinghua.edu.cn/simple/
     [install]
     trusted-host = pypi.tuna.tsinghua.edu.cn
     ```
   

+ Windows 系统

  - 在用户目录下（一般是 `C:\Users\[你的用户名]`）创建一个 `pip` 文件夹，然后在这个文件夹里创建一个 `pip.conf` 文件。


  - 可以使用记事本等文本编辑器打开这个文件，并添加和 `Linux/macOS` 系统类似的内容。例如，以豆瓣源为例：

    ```ini
  [global]
    index-url = https://pypi.doubanio.com/simple/
  [install]
    trusted-host = pypi.doubanio.com
  ```

