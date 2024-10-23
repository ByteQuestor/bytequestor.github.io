---
title: 在win上配置快捷键启动应用
date: 2024-10-23 11:02:59
tags:
    - windows
cover: https://raw.gitmirror.com/ByteQuestor/picture/main/windows.jpg
description: 最近觉得使用终端很酷，所以配置一下在win上快捷键启动应用程序（主要是快捷键启动终端）
---
+ 终端软件
官方网站（https://tabby.sh/）
+ 首先安装 `AutoHotkey`
下载地址：（https://www.autohotkey.com/）

在任意位置写一个文件（命名为`*.ahk`，比如`openTabbby.ahk`），内容如下

```ahk
#!t::
Run "C:\Program Files\Tabby\Tabby.exe"
Return
```

这个代码的意思是：按下`win + alt + t`即可打开下面的安装目录

注意把`C:\Program Files\Tabby\Tabby.exe`替换为自定义安装的目录

+ 运行

写好文件以后，双击此文件（要先安装 `AutoHotkey`），选择“是”即可

如果一次没运行成功再试一次，托盘出现这个就可以用了

![图片](https://raw.gitmirror.com/ByteQuestor/picture/questforter/1.png)

此时，重启后这个快捷键就会失效

将` AutoHotkey` 脚本设置为开机自动运行。具体方法如下：

- 按下`Win + R`组合键，打开 “运行” 对话框，输入`shell:startup`，然后回车。这会打开系统的 “启动” 文件夹。
- 将之前编写的`.ahk`文件（例如`OpenTabby.ahk`）**或者**它的的快捷方式复制到这个 “启动” 文件夹中。这样，每次开机时，`AutoHotkey` 脚本就会自动运行，设置的打开`Tabby`的快捷键就可以正常使用了
