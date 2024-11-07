---
title: 记录如何使用github创建release
date: 2024-11-07 09:17:38
tags:
    - github
description: 本篇文章记录如何使用GitHub Action自动化部署Hexo博客
cover: https://raw.gitmirror.com/ByteQuestor/picture/main/GitHub-Logo.png
---

# 准备成品

首先，确保所有源文件改动已经上传至`main`分支

然后，导出成品到成品文件夹

https://raw.gitmirror.com/ByteQuestor/picture/blob/main/createrelease/1.png
# 创建成品分支

这个分支，将直接被全部打包为一个`release`

这就是为什么要确保源文件所有改动已经上传至`main`分支的原因，不然源码也会有一部分到这里

使用`git checkout -b pdf`来创建并切换到`pdf`分支。其中`-b`选项表示创建一个新分支并切换到它

```git
git checkout -b pdf
```

https://raw.gitmirror.com/ByteQuestor/picture/blob/main/createrelease/2.png
# 上传成品

## 提交更改到本地分支

```git
git add ./云计算笔记/
```

```shell
git commit -m "update v1.0"
```

## 推送更改到远程仓库

```git
git push origin pdf
```

此图是个私有仓库

https://raw.gitmirror.com/ByteQuestor/picture/blob/main/createrelease/4.png
上传完毕，就可以创建`release`了

# 创建`release`

https://raw.gitmirror.com/ByteQuestor/picture/blob/main/createrelease/5.png
https://raw.gitmirror.com/ByteQuestor/picture/blob/main/createrelease/6.png
