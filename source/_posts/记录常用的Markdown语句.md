---
title: 记录常用的Markdown语句
date: 2024-10-25 17:34:17
tags:
  - markdown
cover: https://raw.gitmirror.com/ByteQuestor/picture/main/markdown.png
description: markdown功能很强大，在此记录一下常用的命令及效果
---
# Markdown篇

+ 基本样式
  ```markdown
  **加粗内容**
  *斜体内容*
  ~~删除线效果~~
  ```

  **加粗内容**
  *斜体内容*
  ~~删除线效果~~

+ 描述类
  ```markdown
  > 这是一段描述
  ```

  > 这是一段描述

+ 字符串

  > 写法描述：将字符串放在中间

  ```markdown
  `string`
  ```

  效果`string`

+ 代码类

  > 写法描述： ```后接语言类型，如：java、c、python

  ````markdown
  ```c
  #include<sdtio.h>
  int main()
  {
      printf("hello world")
      return 0;
  }
  ```
  ````

  > 效果

  ```c
  #include<sdtio.h>
  int main()
  {
      printf("hello world")
      return 0;
  }
  ```

  

+ 标题类

  > 写法描述：几个`#`就是几级标题

  ```markdown
  # 这是一个一级标题
  ## 这是一个二级标题
  ### 这是一个三级标题
  ```

  > 效果

  # 这是一个一级标题
  ## 这是一个二级标题
  ### 这是一个三级标题

+ 列表类【了解即可】

  > 写法描述

  ```markdown
  + 第一个
  
    + 子列表
  + 第二个
  
    + 子列表
  ```
> 效果


+ 第一个

  + 子列表
+ 第二个

  + 子列表

注意：按下`tab`键即可切换为子列表



# hexo写博客

1. 创建博客

   > 写法描述：hexo new $博客名字

   ```shell
   hexo new 记一下我第一次写博客
   ```

2. 写博客
   > 使用`markdown`篇的语法写内容
   
3. 清理缓存

   > 每次上传前把缓存清理掉

   ```shell
   hexo clean
   ```

4. 预览&&上传

   > 先预览再上传

   ```shell
   hexo s # 本地预览
   hexo d # 上传至github（可以公网访问）
   ```

   