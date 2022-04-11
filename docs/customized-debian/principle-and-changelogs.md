# HITsz LUG Customized Debian 设计原则与变更日志

这是一个经过修改的基于 Xfce4 的 Debian 11 系统。

本修改版系统的诞生离不开整个开源社区，如有空闲请给下面提到的 Repo 们点个星星或捐赠其作者/维护者。

## 设计原则

本系统的目标是提供一个易用稳定的 Linux 虚拟机，以供新手在 Windows 上实践 Linux 操作或完成各类实验。在做修改时，遵循的基本原则是：

- 集合常用软件，不管你会不会用到
- 审美要正常，现在是 2022 年不是 2002 年
- 要有中文，要本地化
- 贴合主流，网上资料齐全
- 要适合作为虚拟机使用，不能太卡
- 与 Windows 集成正常，可以互通剪切板/复制粘贴文件，要有 Xrdp 支持

## Change - V1.0

### 主题

选择了一款还算可以的主题 `Qogir`：

- [Qogir Theme](https://github.com/vinceliuice/Qogir-theme)
- [Qogir Icons](https://github.com/vinceliuice/Qogir-icon-theme)
- [Qogir Cursor](https://github.com/vinceliuice/Qogir-icon-theme/tree/master/src/cursors)

### 字体

加入了 `Cascadia Code` 以及 `Sarasa Mono CL` 分别作为默认等宽字体与默认系统字体。

- [Cascadia Code](https://github.com/microsoft/cascadia-code)
- [Sarasa Mono CL](https://github.com/be5invis/Sarasa-Gothic)

### 本地化支持

系统语言默认为中文。VS Code 安装了中文插件。系统内含基本的中文字体以防止乱码。

时区设置为东八区。面板右上角时钟格式与文件管理器时间格式设置为适合国人的形式。

默认的基于 `fcitx5` 的拼音输入法支持，亦可以通过桌面右上角的输入法选项切换五笔等其他输入方案。

导入了基于维基百科，搜狗，以及萌娘百科的词库。

- [中文维基百科词库](https://github.com/felixonmars/fcitx5-pinyin-zhwiki)
- [萌娘百科词库](https://github.com/outloudvi/mw2fcitx)
- [搜狗词库](https://github.com/CHN-beta/sougou-dict)

### 软件改动

#### GUI

添加了 VS Code 作为系统编辑器，Chromium 作为系统浏览器。

移除了自带的 Firefox 浏览器以及 LibreOffice 办公套件。

安装了 Xrdp 以支持基于 rdp 协议的远程桌面访问。

对 VS Code 进行了基本的配置与美化，详见设置。

- [VS Code 文档](https://code.visualstudio.com/docs)
- [Xrdp](https://github.com/neutrinolabs/xrdp)

#### CLI

设置了当前用户的 sudo 使用权限。

将 Home 文件夹下的中文目录都改成了英文。

安装好了现代版本的 Docker 以及 docker-compose 以便于环境部署。配置好了对应的 systemd 启动以及当前用户免 root 执行。源已被修改为道云源以加快国内访问速度。

安装了实用解压工具 atool，版本管理工具 git，默认构建工具链 build-essential（内含 C/C++ 语言编译器 gcc 与构建工具 make）。

安装了 `python-is-python3` 以将 python3 设置为默认 python 解释器。安装了 `pip` 并将源修改为清华源以加快装包速度。将 `~/.local/bin` 加入到 `PATH` 内以支持调用 pip 安装的包。

启动了 SSH 服务器，允许 X11 forward。

- [Docker 指南](https://yeasy.gitbook.io/docker_practice/introduction/what)
- [Python 文档](https://docs.python.org/zh-cn/3.9/)
- [Git 指南](https://www.progit.cn/)
- [atool](https://www.nongnu.org/atool/)

### 存在问题 & TODO

Linux 一贯地对高分屏 （HiDPI） 支持不佳，可能在小的高分辨率屏幕上有的字会看不清楚。在测试者的 15.6 寸 2k 屏幕上，除了输入法之外的 UI 大小一般是适合人眼的。

一套基于 Docker 与 VSCode 的 Container Remote 插件的学校各类实验环境配置指南 + 工作流正在计划中。
