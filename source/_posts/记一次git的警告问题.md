---
title: 记一次git的警告问题
date: 2024-12-03 22:21:13
tags:
  - git
description: 重置电脑后，发现上传时总是会报一些警告
cover: https://raw.gitmirror.com/ByteQuestor/picture/main/GitHub-Logo.png
---
> 本篇省流：虽然不好看，但是并没有什么影响
```tex
wzy@DESKTOP-AFD56J4 MINGW64 /e/User/wzy/github/Vue3-learn (main)
$ git add .
warning: LF will be replaced by CRLF in hello_vue3/.gitignore.
The file will have its original line endings in your working directory
warning: LF will be replaced by CRLF in hello_vue3/.vscode/extensions.json.
The file will have its original line endings in your working directory
warning: LF will be replaced by CRLF in hello_vue3/README.md.
The file will have its original line endings in your working directory
warning: LF will be replaced by CRLF in hello_vue3/env.d.ts.
The file will have its original line endings in your working directory
warning: LF will be replaced by CRLF in hello_vue3/index.html.
The file will have its original line endings in your working directory
warning: LF will be replaced by CRLF in hello_vue3/package-lock.json.
The file will have its original line endings in your working directory
warning: LF will be replaced by CRLF in hello_vue3/package.json.
The file will have its original line endings in your working directory
warning: LF will be replaced by CRLF in hello_vue3/src/App.vue.
The file will have its original line endings in your working directory
warning: LF will be replaced by CRLF in hello_vue3/src/assets/base.css.
The file will have its original line endings in your working directory
warning: LF will be replaced by CRLF in hello_vue3/src/assets/logo.svg.
The file will have its original line endings in your working directory
warning: LF will be replaced by CRLF in hello_vue3/src/assets/main.css.
The file will have its original line endings in your working directory
warning: LF will be replaced by CRLF in hello_vue3/src/components/HelloWorld.vue.
The file will have its original line endings in your working directory
warning: LF will be replaced by CRLF in hello_vue3/src/components/TheWelcome.vue.
The file will have its original line endings in your working directory
warning: LF will be replaced by CRLF in hello_vue3/src/components/WelcomeItem.vue.
The file will have its original line endings in your working directory
warning: LF will be replaced by CRLF in hello_vue3/src/components/icons/IconCommunity.vue.
The file will have its original line endings in your working directory
warning: LF will be replaced by CRLF in hello_vue3/src/components/icons/IconDocumentation.vue.
The file will have its original line endings in your working directory
warning: LF will be replaced by CRLF in hello_vue3/src/components/icons/IconEcosystem.vue.
The file will have its original line endings in your working directory
warning: LF will be replaced by CRLF in hello_vue3/src/components/icons/IconSupport.vue.
The file will have its original line endings in your working directory
warning: LF will be replaced by CRLF in hello_vue3/src/components/icons/IconTooling.vue.
The file will have its original line endings in your working directory
warning: LF will be replaced by CRLF in hello_vue3/src/main.ts.
The file will have its original line endings in your working directory
warning: LF will be replaced by CRLF in hello_vue3/tsconfig.app.json.
The file will have its original line endings in your working directory
warning: LF will be replaced by CRLF in hello_vue3/tsconfig.json.
The file will have its original line endings in your working directory
warning: LF will be replaced by CRLF in hello_vue3/tsconfig.node.json.
The file will have its original line endings in your working directory
warning: LF will be replaced by CRLF in hello_vue3/vite.config.ts.
The file will have its original line endings in your working directory

```

# 问题的原因
+ 行尾符差异：

  `Windows` 使用 `CRLF (\r\n) `作为行尾符。
  `Linux` 和 `macOS` 使用` LF (\n)` 作为行尾符。

+ `Git` 的默认行为：

  当配置了 `core.autocrlf` 为` true` 时：
  提交时，`Git` 会将 `Windows` 的 `CRLF` 转换为` LF`。
  检出时，`Git` 会将 `LF` 转换为 `Windows` 的 `CRLF`。
  这有助于跨平台兼容，但在某些场景下会引发警告。

+ 警告的含义：

  `Git` 检测到文件行尾符被替换，提示这些文件会在提交后以 `LF` 格式存储，但在工作目录中保留为原始格式（`CRLF`）。
  这种行为不会对文件内容或功能产生影响，但可能会对某些工具或环境引发不兼容问题。

## 解决

```shell
git config --global core.autocrlf true		# 无效
```


```shell
wzy@DESKTOP-AFD56J4 MINGW64 /e/User/wzy/github (main)
$ git config --global core.autocrlf true
```

#### **确认 `.gitattributes` 文件**

在项目根目录创建或检查 `.gitattributes` 文件，明确指定文件行尾符规则。通过 `.gitattributes` 可以确保 Git 在处理文件时一致地转换行尾符。你可以添加以下内容：

```
text复制代码# 强制所有文本文件使用 LF 作为行尾符
* text=auto eol=lf

# 如果有需要，特别指定某些文件使用 CRLF
# *.bat text eol=crlf
```

 