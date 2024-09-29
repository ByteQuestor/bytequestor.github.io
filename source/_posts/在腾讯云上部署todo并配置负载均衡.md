---
title: 在腾讯云上部署todo并配置负载均衡
date: 2024-09-19 17:07:45
tags:
    - 腾讯云
    - 负载均衡
cover: https://raw.gitmirror.com/ByteQuestor/picture/main/tencent/tencent.png
description: 使用腾讯云跑todo并配置负载均衡,负载均衡至两台实例,两台实例再分别对内部多个容器进行nginx代理均衡
---
实验目标：
首先起两台实例，部署todo服务，同时对接redis和mysql，并且实现负载均衡
实验步骤：

+ 安装docker
+ 在腾讯云镜像服务拉取镜像
+ 安装nginx（在实例上配置，主要是为了负载均衡）
注意事项：
+ 需要进入mysql容器创建 todolist 数据库
+ 必须会写脚本同时起100台容器，并且写入nginx的轮询以负载均衡
+ 必须会写脚本删除脚本起的容器
+ 必须会将日志挂载到本机的同一个文件下（如果题目要求的话）



---

# 一键搭建脚本

> 先在本地实验一下，写个shell脚本去云上

先登录腾讯云

```shell
docker login ccr.ccs.tencentyun.com --username=100032122665
RootRoot123.
```

然后执行以下脚本

```shell
#!/bin/bash
systemctl stop firewalld
systemctl disable firewalld
# 关闭 SELinux 强制模式，解决 Nginx 502; 因为 SELinux 限制了 Nginx 访问上游服务器或网络资源
setenforce 0 
# 删除有风险
# rm -rf /etc/yum.repos.d/* # 如果是Debian则不用取消注释

# 如果用Debian就删除10-25行，反之也是
# --------------------centos ------------------------------------
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
curl -o /etc/yum.repos.d/docker-ce.repo https://mirrors.cloud.tencent.com/docker-ce/linux/centos/docker-ce.repo 
yum makecache
yum install epel-release -y

# 注意：如果是Debian，则需要将yum换成apt
# 接下来是安装docker
yum install -y docker-ce
systemctl start docker
systemctl enable docker
# 安装nginx
yum install -y nginx
systemctl start nginx
systemctl enable nginx
# --------------------------------------------------------
#---------------------------- Debian------------------------
sudo apt-get update -y
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://mirrors.cloud.tencent.com/docker-ce/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo   "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://mirrors.cloud.tencent.com/docker-ce/linux/debian/ \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" |   sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update -y
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
systemctl start docker
systemctl enable docker
# --------------------------------------------------------

# 从腾讯云拉取镜像
# 听说这样可以交互密码
echo "$password" | docker login ccr.ccs.tencentyun.com --username=$username --password-stdin
docker pull ccr.ccs.tencentyun.com/$user/redis:7.4.0
docker pull ccr.ccs.tencentyun.com/$user/mysql:8.0.39
docker pull ccr.ccs.tencentyun.com/$user/todo:v2

# MySql
docker run -d \
	-e MYSQL_ROOT_PASSWORD=000000 \
  -e MYSQL_DATABASE=todolist \
  --name my-mysql  \
  -p 13306:3306  ccr.ccs.tencentyun.com/troml/mysql:8.0.39
# Redis   --requirepass 一定要放在镜像以后
docker run -d \
  --name my-redis \
  -p 16379:6379 \
  ccr.ccs.tencentyun.com/troml/redis:7.4.0  --requirepass 000000
# todo 的配置文件
cat <<EOL > config.ini
[service]
AppMode = release
HttpPort = :3000

[redis]
RedisDb = redis
RedisAddr = 172.17.0.3:6379
RedisPw = 000000
RedisDbName = 2

[mysql]
Db = mysql
DbHost = 172.17.0.2
DbPort = 3306
DbUser = root
DbPassWord = 000000
DbName = todolist
EOL

# Todo 服务
docker run -itd -p 3000:3000 -v ./config.ini:/app/config/config.ini --name todo  ccr.ccs.tencentyun.com/troml/todo:v2
curl localhost:3000/api/v1/ping
```

# 批量起容器的脚本

批量生成脚本（应当琢磨如何简单化），保存变量是必须的，因为最后要去nginx的配置文件里配置

```shell
#!/bin/bash
# 起始端口
base_port=3000
output=""
# 启动10个容器
for i in {1..10}; do
    port=$((base_port + i))
    # 启动容器
    docker run -d -p $port:3000 -v "$(pwd)/config.ini:/app/config/config.ini" --name "todo-$i" ccr.ccs.tencentyun.com/troml/todo:v2   
    # 获取容器的IP地址
    container_ip=$(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' "todo-$i")   
    # 拼接输出字符串
    output+="server $container_ip:3000;\n"
done
# 最后集中输出所有IP
echo -e "$output"
```

批量删除脚本

```shell
#!/bin/bash
# 删除名为 todo-1 到 todo-n 的容器
for i in {1..10}; do
    docker rm -f "todo-$i"
    echo "Deleted container todo-$i"
done
```

# 配置nginx负载均衡

搭建基础环境基本上在脚本里都能看到，直接开始负载均衡

> 在本机负载均衡，将利用脚本批量生成10个应用服务容器，利用nginx进行轮询
>
> 为问不同端口的todo容器将会回显其主机名，由于主机名不同，达到验证负载均衡成功的目的
>
> 如下图：

curl访问

![curl访问效果](https://raw.gitmirror.com/ByteQuestor/picture/main/nginxTencent/001.png)

浏览器访问

![curl访问效果](https://raw.gitmirror.com/ByteQuestor/picture/main/nginxTencent/002.png)

## 运行脚本批量生成容器

（注意：新容器端口号要从3001开始）

将生成的容器加入到`/etc/nginx/nginx.conf`中

```shell
vi create.sh
bash create.sh
```

![批量生成容器](https://raw.gitmirror.com/ByteQuestor/picture/main/nginxTencent/003.png)

## 修改配置文件

```shell
vi /etc/nginx/nginx.conf
```

![nginx配置文件](https://raw.gitmirror.com/ByteQuestor/picture/main/nginxTencent/004.png)

```shell
systemctl restart nginx
```

查看报错的命令

```shell
tail -f /var/log/nginx/access.log /var/log/nginx/error.log
```



接下来，只需要访问我们虚拟机的`80`端口，即可实现轮询-负载均衡

![浏览器轮询](https://raw.gitmirror.com/ByteQuestor/picture/main/nginxTencent/005.png)

![curl轮询效果](https://raw.gitmirror.com/ByteQuestor/picture/main/nginxTencent/006.png)

# 配置nginx容错机制

> 难免会有容器出现问题，这时候就需要用到`容错机制`

## 容错机制介绍

+ **Failover（失效转移）**：如果主服务器出问题，自动切换到备用服务器，让系统继续运行。例如：如果数据库服务器崩溃，备用的服务器会接管工作。
  **场景应用**：你可以使用 `proxy_next_upstream` 指定在特定错误条件（如超时、错误代码）下，转移到其他服务器：

  ```json
  upstream server_list {
      server 172.17.0.5:3000 max_fails=1 fail_timeout=500ms;
      server 172.17.0.6:3000 max_fails=1 fail_timeout=500ms;
  }
  
  location / {
      proxy_pass http://server_list;
      proxy_next_upstream error timeout invalid_header http_502 http_503 http_504;
  }
  ```

+ **Failfast（快速失败）**：一旦发现错误，立即停止操作，不做多余的尝试。这样可以更快地发现问题，并马上切换到其他解决方案或服务器。
  **场景应用**：适用于需要快速失败的系统，不希望长时间等待，快速转移到下一个可用服务：

  ```shell
  location / {
      proxy_pass http://server_list;
      proxy_connect_timeout 100ms;  # 如果连接超过100毫秒未建立，立即失败
      proxy_read_timeout 300ms;     # 如果读取响应超过300毫秒，立即失败
  }
  ```

+ **Failback（失效恢复）**：当主服务器修复好后，系统会自动从备用服务器切换回主服务器，恢复到最初的状态。
  Nginx 本身并没有专门的 `failback` 机制，但配合健康检查模块或使用商业版的 Nginx Plus可以实现这种功能。当上游服务器恢复健康时，Nginx 可以自动将流量再次转移回去。也可以通过外部监控工具来检测服务器健康状态，并动态调整 Nginx 的上游服务器配置。

+ **Failsafe（失效安全）**：即使系统出错，也会确保安全运行，尽量减少损失或危险。例如：红绿灯出故障时会全亮红灯，而不是继续让车子通过，确保安全。
  **场景应用**：在所有上游服务器失败时，提供一个安全的降级机制或错误页面

  ```shell
  error_page 502 = /fallback.html;
  
  location /fallback.html {
      root /usr/share/nginx/html;
  }
  ```

  


根据我的需求，我需要在某个服务器挂掉以后，nginx**必须快速做出反应**请求到下一台服务器，因此本次选择`Failfast(快速失败)`


因为和之前的记录中断了，首先展示本次用到的上游服务器
![curl轮询效果](https://raw.gitmirror.com/ByteQuestor/picture/main/nginxTencent/007.png)


修改nginx配置文件
![curl轮询效果](https://raw.gitmirror.com/ByteQuestor/picture/main/nginxTencent/008.png)

```shell
    proxy_connect_timeout 100ms;  # 如果连接超过100毫秒未建立，立即失败
    proxy_read_timeout 300ms;     # 如果读取响应超过300毫秒，立即失败
```


用一个脚本验证

```python
import requests
import time  # 导入time模块
# 定义接口的URL
url = "http://192.168.100.201"
# 循环访问10次
for i in range(10):
    try:
        # 发送GET请求
        response = requests.get(url)        
        # 输出返回值
        print(f"请求 {i+1} 的返回值: {response.text}")   
    except requests.RequestException as e:
        print(f"请求 {i+1} 失败: {e}")
    # time.sleep(2)  # 每次请求之间等待2秒
```

## 长连接

在使用脚本验证失效服务之前，发现一个奇怪的现象

**`注意：图片标反了`**
![长连接干涉轮询](https://raw.gitmirror.com/ByteQuestor/picture/main/nginxTencent/009.png)

> 原因是由于长连接的存在，所以有复用的现象
> 如果keepalive_timeout=0，那么第一个请求还没结束，nginx就给切掉了，下一个请求来了还是在这个容器里面
> 如果keepalive_timeout=65，那么第一个请求不管它多久，我们的容器就会为它服务65秒，下一个请求来了就会轮询到下一台

1. **`keepalive_timeout=0`**：
   - 当 `keepalive_timeout` 设置为 0 时，每次请求都可能建立一个全新的连接，这意味着负载均衡器能够均匀地将请求分配到不同的容器上。因此，你在这种情况下成功地轮询到了 10 台机器。
2. **`keepalive_timeout=65`**：
   - 当 `keepalive_timeout` 设置为 65 秒时，HTTP 连接可能在多个请求之间复用。如果客户端在同一个连接上发送多个请求，负载均衡器不会重新分配这些请求，而是保持使用同一个后端服务器。因此，返回的容器主机名看起来没有明显的轮询规律，且多个请求可能被同一个容器处理。

### 解释：

- **Keepalive Connection**：HTTP keepalive 允许客户端与服务器保持持久连接，在同一个连接上发送多个请求。这减少了建立新连接的开销，但会导致请求可能多次被同一个服务器处理。
- **轮询模式失效**：当连接复用时，负载均衡器不会参与后续请求的重新分配。因此你看到某些容器多次响应同样的请求。

> 长连接65  再关掉了一个服务器

```shell
docker stop 1febd25736e3
```

![关闭一台上游服务器](https://raw.gitmirror.com/ByteQuestor/picture/main/nginxTencent/010.png)

再次轮询

![关一台后轮询效果](https://raw.gitmirror.com/ByteQuestor/picture/main/nginxTencent/011.png)

# 在腾讯云上实操

## 创建一个私有网络

<013>

<014>

## 购买两台实例

<016>

<017>

> 直接按上述步骤，搭建起所需要的服务

搭建方案：在两台实例上安装`nginx`，和``todo、redis、mysql`，并且在每个实例上都起多台实例
注意：Debian的nginx配置文件如下
<018>

## 购买负载均衡



