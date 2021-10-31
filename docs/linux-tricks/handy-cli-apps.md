---
title: 另类好用的 Linux 命令行工具
authors:
  - 李皓钧
date: 2021-10-31
---

## Homebrew

### 简介

Homebrew 是一款起源于服务 macOS 的包管理器，特色是在用户态进行包管理和安装，做法非常整齐优雅，并且有非常大的社区支持。虽说 Linux 下各发行版也有用很多包管理器，但是总是会缺少一些好用的小软件，这时 Homebrew 就有用武之地了。强烈推荐先安装 Homebrew，因为可能会更方便地安装下面介绍的软件。

### 资源

- 官方网站、快速入门：[https://brew.sh](https://brew.sh)
- Homebrew TUNA 镜像源配置教程：[https://mirrors.tuna.tsinghua.edu.cn/help/homebrew/](https://mirrors.tuna.tsinghua.edu.cn/help/homebrew/)

## tldr

### 简介

Too long; didn't read. 废话不多说，就是这款小工具的目的。想必各位时不时都要用一些自己没用过的命令，做一些简单的事情，但却要翻看天书一般 man，可谓杀鸡用牛刀。tldr 提供的文档则恰恰相反：直击痛点，通俗易懂，示例清晰。

### 资源

- 官方网站：[https://tldr.sh](https://tldr.sh)
- 快速入门：安装 Homebrew 后使用 `brew install tldr` 命令即可安装。
- 相关中文教程：[https://learnku.com/articles/23834](https://learnku.com/articles/23834)

## fasd

### 简介

虽说 `z` 快速转跳鼎鼎大名，但我用过最舒服的还是这个工具。只要你曾经访问过某个目录货文件，fasd 能让你只输入目录名或文件名就实现全路径的引用或转跳，而且转跳命令也是 `z`————例如，输入 `z dow` 即可从任意位置转跳到 `~/Download`。fasd 的命令还方便记忆：`f` 代表 file，`a` 代表 all，`s` 代表 show，`d` 代表 directory，完全没有上手难度。

### 资源

- 官方 GitHub：[https://github.com/clvv/fasd](https://github.com/clvv/fasd)
- 快速入门：同样地，使用 `brew install fasd` 命令即可安装。安装后可能需要配置 rc 文件（如 zsh 为 `.zshrc`），详细配置参考官方文档。
- 相关中文教程：[https://www.howtoing.com/fasd-quick-access-to-linux-files-and-directories](https://www.howtoing.com/fasd-quick-access-to-linux-files-and-directories)

## 附录

暂时就想到这么多，到时候再添加吧。