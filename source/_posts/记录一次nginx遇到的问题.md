---
title: 记录一次nginx遇到的问题
date: 2024-11-13 11:34:49
tags:
    - nginx
cover: https://raw.gitmirror.com/ByteQuestor/picture/main/nginx/nginx.jpg
description: 有一次配反向代理，结果报错404，决定研究下这个反向代理具体用法
---
```shell
root@jsmc:~# curl 172.17.0.11

# 回显
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>

</head>

<body>

    <section>
        <div class="into">
            <h2>欢迎登录家谱网</h2>
            <form name="f" onsubmit="check(this)">
                <div class="box">
                    <label>用户名:</label>
                    <input id="name" type="text">
                </div>
                <div class="box2">
                    <label>密码:</label><input id="pwd" type="password">
                </div>
                <div class="remember">
                    <label><input type="checkbox" value="记住密码">记住密码</label>
                </div>
                <button onclick="check(this)">登录</button>
            </form>
        </div>
        <p><a href="sing.html">注册</a></p>
    </section>
    <div id="box">
        <i>什么！进不去？加我qq</i>
        <img src="qq1.jpg">
        <i id="btn">
            <p>x</p>
        </i>
    </div>
</body>
```

使用`ngixn`反向代理这个容器报`404`，配置文件如下

```html
root@jsmc:~# cat /etc/nginx/sites-available/default     
server {
        listen 80 default_server;
        listen [::]:80 default_server;


        root /var/www/html;
    
        # Add index.php to the list if you are using PHP
        index index.html index.htm index.nginx-debian.html;
    
        server_name _;
    
        location /monopoly/ {
                proxy_pass http://172.17.0.2/;
        }
    
        location /test/ {
                proxy_pass http://172.17.0.4:80;
        }
        location /todo/ {
                proxy_pass http://172.17.0.9:5000;
        }
        location /test_phone/ {
                proxy_pass http://172.17.0.4:80;
        }
        location /wzy/ {
                proxy_pass http://localhost:9000/;  # 注意结尾的斜杠
        }
        location /pyf {
                proxy_pass http://172.17.0.11:80;
        }
    location / {
        # 根据 $mobile 变量进行代理
        set $target http://172.17.0.4;  # 默认代理到 PC 设备
    
        if ($mobile) {
            set $target http://172.17.0.4;  # 移动设备
        }
    
        proxy_pass $target;  # 使用变量进行代理
    }
}
```
`ChatGPT说：`

> 从你提供的 `Nginx` 配置来看，问题可能出在你的反向代理配置上。具体来说，`proxy_pass` 后面的 URL 地址和路径处理方式有些问题。

+ 问题分析
  代理目标路径匹配问题：在你配置的` location /pyf` 中，`proxy_pass http://172.17.0.11:80;` 会将请求的路径 `/pyf` 直接代理到目标服务器上，这就可能导致目标服务器处理时出现路径问题。
  换句话说，`Nginx` 会将请求 `/pyf `转发给` http://172.17.0.11:80/pyf`，这可能是目标容器应用并不存在的路径，导致 `404` 错误。
  反向代理时路径截取：如果你不想让代理时路径的前缀 `/pyf` 出现在目标请求中，需要在 `proxy_pass` 后加上一个 `/`（斜杠），这样代理时路径就会从目标` URL` 后拼接而不是带上前缀。

+ 解决方案：
  修改 `Nginx` 配置中的 `proxy_pass` 设置，确保路径正确传递给后端服务。具体修改如下：

  ```ini
  location /pyf/ {  # 注意这里的路径要带上斜杠
      proxy_pass http://172.17.0.11:80/;  # 注意结尾的斜杠
  }
  ```
+ 解释：
`location /pyf/`：匹配所有以` /pyf/` 开头的请求。
`proxy_pass http://172.17.0.11:80/;` 
       这里的 `proxy_pass` 后面加上了斜杠` /`，这样 `Nginx` 会把请求 `/pyf/xxx` 转发为` http://172.17.0.11:80/xxx`，避免了路径前缀被带到目标服务器。
+ 其他注意事项：

  路径匹配：如果你的容器内的 Web 应用默认使用根路径 /，则确保 `Nginx` 配置中的路径与容器内部应用的路径匹配。
  确保容器可访问：确认 `Nginx` 容器能够访问到 `172.17.0.11` 的 IP 地址（即目标容器的 IP）。
  修改完成后，重启`nginx`查看是否解决问题
