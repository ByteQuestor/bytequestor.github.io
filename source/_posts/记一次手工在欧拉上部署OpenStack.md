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
> https://docs.openstack.org/install-guide

# 环境准备

## 虚拟环境准备

利用`VMWare`安装，与以往不同的是，要选`centos8`，内存推荐`5G`以上，`4核`，根据自身实际情况配置

| 主机         | `IP`(单网卡NAT)  | 配置             |
| ------------ | ---------------- | ---------------- |
| `controller` | `192.168.100.10` | `4v_8G_100G`     |
| `compute`    | `192.168.100.20` | `4V_5G_100G_50G` |

网关：`192.168.100.2`

欧拉基本上跟`Centos`一样，安装过程略，区别是**不能弱密码**（但是可以进去后修改为弱密码）：

```shell
passwd
# 然后输入新密码 000000
```

![修改密码](https://raw.gitmirror.com/ByteQuestor/picture/main/OpenStackByHandImg/001.jpg)

然后可以建个快照

## 系统环境准备

本次用的系统内核为`5.1`比较新，参考文档[OpenStack packages for RHEL and CentOS — Installation Guide documentation](https://docs.openstack.org/install-guide/environment-packages-rdo.html)

### 配置网卡

留下如下配置即可：

![网卡配置](https://raw.gitmirror.com/ByteQuestor/picture/main/OpenStackByHandImg/new_1.png)

**注意：重启网卡的命令有区别**

```shell
nmcli c reload
nmcli c up eth0
```

注意：此阶段需要能`ping`通百度表示成功

### 更改主机名

**controller**

```shell
hostnamectl set-hostname controller && bash
```

**compute**

```shell
hostnamectl set-hostname compute && bash
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
# 部署`OpenStack`基本环境

> 文档内`Environment`

## 部署时间

```shell
yum install -y chrony vim bash-completion net-tools
```

修改配置文件

```shell
vim /etc/chrony.conf
```

```ini
pool ntp6.aliyun.com iburst
```

![时间](https://raw.gitmirror.com/ByteQuestor/picture/main/OpenStackByHandImg/002.jpg)

重启

```shell
systemctl restart chronyd
chronyc sources -v
date
clock -w
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
systemctl enable --now mariadb.service
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

**以下仍有需要手工的部分：（初始化数据库）**

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

---

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

示例：

```ini
PORT="11211"
USER="memcached"
MAXCONN="1024"
CACHESIZE="64"
OPTIONS="-l 0.0.0.0,::1"
```

启动（看到`11211`端口表示成功）

```shell
systemctl enable --now memcached.service
ss -tnl
```

---

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

参考文档：[Install OpenStack services — Installation Guide documentation](https://docs.openstack.org/install-guide/openstack-services.html)

## keystone

登录数据库

```shell
mysql
```

为`keystone`创建数据库，并为`keystone`授权，注意：以往设置的`000000`，本次设置的`keystone123`

```shell
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'keystone123';
```

验证：

```shell
show create database keystone; # 仅作验证
```

安装（`dnf`和`yum`完全等价）

```shell
yum install -y openstack-keystone httpd python3-mod_wsgi
```

然后会有一个配置文件，只需要取需要的即可，先把原来的备份

```shell
cp /etc/keystone/keystone.conf{,.bak}
```

然后从 `/etc/keystone/keystone.conf.bak` 文件中筛选出所有不为空且不以 `#` 开头的行（也就是忽略掉**空行**和**注释行**）

然后再去编辑就会很好看

```shell
grep -Ev "^$|#" /etc/keystone/keystone.conf.bak > /etc/keystone/keystone.conf
vim /etc/keystone/keystone.conf
# 解释
# -E 是正则表达式，v是反向匹配（不输出匹配到的内容）
# ^$：匹配空行。^ 表示行的开始，$ 表示行的结束，因此 ^$ 表示没有字符的行。
# |：表示“或”，用于连接多个条件。
# #：匹配以 # 开头的行，这通常用于注释行。
```

添加配置内容

```ini
[database]
connection = mysql+pymysql://keystone:keystone123@controller/keystone
[token]
provider = fernet
```

同步数据库

```shell
su -s /bin/sh -c "keystone-manage db_sync" keystone
```

验证方法就是看`keystone`数据库里有无新出现的表，其他的组件也是如此验证的

初始化密钥库

```shell
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```

引导身份服务（指定密码为`000000`）即可

```shell
keystone-manage bootstrap --bootstrap-password 000000 \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```

配置`HttpService`服务

```shell
vi /etc/httpd/conf/httpd.conf
```

进入后，修改`97`行

```ini
ServerName controller
```

软连接`Keystone` 的 `WSGI `配置文件到`Apache HTTP` 服务器的配置目录

`-s`表示软连接，软链接我们修改一个位置其他位置也会同步修改

```shell
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
```

启动，看到`5000`端口说明成功（如果出现权限问题，可能是`SELinux 策略`)

```shell
systemctl enable --now httpd.service
```

编写环境变量文件

```shell
vim /etc/keystone/admin-openrc.sh
```

内容（修改密码为`000000`，其他的不动，也可以直接配置下面那一个）

```shell
#!/bin/bash
export OS_USERNAME=admin
export OS_PASSWORD=000000
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
```

验证有无生效（`export`输出环境变量）

```shell
source /etc/keystone/admin-openrc.sh
export
```

创建域、项目、用户、角色

```shell
openstack domain create --description "An Example Domain" example
openstack project create --domain default --description "Service Project" service
```

创建 `OpenStack` 客户端环境脚本(其实和之前那一步一样的，只是更详细点，可以直接配置这个)

```shell
vim /etc/keystone/admin-openrc.sh
```

内容

```shell
#!/bin/bash
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=000000
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```

验证（返回`value`表示成功）

```shell
openstack token issue
```

提示：至此是没有问题的

---

> 本阶段脚本（暂时不能一键执行，放这里只是为了方便粘贴）

```shell
mysql -u root -p # 注意数据库没密码，直接回车
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'keystone123';
quit
dnf install -y openstack-keystone httpd python3-mod_wsgi
cp /etc/keystone/keystone.conf{,.bak}
grep -Ev "^$|#" /etc/keystone/keystone.conf.bak > /etc/keystone/keystone.conf
# 对配置文件进行一系列添加
su -s /bin/sh -c "keystone-manage db_sync" keystone
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
keystone-manage bootstrap --bootstrap-password 000000 \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
echo "ServerName controller" >> /etc/httpd/conf/httpd.conf
# 定义文件路径
CONFIG_FILE="/etc/keystone/admin-openrc.sh"
# 写入配置内容
cat <<EOL > "$CONFIG_FILE"
#!/bin/bash
export OS_USERNAME=admin
export OS_PASSWORD=000000
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
EOL
# 赋予执行权限
chmod +x "$CONFIG_FILE"
# 输出完成信息
echo "配置已写入 $CONFIG_FILE"
source /etc/keystone/admin-openrc.sh
openstack domain create --description "An Example Domain" example
openstack project create --domain default --description "Service Project" service
# 验证一下能否获取TOKEN
CONFIG_FILE="/etc/keystone/admin-openrc.sh"
cat <<EOL > "$CONFIG_FILE"
#!/bin/bash
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=000000
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
EOL
source /etc/keystone/admin-openrc.sh
openstack token issue
```

## glance

创建数据库并赋予权限

```shell
mysql
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'glance123';
quit
```

如果过程中重启了虚拟机，需要重新导入环境变量（也可以写入`~/.bashrc`，如果是做一次实验的话就不用了）

创建`glance`用户

```shell
openstack user create --domain default --password glance glance # 前一个glance是密码，后一个glance是用户
```

给`glance`用户和`service`项目增加管理员角色

```shell
openstack role add --project service --user glance admin
```

创建`service`实体

```shell
openstack service create --name glance  --description "OpenStack Image" image
```

创建镜像服务（Glance）的端点

1. **public**：公开的端点，允许外部用户访问镜像服务。
2. **internal**：内部端点，仅供内部服务或项目使用。
3. **admin**：管理端点，供管理员使用，通常具有更高的权限。

这些端点使得不同角色的用户能够以不同方式访问镜像服务

```shell
openstack endpoint create --region RegionOne image public http://controller:9292
openstack endpoint create --region RegionOne image internal http://controller:9292
openstack endpoint create --region RegionOne image admin http://controller:9292
```

安装`glance`安装包

```shell
yum install -y openstack-glance
```

和`keystone`一样的操作，备份原来的配置文件，然后去掉没必要的进行配置

```shell
cp /etc/glance/glance-api.conf{,.bak}
grep -Ev "^$|#" /etc/glance/glance-api.conf.bak > /etc/glance/glance-api.conf
```

添加配置内容（注意：对应的配置必须放置在对应的块下面）

```ini
[DEFAULT]
log_file = /var/log/glance/glance-api.log

[database]
connection = mysql+pymysql://glance:glance123@controller/glance

[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/

[keystone_authtoken]
www_authenticate_uri  = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = glance

[paste_deploy]
flavor = keystone
```

同步 Glance（镜像服务）的数据库

```shell
su -s /bin/sh -c "glance-manage db_sync" glance
```

启动服务
```shell
systemctl enable --now openstack-glance-api.service
```

查看端口验证（出现`9292`算成功）

```shell
ss -tnl
```

查看日志验证

```shell
tail -f /var/log/glance/glance-api.log
```

服务验证（必须做）建议从浏览器下载再上传，通过任何办法上传至虚拟机

```shell
wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
```

创建一个镜像

```shell
glance image-create --name "cirros" \
--file cirros-0.4.0-x86_64-disk.img \
--disk-format qcow2 --container-format bare \
--visibility public

# 验证
glance image-list
```

注意：至此是没有任何问题

---

> 完成此阶段的脚本

```shell
比完赛再写
```

## placement

创建数据库并赋予权限

```shell
mysql
CREATE DATABASE placement;
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'placement123';
quit
```

创建`placement`用户和密码

```shell
openstack user create --domain default --password placement placement # 前面是密码，后面是用户
```

将 `placement` 用户分配给 `service` 项目，并赋予 `admin` 角色（这个是`scheduler`和`conductor`服务起不来的原因）

```shell
openstack role add --project service --user placement admin
```

创建一个名为 `placement` 的服务

```shell
openstack service create --name placement --description "Placement API" placement
```

为 **Placement** 服务创建端点（直接复制执行即可，官方文档的）

```shell
openstack endpoint create --region RegionOne placement public http://controller:8778
openstack endpoint create --region RegionOne placement internal http://controller:8778
openstack endpoint create --region RegionOne placement admin http://controller:8778
```

安装

```shell
yum install -y openstack-placement-api
```

筛选出参数

```shell
cp /etc/placement/placement.conf{,.bak}
```

```shell
grep -Ev "^$|#" /etc/placement/placement.conf.bak > /etc/placement/placement.conf
```

编辑文件：`vim /etc/placement/placement.conf`

添加如下配置

```ini
[api]
auth_strategy = keystone

[keystone_authtoken]
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = placement

[placement_database]
connection = mysql+pymysql://placement:placement123@controller/placement
```

同步数据库

```shell
[root@controller ~]# su -s /bin/sh -c "placement-manage db sync" placement
# 报以下回显，不用管
/usr/lib/python3.9/site-packages/pymysql/cursors.py:170: Warning: (1280, "Name 'alembic_version_pkc' ignored for PRIMARY key.")
  result = self._query(query)
```

重启`httpd`之前，需要注意`2.4`版本以上**重启服务器时需要对文件进行同步**

```shell
[root@controller ~]# httpd -v
Server version: Apache/2.4.51 (Unix)
Server built:   May  6 2024 00:00:00
```

```shell
vim /etc/httpd/conf.d/00-placement-api.conf
```

![关于文件同步](https://raw.gitmirror.com/ByteQuestor/picture/main/OpenStackByHandImg/006.jpg)

重启`httpd`

```shell
systemctl restart httpd
```

`ss -tnl`查看出现`8778`端口表示成功

验证

```shell
placement-status upgrade check
```

```shell
[root@controller ~]# placement-status upgrade check
+----------------------------------+
| Upgrade Check Results            |
+----------------------------------+
| Check: Missing Root Provider IDs |
| Result: Success                  |
| Details: None                    |
+----------------------------------+
| Check: Incomplete Consumers      |
| Result: Success                  |
| Details: None                    |
+----------------------------------+
```

注意：至此没有问题

---

## nova

### 控制节点

创建数据库并赋予权限

```shell
mysql
CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;

GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'nova123';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'nova123';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'nova123';
```

创建用户（很慢，稍安勿躁）

```shell
openstack user create --domain default --password nova nova # 前密码后用户
```

赋予管理员权限

```shell
openstack role add --project service --user nova admin
```

创建服务

```shell
openstack service create --name nova --description "OpenStack Compute" compute
```

为服务创建`API`端点

```shell
openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1
openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1
openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1
```

安装组件

**openstack-nova-api**：提供 Nova 的 REST API 接口，处理用户请求并与其他组件进行通信。

**openstack-nova-conductor**：负责管理计算节点的状态和任务，协调数据库操作，确保任务的分发和执行。

**openstack-nova-novncproxy**：提供 VNC 代理服务，使用户能够通过浏览器访问虚拟机的控制台，方便远程管理。

**openstack-nova-scheduler**：负责选择适当的计算节点来运行实例，基于资源可用性和调度策略做出决策。

```shell
yum install -y openstack-nova-api openstack-nova-conductor openstack-nova-novncproxy openstack-nova-scheduler
```

过滤参数

```shell
cp /etc/nova/nova.conf{,.bak}
```

```shell
grep -Ev "^$|#" /etc/nova/nova.conf.bak > /etc/nova/nova.conf
```

`controller`节点填充参数（8项）

```ini
[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:000000@controller:5672/
my_ip = 192.168.100.10	# 注意这里是本机的地址
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver
log_file = /var/log/nova/nova.log

[api]
auth_strategy = keystone

[api_database]
connection = mysql+pymysql://nova:nova123@controller/nova_api

[database]
connection = mysql+pymysql://nova:nova123@controller/nova

[glance]
api_servers = http://controller:9292

[keystone_authtoken]
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = nova

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = placement

[vnc]
enabled = true
server_listen = $my_ip
server_proxyclient_address = $my_ip
```

`controller`节点同步数据库

```shell
su -s /bin/sh -c "nova-manage api_db sync" nova
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
su -s /bin/sh -c "nova-manage db sync" nova
su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
```

启动（启动不了检查配置文件和`placement`有无漏掉的步骤）

```shell
systemctl enable --now \
    openstack-nova-api.service \
    openstack-nova-scheduler.service \
    openstack-nova-conductor.service \
    openstack-nova-novncproxy.service
```

方便重启

```shell
systemctl restart \
    openstack-nova-api.service \
    openstack-nova-scheduler.service \
    openstack-nova-conductor.service \
    openstack-nova-novncproxy.service
```



`vnc`的端口是`6080`，还有个`8774`看到就算成功（暂时不理解，这些端口代表的服务）

### 计算节点

安装

```shell
yum install -y openstack-nova-compute
```

配置（配置较多，要小心）

```shell
cp /etc/nova/nova.conf{,.bak}
```

```shell
grep -Ev "^$|#" /etc/nova/nova.conf.bak > /etc/nova/nova.conf
```

修改配置文件`vim /etc/nova/nova.conf`

详细内容

```ini
[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:000000@controller
compute_driver=libvirt.LibvirtDriver	# 【1坑】这一行要去.bak里面找到并复制过来
my_ip = 192.168.100.20		# 注意这里是本机的地址
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver
log_file = /var/log/nova/nova-compute.log

[api]
auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = nova

[vnc]
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://192.168.100.10:6080/vnc_auto.html

[glance]
api_servers = http://controller:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service   
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = placement
```

注意：在`centos`中，需要修改`[libvirt]`，但是在欧拉系统中分情况；查看虚拟机是否支持虚拟化，如果是`0`则需要配置`[libvirt]`

```shell
egrep -c '(vmx|svm)' /proc/cpuinfo
```

添加如下内容`注意：如果不显示为0则不需要配置`

```ini
[libvirt]
virt_type = qemu
```

重启`注意：`不可以直接起，先启动`libvirtd`，再启动`openstack-nova-compute`

```shell
systemctl enable libvirtd.service openstack-nova-compute.service
systemctl start libvirtd
systemctl start libvirtd.service openstack-nova-compute.service
```

`报错`	解决：显示没用目录，直接创建一个`/usr/lib/python3.9/site-packages/instances`，并给`nova`权限去管理

```shell
[root@compute ~]# tail -f  /var/log/nova/nova-compute.log 
2024-10-02 09:26:21.603 2987 ERROR nova.compute.manager   File "/usr/lib/python3.9/site-packages/nova/compute/resource_tracker.py", line 871, in update_available_resource
2024-10-02 09:26:21.603 2987 ERROR nova.compute.manager     resources = self.driver.get_available_resource(nodename)
2024-10-02 09:26:21.603 2987 ERROR nova.compute.manager   File "/usr/lib/python3.9/site-packages/nova/virt/libvirt/driver.py", line 8025, in get_available_resource
2024-10-02 09:26:21.603 2987 ERROR nova.compute.manager     disk_info_dict = self._get_local_gb_info()
2024-10-02 09:26:21.603 2987 ERROR nova.compute.manager   File "/usr/lib/python3.9/site-packages/nova/virt/libvirt/driver.py", line 6483, in _get_local_gb_info
2024-10-02 09:26:21.603 2987 ERROR nova.compute.manager     info = libvirt_utils.get_fs_info(CONF.instances_path)
2024-10-02 09:26:21.603 2987 ERROR nova.compute.manager   File "/usr/lib/python3.9/site-packages/nova/virt/libvirt/utils.py", line 403, in get_fs_info
2024-10-02 09:26:21.603 2987 ERROR nova.compute.manager     hddinfo = os.statvfs(path)
2024-10-02 09:26:21.603 2987 ERROR nova.compute.manager FileNotFoundError: [Errno 2] No such file or directory: '/usr/lib/python3.9/site-packages/instances'
2024-10-02 09:26:21.603 2987 ERROR nova.compute.manager 
```

解决问题

```shell
mkdir /usr/lib/python3.9/site-packages/instances
chown -R root.nova /usr/lib/python3.9/site-packages/instances
systemctl restart openstack-nova-compute.service
```

![报错解决](https://raw.gitmirror.com/ByteQuestor/picture/main/OpenStackByHandImg/007.jpg)

此时依然有问题（没有发现计算节点的主机），虽然是之前错写了`192.168.100.10`，但是写`192.168.100.20`也会出现同样的错误

解决问题

在`controller`上查看一下，如何有哪些计算节点

```shell
openstack compute service list --service nova-compute
```

如下

```shell
[root@controller ~]# openstack compute service list --service nova-compute
+----+--------------+---------+------+---------+-------+----------------------------+
| ID | Binary       | Host    | Zone | Status  | State | Updated At                 |
+----+--------------+---------+------+---------+-------+----------------------------+
|  9 | nova-compute | compute | nova | enabled | up    | 2024-10-29T11:07:47.000000 |
+----+--------------+---------+------+---------+-------+----------------------------+
```

同步。主机发现【`controller`执行】

```shell
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
```

![报错解决](https://raw.gitmirror.com/ByteQuestor/picture/main/OpenStackByHandImg/008.jpg)

再次重启【`compute`执行】

```shell
systemctl restart openstack-nova-compute.service
```

![报错解决](https://raw.gitmirror.com/ByteQuestor/picture/main/OpenStackByHandImg/009.jpg)

验证

```shell
openstack compute service list
```

以下`up`表示成功

```shell
+----+----------------+------------+----------+---------+-------+----------------------------+
| ID | Binary         | Host       | Zone     | Status  | State | Updated At                 |
+----+----------------+------------+----------+---------+-------+----------------------------+
|  4 | nova-conductor | controller | internal | enabled | up    | 2024-10-02T01:40:13.000000 |
|  7 | nova-scheduler | controller | internal | enabled | up    | 2024-10-02T01:40:15.000000 |
| 10 | nova-compute   | compute    | nova     | enabled | up    | 2024-10-02T01:40:19.000000 |
+----+----------------+------------+----------+---------+-------+----------------------------+
```

注意：执行到此没有问题

## neutron

### 控制节点

数据库创建、授权

```shell
mysql
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'neutron123';
```

创建用户

```shell
openstack user create --domain default --password neutron neutron
```

给用户赋予`admin`角色

```shell
openstack role add --project service --user neutron admin
```

创建网络服务`Neutron`的服务条目

```shell
openstack service create --name neutron --description "OpenStack Networking" network
```

为这个服务创建三个端点（分别是：内网、公网、广域网）

```shell
openstack endpoint create --region RegionOne network public http://controller:9696
openstack endpoint create --region RegionOne network internal http://controller:9696
openstack endpoint create --region RegionOne network admin http://controller:9696
```

#### 配置 server 组件

安装相应的服务

```shell
# yum install -y openstack-neutron openstack-neutron-ml2 openstack-neutron-openvswitch ebtables
yum install -y openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables
# yum install -y openstack-neutron-linuxbridge.noarch
```

有一个报错：（后面会解决）

```shell
Couldn't write '1' to 'net/bridge/bridge-nf-call-ip6tables', ignoring: No such file or directory
Couldn't write '1' to 'net/bridge/bridge-nf-call-iptables', ignoring: No such file or directory
```

备份、过滤

```shell
cp /etc/neutron/neutron.conf{,.bak}
grep -Ev "^$|#" /etc/neutron/neutron.conf.bak > /etc/neutron/neutron.conf
```

添加配置信息`vim /etc/neutron/neutron.conf`

```ini
[DEFAULT]
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true
transport_url = rabbit://openstack:000000@controller
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

[database]
connection = mysql+pymysql://neutron:neutron123@controller/neutron

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = neutron
password = neutron

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp

[nova]
# ...
auth_url = http://controller:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = nova
password = nova
```

#### 配置模块化第 2 层

备份、过滤

```shell
cp /etc/neutron/plugins/ml2/ml2_conf.ini{,.bak}
grep -Ev "^$|#" /etc/neutron/plugins/ml2/ml2_conf.ini.bak > /etc/neutron/plugins/ml2/ml2_conf.ini
```

添加配置信息`vim /etc/neutron/plugins/ml2/ml2_conf.ini`

```ini
[ml2]
# ... 
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge,l2population
extension_drivers = port_security

[ml2_type_flat]
# ...
flat_networks = extnal

[ml2_type_vxlan]
# ...
vni_ranges = 1:10000

[securitygroup]
# ...
enable_ipset = true
```

#### 配置 Open vSwitch 代理

备份、过滤

```shell
cp /etc/neutron/plugins/ml2/linuxbridge_agent.ini{,.bak}
grep -Ev "^$|#" /etc/neutron/plugins/ml2/linuxbridge_agent.ini.bak > /etc/neutron/plugins/ml2/linuxbridge_agent.ini
```

添加配置信息`vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini`

```ini
[linux_bridge]
bridge_interface_mappings = extnal:eth0 # 注意要和上一步定义的一样，而且还要看自己的网卡名字

[vxlan]
enable_vxlan = true
local_ip = 192.168.100.10
l2_population = true

[securitygroup]
# ...
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

配置内核开放（注意：手工在结尾处加上`EOF`表示结束）

```shell
tee -a /etc/sysctl.conf <<EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
```

加载模块并使其生效

```shell
modprobe br_netfilter
sysctl -p
```

#### 配置第 3 层代理

因为就添加一条，所以就不过滤了

```shell
vim /etc/neutron/l3_agent.ini
```

添加内容

```ini
[DEFAULT]
interface_driver = linuxbridge
```

#### 配置 DHCP 代理

因为就添加一条，所以就不过滤了

```shell
vim /etc/neutron/dhcp_agent.ini
```

添加内容

```ini
[DEFAULT]
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
```

注意：至此，没有出现分歧现象

####  配置元数据代理

只添加一条，无需过滤

```shell
vim /etc/neutron/metadata_agent.ini
```

添加内容

```ini
[DEFAULT]
nova_metadata_host = controller
metadata_proxy_shared_secret = wzy # 自定义一个标记，要牢记，后面要用
```

#### 配置 Compute 服务以使用 Networking 服务

向`nova`添加配置信息`vim /etc/nova/nova.conf`

```ini
[neutron]
auth_url = http://controller:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = neutron
password = neutron
service_metadata_proxy = true
metadata_proxy_shared_secret = wzy	# 注意对应
```

#### 完成安装

`networking `服务初始化脚本需要一个指向 `ML2 `插件配置的符号链接文件

```shell
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
```

同步数据库

```shell
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

重启`nova-api`服务（为了生成我们刚刚添加的`neutron`配重块）

```shell
systemctl restart openstack-nova-api.service
```

启动 Networking 服务并配置自启动

```shell
systemctl enable neutron-server.service \
  neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service neutron-l3-agent.service
# 方便重启
systemctl restart neutron-server.service \
  neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service neutron-l3-agent.service
```

报错：（但是最后成功了，不用管）

```shell
==> /var/log/neutron/metadata-agent.log <==
2024-10-29 19:53:27.330 23645 ERROR neutron.agent.metadata.agent   File "/usr/lib/python3.9/site-packages/oslo_messaging/_drivers/amqpdriver.py", line 661, in _send
2024-10-29 19:53:27.330 23645 ERROR neutron.agent.metadata.agent     result = self._waiter.wait(msg_id, timeout,
2024-10-29 19:53:27.330 23645 ERROR neutron.agent.metadata.agent   File "/usr/lib/python3.9/site-packages/oslo_messaging/_drivers/amqpdriver.py", line 551, in wait
2024-10-29 19:53:27.330 23645 ERROR neutron.agent.metadata.agent     message = self.waiters.get(msg_id, timeout=timeout)
2024-10-29 19:53:27.330 23645 ERROR neutron.agent.metadata.agent   File "/usr/lib/python3.9/site-packages/oslo_messaging/_drivers/amqpdriver.py", line 427, in get
2024-10-29 19:53:27.330 23645 ERROR neutron.agent.metadata.agent     raise oslo_messaging.MessagingTimeout(
2024-10-29 19:53:27.330 23645 ERROR neutron.agent.metadata.agent oslo_messaging.exceptions.MessagingTimeout: Timed out waiting for a reply to message ID c12fb6e656654662b5d7f78231d9a2e8
2024-10-29 19:53:27.330 23645 ERROR neutron.agent.metadata.agent 
2024-10-29 19:53:27.333 23645 WARNING oslo.service.loopingcall [-] Function 'neutron.agent.metadata.agent.UnixDomainMetadataProxy._report_state' run outlasted interval by 30.08 sec
2024-10-29 19:53:27.342 23645 INFO neutron.agent.metadata.agent [-] Successfully reported state after a previous failure.
```

注意：至此没有出现分歧

### 计算节点

安装

```shell
yum install -y openstack-neutron-linuxbridge ebtables ipset
```

#### 配置通用组件

备份、过滤

```shell
cp /etc/neutron/neutron.conf{,.bak}
grep -Ev "^$|#" /etc/neutron/neutron.conf.bak > /etc/neutron/neutron.conf
```

添加配置信息`vim /etc/neutron/neutron.conf`

```ini
[DEFAULT]
transport_url = rabbit://openstack:000000@controller
auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = neutron

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
```

#### 配置三层代理

备份、过滤

```shell
cp /etc/neutron/plugins/ml2/linuxbridge_agent.ini{,.bak}
grep -Ev "^$|#" /etc/neutron/plugins/ml2/linuxbridge_agent.ini.bak > /etc/neutron/plugins/ml2/linuxbridge_agent.ini
```

添加配置信息`vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini`

```ini
[linux_bridge]
physical_interface_mappings = extnal:eth0

[vxlan]
enable_vxlan = true
local_ip = 192.168.100.20
l2_population = true

[securitygroup]
# ...
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

配置内核开放

```shell
tee -a /etc/sysctl.conf <<EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
```

加载模块并使其生效

```shell
modprobe br_netfilter
sysctl -p
```

#### 配置 Compute 服务以使用 Networking 服务

新增以下配置`vim /etc/nova/nova.conf`

```ini
[neutron]
auth_url = http://controller:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = neutron
password = neutron
```

#### 完成安装

重启`nova-api`服务，很慢

```shell
systemctl restart openstack-nova-compute.service
```

注意：到这一步完全一致

启动`neutron`

```shell
systemctl enable --now neutron-linuxbridge-agent.service
```

验证（在`controller`节点验证）

```shell
openstack network agent list
neutron agent-list
```

完美成功
![成功](https://raw.gitmirror.com/ByteQuestor/picture/main/OpenStackByHandImg/new_neutron_success.png)
---

这些不用管

下图是`linuxbridge`的日志

![报错解决](https://raw.gitmirror.com/ByteQuestor/picture/main/OpenStackByHandImg/010.jpg)

没用`down`或`false`表示成功

~~由于部署过程出现些小差错，应当是两个`linuxbridge-agent`、一个`neutron-13-agent`，一个`dhcp-agent`，一个`metadata-agent`~~

---

## dashboard

安装

```shell
yum install -y openstack-dashboard
```

配置

```shell
vim /etc/openstack-dashboard/local_settings
```

配置内容

```shell
OPENSTACK_HOST = "controller"
ALLOWED_HOSTS = ["*"]
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
        'LOCATION': 'controller:11211',
    },
}
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 3,
}
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
TIME_ZONE = "Asia/Shanghai"
```

配置（这一步没啥做的，关于这个配置文件可以进一步了解）

```shell
vim /etc/httpd/conf.d/openstack-dashboard.conf 
# 如果没有下面这行需要加上
WSGIApplicationGroup %{GLOBAL}
```

### 解决问题

> 访问地址（存在的问题是：登录进去后，直接跳转`apache`测试页面）

```shell
http://192.168.100.10/dashboard/auth/login/
```

问题解决：修改`vim /etc/httpd/conf.d/openstack-dashboard.conf `的如下地方

![报错解决](https://raw.gitmirror.com/ByteQuestor/picture/main/OpenStackByHandImg/013.jpg)

![报错解决](https://raw.gitmirror.com/ByteQuestor/picture/main/OpenStackByHandImg/011.jpg)

![报错解决](https://raw.gitmirror.com/ByteQuestor/picture/main/OpenStackByHandImg/012.jpg)

> 问题（说找不到`/usr/share/openstack-dashboard/openstack_dashboard/conf/nova_policy.json`）

```shell
tail -f /var/log/httpd/openstack-dashboard-error_log
```

![报错解决](https://raw.gitmirror.com/ByteQuestor/picture/main/OpenStackByHandImg/014.jpg)

使用`find`工具找一下

```shell
[root@controller ~]# find / -name nova_policy.json
/usr/lib/python3.9/site-packages/openstack_auth/tests/conf/nova_policy.json
/etc/openstack-dashboard/nova_policy.json	# <-- 这个是需要的

# 其他的也都在这里
[root@controller ~]# ll /etc/openstack-dashboard/
total 92K
-rw-r----- 1 root apache 8.4K May 17  2021 cinder_policy.json
-rw-r----- 1 root apache 1.4K May 17  2021 glance_policy.json
-rw-r----- 1 root apache  10K May 17  2021 keystone_policy.json
-rw-r----- 1 root apache  13K Oct  2 14:39 local_settings
-rw-r----- 1 root apache  13K Oct  2 14:13 local_settings.rpmsave
-rw-r----- 1 root apache  13K May 17  2021 neutron_policy.json
drwxr-xr-x 2 root root   4.0K Oct  2 14:35 nova_policy.d
-rw-r----- 1 root apache  10K May 17  2021 nova_policy.json
```

之所以会报错是因为找的不是存放这些文件的目录：（找的是下面这个目录，本地都没有这个目录，所以肯定会报错）

```shell
/usr/share/openstack-dashboard/openstack_dashboard/conf/cinder_policy.json
```

解决方法：

+ 拷贝过去

+ 软链接【推荐】
  ```shell
  ln -s /etc/openstack-dashboard /usr/share/openstack-dashboard/openstack_dashboard/conf
  ```

  再次查看`/usr/share/openstack-dashboard/openstack_dashboard/conf/`会发现下面有东西了

重启`httpd`

```shell
systemctl restart httpd memcached
```

成功解决

> 问题：身份管理界面，一点就`Internal Server Error`或 `Forbidden``

```shell
[Wed Oct 02 15:12:33.837776 2024] [authz_core:error] [pid 24340:tid 24518] [client 192.168.100.1:9373] AH01630: client denied by server configuration: /usr/bin/keystone-wsgi-public, referer: http://192.168.100.10/admin/metadata_defs/
[Wed Oct 02 15:12:41.231599 2024] [authz_core:error] [pid 24340:tid 24528] [client 192.168.100.1:9373] AH01630: client denied by server configuration: /usr/bin/keystone-wsgi-public, referer: http://192.168.100.10/admin/metadata_defs/
[Wed Oct 02 15:12:47.842971 2024] [authz_core:error] [pid 24569:tid 24619] [client 192.168.100.1:9382] AH01630: client denied by server configuration: /usr/bin/keystone-wsgi-public, referer: http://192.168.100.10/admin/metadata_defs/
[Wed Oct 02 15:12:57.859191 2024] [authz_core:error] [pid 24341:tid 24546] [client 192.168.100.1:9386] AH01630: client denied by server configuration: /usr/bin/keystone-wsgi-public, referer: http://192.168.100.10/admin/metadata_defs/
```

这两个解决方法一样

```shell
vim /etc/httpd/conf.d/openstack-dashboard.conf
```

![报错解决](https://raw.gitmirror.com/ByteQuestor/picture/main/OpenStackByHandImg/015.jpg)

```shell
vim /etc/openstack-dashboard/local_settings
```

![报错解决](https://raw.gitmirror.com/ByteQuestor/picture/main/OpenStackByHandImg/016.jpg)

注意：此问题属于最开始访问页面的衍生问题，其实只需要最开始加一个`WEBROOT="/dashboard/"`，然后通过此路径访问即可

以下是正常的

![报错解决](https://raw.gitmirror.com/ByteQuestor/picture/main/OpenStackByHandImg/017.jpg)

> 创建一个实例测一下

![报错解决](https://raw.gitmirror.com/ByteQuestor/picture/main/OpenStackByHandImg/018.jpg)

![创建网络](https://raw.gitmirror.com/ByteQuestor/picture/main/OpenStackByHandImg/019.jpg)

![报错解决](https://raw.gitmirror.com/ByteQuestor/picture/main/OpenStackByHandImg/020.jpg)

![报错解决](https://raw.gitmirror.com/ByteQuestor/picture/main/OpenStackByHandImg/021.jpg)

![报错解决](https://raw.gitmirror.com/ByteQuestor/picture/main/OpenStackByHandImg/022.jpg)

由于网络部分出现小问题，所以暂时无法做云主机实例

接下来是创建实力类型、创建实例、使用我们上传的`cirros`镜像，网络选择内网网卡

`注意`:创建实例的时候会出现报错，没有文件夹的权限

```shell
chown -R nova.nova $文件夹
```

这样创建的云主机是可以用`shell`连接的，还可以上网

---

先暂停，后续再补上

## cinder

## swift

