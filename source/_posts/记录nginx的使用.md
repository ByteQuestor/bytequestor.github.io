---
title: 记录nginx的使用
date: 2024-09-18 20:38:06
tags:
    - nginx
    - 负载均衡
cover: https://raw.gitmirror.com/ByteQuestor/picture/main/nginx/nginx.jpg
description: 以前总是对这个nginx很感兴趣，但是一直没怎么用，最近决定一举拿下
---
# nginx的安装

本次使用yum在`centos7.9（能联网）`安装nginx

## 准备yum源

复制下列命令粘贴后执行（注意有删除命令，应在纯净系统搞）

```shell
systemctl stop firewalld
systemctl disable firewalld
rm -rf /etc/yum.repos.d/*
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
curl -o /etc/yum.repos.d/docker-ce.repo https://mirrors.cloud.tencent.com/docker-ce/linux/centos/docker-ce.repo 
yum makecache
yum install epel-release -y

```

## 安装nginx

```shell
yum install -y nginx
systemctl start nginx
systemctl enable nginx

```

# 安装docker

为了在同一台虚拟机里，实现起多个容器，实现负载均衡的效果

```shell
yum install -y docker-ce
systemctl start docker
systemctl enable docker

```

加载`nginx`容器镜像（这两个是为了开两个容器，使用本机代理到两个容器，两个容器中会存放两个不同的网页以区分负载均衡效果）

本节用到的镜像链接`此链接永久有效`

<此处插入百度网盘永久地址>

```shell
docker load -i nginx_latest.tar
```

# 编写简易网页和dockerfile

+ 1.html

  ```html
  1111111111111
  ```

+ 2.html

  ```html
  2222222222222
  ```

+ docker_nginx1
  ```dockerfile
  FROM nginx:latest
  COPY 1.html /usr/share/nginx/html/index.html
  EXPOSE 80
  CMD ["nginx","-g","daemon off;"]
  ```

+ docker_nginx2
  ```dockerfile
  FROM nginx:latest
  COPY 2.html /usr/share/nginx/html/index.html
  EXPOSE 80
  CMD ["nginx","-g","daemon off;"]
  ```

+ 构建镜像
  ```shell
  docker build -t nginx1:v1 -f docker_nginx1 .
  docker build -t nginx2:v1 -f docker_nginx2 .
  ```

+ 运行镜像
  ```shell
  docker run -d -p 8090:80 nginx1:v1
  docker run -d -p 8091:80 nginx2:v1
  ```

+ 访问

  + 浏览器访问8090
    ![nginx配置](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/nginx/002.png)
  + 浏览器访问8091
    ![nginx配置](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/nginx/003.png)
  + 本地``curl`访问
    ![nginx配置](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/nginx/004.png)

# 反向代理

即请求到某个域名/端口被nginx接收到，nginx会根据配置把特定的请求转发到对应的服务器，类似DNS，本次

## 修改nginx配置文件

常用配置文件

```shell
vi /etc/nginx/nginx.conf
```

另一个配置文件位置

```shell
/etc/nginx/sites-enabled# cat default
```



查看容器信息（比如IP）

```shell
docker inspect $ID
```

配置文件写容器IP可进行代理

![nginx配置](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/nginx/005.png)

## 重启nginx并访问

```shell
systemctl restart nginx

```

成功代理

# 负载均衡

修改配置文件

```shell
vi /etc/nginx/nginx.conf
```

![nginx配置](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/nginx/006.png)

# 长连接

`keepalive`

如果长连接进来，必须保证其保持指定时间

![nginx配置](https://raw.gitmirror.com/ByteQuestor/picture/main/Ink8sUpdateWeatherForecast/nginx/007.png)

# 容错机制

+ failover：失效转移
  `Fail-Over`的含义为“失效转移”，是一种备份操作模式，当主要组件异常时，其功能转移到备份组件。其要点在于有主有备，且主故障时备可启用，并设置为主。如Mysql的双Master模式，当正在使用的Master出现故障时，可以拿备Master做主使用

+ failfast：快速失败
  从字面含义看就是“快速失败”，尽可能的发现系统中的错误，使系统能够按照事先设定好的错误的流程执行，对应的方式是“fault-tolerant（错误容忍）”。以JAVA集合（Collection）的快速失败为例，当多个线程对同一个集合的内容进行操作时，就可能会产生fail-fast事件。例如：当某一个线程A通过iterator去遍历某集合的过程中，若该集合的内容被其他线程所改变了；那么线程A访问集合时，就会抛出ConcurrentModificationException异常（发现错误执行设定好的错误的流程），产生fail-fast事件

+ failback：失效自动恢复
  `Fail-ove`r之后的自动恢复，在簇网络系统（有两台或多台服务器互联的网络）中，由于要某台服务器进行维修，需要网络资源和服务暂时重定向到备用系统。在此之后将网络资源和服务器恢复为由原始主机提供的过程，称为自动恢复

+ failsafe：失效安全
  `Fail-Safe`的含义为“失效安全”，即使在故障的情况下也不会造成伤害或者尽量减少伤害。维基百科上一个形象的例子是红绿灯的“冲突监测模块”当监测到错误或者冲突的信号时会将十字路口的红绿灯变为闪烁错误模式，而不是全部显示为绿灯



> 又没用上，下面内容纯当记录命令了

---

# 比赛要用到的命令

比如

```shell
curl -XGET localhost:3000/api/v1/ping
或
看日志没报错就行
```



# 安装tomcat

为了利用`tomcat`的8080端口来验证反向代理是否成功

```shell
sudo yum install java-11-openjdk
java -version
echo "export JAVA_HOME=/usr/lib/jvm/java-11-openjdk" >> ~/.bashrc 
echo "export PATH=$PATH:$JAVA_HOME/bin" >> ~/.bashrc
source ~/.bashrc 

yum install -y tomcat
systemctl start tomcat
systemctl enable tomcat

```

# 防火墙开放端口

开放80端口访问nginx

```shell
firewall-cmd --zone=public --add-port=80/tcp --permanent		# 回显 success 
firewall-cmd --reload  # 回显 success 
firewall-cmd --zone=public --query-port=80/tcp # 回显 yes
```
