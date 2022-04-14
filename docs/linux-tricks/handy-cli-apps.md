---
title: 另类好用的 Linux 命令行工具
date: 2021-10-31
---

## Homebrew

### 简介

Homebrew 是一款起源于服务 macOS 的包管理器，特色是在用户态进行包管理和安装，做法非常整齐优雅，并且有非常大的社区支持。虽说 Linux 下各发行版也有用很多包管理器，但是总是会缺少一些好用的小软件，这时 Homebrew 就有用武之地了。强烈推荐先安装 Homebrew，因为可能会更方便地安装下面介绍的软件。

### 资源

- 官方网站、快速入门：[https://brew.sh](https://brew.sh)
- Homebrew TUNA 镜像源配置教程：[https://mirrors.tuna.tsinghua.edu.cn/help/homebrew/](https://mirrors.tuna.tsinghua.edu.cn/help/homebrew/)

## tldr [![tldr homebrew version](https://img.shields.io/homebrew/v/tldr?style=flat-square)](https://formulae.brew.sh/formula/tldr)

### 简介

Too long; didn't read. ~~即 “太长不看”~~ 废话不多说，就是这款小工具的目的。想必各位时不时都要用一些自己没用过的命令，做一些简单的事情，但却要翻看天书一般 man，可谓杀鸡用牛刀。tldr 提供的文档则恰恰相反：直击痛点，通俗易懂，示例清晰。

### 资源

- 官方网站：[https://tldr.sh](https://tldr.sh)
- 快速入门：安装 Homebrew 后运行 `brew install tldr` 安装。用法与 `man` 类似，在想要查询的命令前加上 `tldr`，比如查询 `tar` 的用法就运行 `tldr tar`。
- 其他实现：~~因为是使用 NodeJS 实现~~，官方的实现很慢。可以使用 Rust 实现 tealdeer 来代替。 [![tealdeer homebrew version](https://img.shields.io/homebrew/v/tealdeer?style=flat-square)](https://formulae.brew.sh/formula/tealdeer)
- 相关中文教程：[https://learnku.com/articles/23834](https://learnku.com/articles/23834)

## fasd [![fasd homebrew version](https://img.shields.io/homebrew/v/fasd?style=flat-square)](https://formulae.brew.sh/formula/fasd)

### 简介

虽说 `z` 快速转跳鼎鼎大名，但我用过最舒服的还是这个工具。只要你曾经访问过某个目录货文件，fasd 能让你只输入目录名或文件名就实现全路径的引用或转跳，而且转跳命令也是 `z`————例如，输入 `z dow` 即可从任意位置转跳到 `~/Download`。fasd 的命令还方便记忆：`f` 代表 file，`a` 代表 all，`s` 代表 show，`d` 代表 directory，完全没有上手难度。

### 资源

- 官方 GitHub：[https://github.com/clvv/fasd](https://github.com/clvv/fasd)
- 快速入门：可使用 Homebrew 安装。安装后可能需要配置 rc 文件（如 zsh 为 `.zshrc`），详细配置参考官方文档。
- 相关中文教程：[https://www.howtoing.com/fasd-quick-access-to-linux-files-and-directories](https://www.howtoing.com/fasd-quick-access-to-linux-files-and-directories)

## sfk [![sfk homebrew version](https://img.shields.io/homebrew/v/sfk?style=flat-square)](https://formulae.brew.sh/formula/sfk)

### 简介

违背 UNIX 哲学的 “瑞士军刀”，堪比 “要你命 3000”，但是确实好用。它可以批量转换文件换行符，集成并改进了大多数 GNU/Linux 的工具，快速启动一个简易的服务器，还可以转换 Word 文档……同时，还配套了很贴心的教程与资料——“只有想不到，没有做不到”。

软件有点 Old School，不喜勿喷。

### 资源

- 官方网站：[http://stahlworks.com/dev/swiss-file-knife.html](http://stahlworks.com/dev/swiss-file-knife.html)
- 快速入门：可使用 Homebrew 安装。输入 `sfk` 查看帮助，即可快速上手。

## httpie [![httpie homebrew version](https://img.shields.io/homebrew/v/httpie?style=flat-square)](https://formulae.brew.sh/formula/httpie)

### 简介

觉得 `curl` 太麻烦？想要在 CLI 快速收发数据包来测试 API？那就用**更现代**的 httpie 吧！httpie 有 GUI 版本（像 Postman 一样），也有 CLI 版本，命令用法非常丝滑。举个例子：`http -a USERNAME POST https://api.github.com/repos/httpie/httpie/issues/83/comments body="HTTPie is awesome! :heart:"`。

~~curl sucks!!~~

### 资源

- 官方网站：[https://httpie.io/cli](https://httpie.io/cli)
- 快速入门：可使用 Homebrew 安装。输入 `tldr http` 或 `tldr https` 查看帮助，即可快速上手。
- 相关中文教程：[https://linux.cn/article-10765-1.html](https://linux.cn/article-10765-1.html)

## 附录

暂时就想到这么多，到时候再添加吧。
