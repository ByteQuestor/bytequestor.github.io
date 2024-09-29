---
title: Hexo配合GithubAction自动化部署个人博客
date: 2024-09-29 23:04:32
tags:
    - CICD
description: 本篇文章记录如何使用GitHub Action自动化部署Hexo博客
cover: https://raw.gitmirror.com/ByteQuestor/picture/blob/main/GitHub-Logo.png
---
# 关于自动化部署个人博客

+ 建一个文件夹初始化`hexo`
  ```shell
  npm install -g hexo-cli
  hexo init # 就是这一步，会出现yaml文件和文件夹，如果后面加一个文件夹名字，就会生成一个文件夹并放这些东西
  npm install
  ```

+ 初始化仓库
  直接将这些yaml和文件夹推送到github仓库`main`分支

  ```shell
  git remote add origin https://github.com/ByteQuestor/ByteQuestor.github.io.git
  git add .
  git commit -m "Init"
  git push origin master:main # 新版的直接main即可
  ```

![文件结构](https://raw.gitmirror.com/ByteQuestor/picture/main/Action/001.jpg)

+ 创建`token`
  在个人设置中新增`Personal access tokens`，要包含`repo`权限，这个`token`是给`Github Action`用的，`Github`会把`hexo`编译部署到`gh-pages`分支
  注意不是仓库的`setting`，而是账户的`setting`

  ![文件结构](https://raw.gitmirror.com/ByteQuestor/picture/main/Action/002.jpg)

  ![TOKEN](https://raw.gitmirror.com/ByteQuestor/picture/main/Action/003.jpg)

  ![文件结构](https://raw.gitmirror.com/ByteQuestor/picture/main/Action/004.jpg)

  然后去代码仓库把这个token新增

  ![文件结构](https://raw.gitmirror.com/ByteQuestor/picture/main/Action/005.jpg)

+ 更改`hexo`配置文件（如果之前部署过，直接把yaml文件复制过来更方便）
  ```yaml
  deploy:
    type: git
    repo: https://github.com/ByteQuestor/ByteQuestor.github.io.git
    branch: gh-pages
  ```

+ 配置`Github Action`工作流
  `node-version`要和本地的`node`版本一致
  用`ununtu-latest`作为基础环境，然后按照各种依赖，随后`hexo generate `生成博客网站静态文件，把这个文件夹推送到同一仓库的`gh-pages`分支
  首先在`.github`下创建一个`workflows`文件夹和`deploy.yml`，写入如下内容

  ```yaml
  name: Deploy Hexo
  
  on:
    workflow_dispatch:  # 允许手动触发
    push:
      branches:
        - main
  
  jobs:
    build:
      runs-on: ubuntu-latest
  
      steps:
        - name: Checkout repository
          uses: actions/checkout@v2
          with:
            submodules: true  # 根据需求设置 true/false
            
        - name: Setup Node.js
          uses: actions/setup-node@v2
          with:
            node-version: '20'  # 使用大版本号即可 (v20.x.x)
  
        - name: Install Dependencies
          run: npm install
  
        - name: Install Hexo git Deployer
          run: |
            npm install hexo-deployer-git --save
            npm install hexo-cli -g
  
        - name: Clean and Generate Static Files
          run: |
            hexo clean
            hexo generate
  
        - name: Configure Git
          run: |
            git config --global user.name 'github-action[bot]'
            git config --global user.email 'github-actions[bot]@users.noreply.github.com'
  
        - name: Deploy to Github Pages
          env:
            GH_TOKEN: ${{ secrets.GH_TOKEN }}
          run: |
            cd public/
            git init
            git add -A
            git commit -m "Deploy to GitHub Pages"
            git remote add origin https://${{ secrets.GH_TOKEN }}@github.com/ByteQuestor/ByteQuestor.github.io.git
            git push origin HEAD:gh-pages -f
  ```

+ 推送验证
  ```shell
  git add .
  git commit -m "Init"
  git push origin master:main # 新版的直接main即可
  ```

![文件结构](https://raw.gitmirror.com/ByteQuestor/picture/main/Action/006.jpg)

![文件结构](https://raw.gitmirror.com/ByteQuestor/picture/main/Action/007.jpg)

![文件结构](https://raw.gitmirror.com/ByteQuestor/picture/main/Action/008.jpg)

+ 选择一个喜欢的主题，根据主题需要的环境进行配置
  以`butterfly`为例，直接在hexo目录下克隆

  ```shell
  git submodule add https://github.com/jerryc127/hexo-theme-butterfly.git themes/butterfly
  git submodule update --init --recursive
  ```

+ 安装依赖和渲染器
  ```shell
  npm install hexo-renderer-pug hexo-renderer-stylus --save
  ```

  ![文件结构](https://raw.gitmirror.com/ByteQuestor/picture/main/Action/009.jpg)

+ 在hexo的`_config.yaml`配置主题
  ![文件结构](https://raw.gitmirror.com/ByteQuestor/picture/main/Action/011.jpg)

+ 推送上去查看效果（我是直接复制的之前的yaml，因此直接是成品效果，从头开始需要按照主题的教程来）

  ```shell
  git add .
  git commit -m "Init"
  git push origin master:main # 新版的直接main即可
  ```

  ![文件结构](https://raw.gitmirror.com/ByteQuestor/picture/main/Action/013.jpg)



# 这样做的目的：

如果需要换电脑或者其他操作，只需要克隆`main`分支和即可，而且平时写博客也可以直接想main推送文章，进而实现自动化部署