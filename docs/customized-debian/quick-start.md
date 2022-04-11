# Linux 使用基本说明

## 什么是 Readme ?

顾名思义，“readme” 的意思就是 “看我”。如果你在某个目录里看到了一个写着 `readme`（或者是 `Readme`、`README`）的文件/文件夹，那多半这个文件是作者很想让你看看的，你也 **应该** 去看看这个文件/文件夹。

!!! notes
    在虚拟机里，这是一个 Markdown 格式的说明文件，可以通过点击 VS Code 右上角的 “打开侧边预览” 图标（或点击 Ctrl+K 随后接 V）以获得更好的阅读体验。

## 这是个什么系统？

这是一个经过修改的基于 Xfce4 的 Debian 11 系统。

本修改版系统的诞生离不开整个开源社区，如有空闲请给[变更列表](./principle-and-changelogs.md)提到的 Repo 们点个星星或捐赠其作者/维护者。

## 基本用户信息

- root 账户密码：`root-hitsz.lug`
- 普通用户：`tux` (在 `sudo` 和 `docker` 组中)
- 普通用户密码：`tux-hitsz.lug`

!!! danger
    **安装后请立即修改普通用户和 root 的密码**
    
!!! danger
    **安装后请立即修改普通用户和 root 的密码**
    
!!! danger
    **安装后请立即修改普通用户和 root 的密码**

修改普通用户密码的方法：

- 启动终端
- 输入 `passwd`
- 确保输入法在英文状态，输入并确认新的密码。输入过程中不会有任何显示与反应，打进去就可以了。

修改 root 用户密码的方法：

- 启动终端
- 输入 `sudo passwd`
- 确保输入法在英文状态，输入并确认新的密码。输入过程中不会有任何显示与反应，打进去就可以了。

## Linux 基础操作

- 中科大 Linux 101：<https://101.lug.ustc.edu.cn/>
- 工具使用：<https://linuxtools-rst.readthedocs.io/zh_CN/latest/base/index.html>
- Shell 入门：<http://billie66.github.io/TLCL/book/index.html>
- Linux 命令速查表：<https://262235.xyz/linux-command/>

命令行快速入门：

- `cd xxx`：进入路径 xxx，相当于在 Windows 里点资源管理器
- `ls`：查看当前目录下文件，相当于在 Windows 里看一眼资源管理器
- `ls -la`：查看当前目录下所有文件。在 Linux 中，文件名以点（`.`）开头的文件默认都是隐藏文件。
- `cat xxx`：将文件 xxx 的内容输出到屏幕上，相当于在 Windows 里拿记事本打开
- `./xxx`：执行 xxx 文件，相当于在 Windows 上双击执行可执行程序

在 Linux 里，我们不习惯用后缀名区分文件类型，但是你可以借助 `ls` 输出的颜色来分辨文件。一般来讲，蓝色的是目录，绿色的是可执行文件。

> 动手小测试：
>
> - 查看一下自己的家目录里的所有文件，`.bashrc` 是什么？
> - 查看一下 `~/apps` 跟 `~/Documents/Readmes` 里的所有文件，探索一下这两个目录吧！说不定你能找到我们留下的彩蛋。

## 一些方便的命令行工具

- `htop`：查看系统占用量
- `atool`：基于文件后缀名的智能压缩/解压工具，再也不用背 tar 参数了！
  - `atool -a xxx.tar.gz file_a file_b ...`：压缩
  - `atool -x xxx.tar.gz`：解压
- `code .`：启动 VS Code 并让 VS Code 打开当前目录
