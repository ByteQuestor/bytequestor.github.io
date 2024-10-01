---
title: 记一次手工在欧拉上部署OpenStack
date: 2024-09-30 21:44:37
tags:
    -  OpenStack
cover: https://raw.gitmirror.com//ByteQuestor/picture/main/openstack1.png
description: 一直对OpenStack一知半解，在欧拉上手工部署一下OpenStack，加深一下理解，这个笔记也适用于centos8
---

# 用到的资源

> `openEuler-22.03-LTS-x86_64-dvd.iso`
>
> 参考文档：
>
> https://docs.openstack.org/zh_CN/
>
> https://docs.openstack.org/install-guid

# 规划

+ `keystone`
+ `glance`
+ `placement`
+ `nova`
+ `neutron`
+ `dashboard`
+ `cinder`
+ `swift`

---

# 环境准备

## 虚拟环境准备

利用`VMWare`安装，与以往不同的是，要选`centos8`，内存推荐`5G`以上，`4核`，根据自身实际情况配置

欧拉基本上跟`Centos`一样，安装过程略，区别是**不能弱密码**（但是可以进去后修改为弱密码）：

```shell
passwd
# 然后输入新密码 000000
```

![修改密码](https://raw.gitmirror.com/ByteQuestor/picture/main/OpenStackByHandImg/001.jpg)

然后可以建个快照

## 系统环境准备

本次用的系统内核为`5.1`比较新，参考文档[OpenStack packages for RHEL and CentOS — Installation Guide documentation](https://docs.openstack.org/install-guide/environment-packages-rdo.html)

**注意：重启网卡的命令有区别**

```shell
nmcli c reload
nmcli c up eth0
```

---

> 完成本阶段的脚本【双节点执行，注意修改主机名的地方要改一下】

```shell
#!/bin/bash
hostnamectl set-hostname controller && bash	# 计算节点修改这里
systemctl stop firewalld && systemctl disable firewalld
echo "192.168.100.10 controller" >> /etc/hosts
echo "192.168.100.20 compute" >> /etc/hosts
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config; setenforce 0;
```

# 部署OpenStack基本环境

## 部署时间

```shell
yum install -y chrony
```

修改配置文件

```shell
vim /etc/chrony.conf
```

![时间](https://raw.gitmirror.com/ByteQuestor/picture/main/OpenStackByHandImg/002.jpg)

重启

```shell
systemctl restart chronyd
chronyc sources -v
```

> 完成本阶段的脚本【双节点执行】

```shell
#!/bin/bash
# 安装时间、编辑器、补全组件、网络工具，其中补全功能需要重连生效
yum install -y chrony vim bash-completion net-tools
# 修改配置文件，直接追加
# compute也可以用controller做时间服务器
sed -i "s/pool pool.ntp.org iburst/pool ntp6.aliyun.com iburst/g" /etc/chrony.conf 
echo "allow allow" >>/etc/chrony.conf 
echo "local stratum 10" >>/etc/chrony.conf
systemctl restart chronyd
chronyc sources -v
clock -w # 同步时间
date # 查看时间
```

## 安装opensatck

首先需要查看一下能够安装的`	opensatck`版本，注意：`dnf`等同于`yum`

```shell
dnf list | grep openstack
```

如下图：我们只能选择如下，因为没配置其他版本的源（如果要安装`queens`版就不行）

![关于yum源](https://raw.gitmirror.com/ByteQuestor/picture/main/OpenStackByHandImg/003.jpg)

安装命令（本次安装T版）

```shell
yum install -y openstack-release-train.noarch
ll /etc/yum.repos.d/	# 可以看到我们的仓库下面多了t版的yum源
yum install python3-openstackclient	# 这个计算节点不用装
```

出现如下算成功

![关于yum源](https://raw.gitmirror.com/ByteQuestor/picture/main/OpenStackByHandImg/004.jpg)

---

> 完成本阶段的脚本【双节点执行，注意部分计算节点不用装】但是装了也没事

```shell
#!/bin/bash
yum install -y openstack-release-train.noarch
yum install -y python3-openstackclient	# 这个计算节点不用装
```

## 安装数据库【controller节点】

```shell
yum install -y mariadb mariadb-server python3-PyMySQL
```

修改配置文件

参考文档：[SQL database for RHEL and CentOS — Installation Guide documentation (openstack.org)](https://docs.openstack.org/install-guide/environment-sql-database-rdo.html)

```shell
vim /etc/my.cnf.d/openstack.cnf
```

只需要修改监听地址为`0.0.0.0`，其他的默认即可

```ini
[mysqld]
bind-address = 0.0.0.0
default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```

写入配置后，重启并配置数据库自启动

```shell
systemctl enable mariadb.service
systemctl start mariadb.service
ss -tnl # 看到有3306表示成功
```

---

> 完成本阶段的脚本【controller节点执行】

```shell
#!/bin/bash
yum install -y mariadb mariadb-server python3-PyMySQL

# 定义文件路径
CONFIG_FILE="/etc/my.cnf.d/openstack.cnf"
# 写入配置内容
cat <<EOL > "$CONFIG_FILE"
[mysqld]
bind-address = 0.0.0.0
default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
EOL
# 输出完成信息
echo "配置已写入 $CONFIG_FILE"

# 启动数据库、配置自启动
systemctl enable mariadb.service
systemctl start mariadb.service
```

以下仍有需要手工的部分：（初始化数据库）

```shell
mysql_secure_installation
```

![关于yum源](https://raw.gitmirror.com/ByteQuestor/picture/main/OpenStackByHandImg/005.jpg)

## 安装消息队列【controller节点】

安装

```shell
yum install -y rabbitmq-server
```

启动，如果长时间卡着，直接`ctrl+c`.（看到`5672`端口表示成功）

```shell
systemctl enable --now rabbitmq-server.service
ss -tnl
```

新增一个用户

```shell
rabbitmqctl add_user openstack 000000
```

赋予读写权限

```shell
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

> 完成本阶段的脚本

```shell
#!/bin/bash
yum install -y rabbitmq-server
systemctl enable --now rabbitmq-server.service	# 这个地方要是卡住了需要手动搞一下
rabbitmqctl add_user openstack 000000
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

## 安装缓存【controller节点】

安装

```shell
yum install -y memcached python3-memcached
```

修改配置文件（将监听地址设置为`0.0.0.0`)

默认配置文件写了本地，但是在实验中，我们不只是有这本机需要留缓存，所以直接将`127.0.0.1`修改为``0.0.0.0`即可

```shell
vim /etc/sysconfig/memcached
```

启动（看到`11211`端口表示成功）

```shell
systemctl enable --now memcached.service
```

> 完成本阶段的脚本

```shell
#!/bin/bash
yum install -y memcached python3-memcached
sed -i "s/OPTIONS="-l 127.0.0.1,::1"/OPTIONS="-l 0.0.0.0,::1"/g" /etc/sysconfig/memcached
systemctl enable --now memcached.service
```

## 搭建OpenStack基本环境的所有脚本

> 在数据库部分，替换手工部分的还没写，此脚本还不能用

```shell
#!/bin/bash
# 环境准备
# 安装时间、编辑器、补全组件、网络工具，其中补全功能需要重连生效
yum install -y chrony vim bash-completion net-tools
# 修改配置文件，直接追加
# compute也可以用controller做时间服务器
sed -i "s/pool pool.ntp.org iburst/pool ntp6.aliyun.com iburst/g" /etc/chrony.conf 
echo "allow allow" >>/etc/chrony.conf 
echo "local stratum 10" >>/etc/chrony.conf
systemctl restart chronyd
chronyc sources -v
clock -w # 同步时间
date # 查看时间
yum install -y openstack-release-train.noarch
yum install -y python3-openstackclient	# 这个计算节点不用装

# -------------- 以下部分均不用在compute节点执行 ------------------
# 安装数据库
yum install -y mariadb mariadb-server python3-PyMySQL
# 定义文件路径
CONFIG_FILE="/etc/my.cnf.d/openstack.cnf"
# 写入配置内容
cat <<EOL > "$CONFIG_FILE"
[mysqld]
bind-address = 0.0.0.0
default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
EOL
# 输出完成信息
echo "配置已写入 $CONFIG_FILE"
# 启动数据库、配置自启动
systemctl enable mariadb.service
systemctl start mariadb.service

# 安装消息队列
yum install -y rabbitmq-server
systemctl enable --now rabbitmq-server.service	# 这个地方要是卡住了需要手动搞一下
rabbitmqctl add_user openstack 000000
rabbitmqctl set_permissions openstack ".*" ".*" ".*"

# 安装缓存
yum install -y memcached python3-memcached
sed -i "s/OPTIONS="-l 127.0.0.1,::1"/OPTIONS="-l 0.0.0.0,::1"/g" /etc/sysconfig/memcached
systemctl enable --now memcached.service
```

# 开始部署各组件

## keystone

## glance

## placement

## nova

## neutron

## dashboard

## cinder

## swift

