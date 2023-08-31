# How to Install WSL?

> A Guide for Absolute Beginner

## 什么是 WSL？

WSL，全称 Windows Subsystem for Linux（适用于 Linux 的 Windows 子系统）。可以让开发者在 windows 系统下运行 GNU/Linux 环境，包括大多数命令行工具、应用程序。

## 为什么要安装 WSL？

> 为什么要在 windows 下安装 WSL，~~教我使用 windows 系统你们真的不是 windows user group？~~

### WSL 的有什么优势？

如果你是一个完全的初学者（absolute beginner），在电脑上安装 Linux 系统并进行双系统的使用会较于硬核，而我们的教程也会涉及到一些初学者非常陌生的词汇，比如引导、Legacy 和 UEFI、MBR 和 GPT、文件系统、系统时间与硬件时间等等。

相较于直接安装 Linux 系统，WSL 在安装上更为简便（堪称傻瓜式安装），且不会占用太多稀缺的硬盘空间（硬盘无论多大，硬盘空间都是稀缺资源）。WSL 提供了一种“无痛”的途径让初学者体验到几乎原汁原味的 Linux 系统。

当然，作为一个 ~~正经的~~ Linux 教程，我们也会教你使用真正的 Linux 系统，其中包括实机安装 Linux（包括双系统的使用），云服务器的使用、Linux 虚拟机、树莓派（包括其他单片机电脑）上 linux 的使用，这些教程正在紧急编写中。

### WSL 能做什么？

抛开体验 Linux 这一项，WSL 到底能做些什么呢？

- 你可以在 Windows 上「安装」你喜欢的 GNU/Linux 发行版
- 你可以直接在 Windows 上运行 `grep`、`awk`、`sed` 等 Linux 原生可执行文件
- 你可以在 Windows 上直接使用 Vim、Emacs 等工具，直接使用 Linux 版本的 Javascript/Node.js、Ruby、Python、C/C++、Rust、Go 等语言进行开发，直接运行 MySQL、Apache 等 Linux 原生应用和服务等。

最重要的一点：可以使用 Linux 命令直接操作 windows 系统的文件，个人的体验上，cmd/powershell 的命令很难用，且 Linux 的 shell 明显比 windows 的 cmd/powershell 强大。

## WSL 1 与 WSL 2

| 功能                                           |          WSL 1          |          WSL 2          |
| :--------------------------------------------- | :---------------------: | :---------------------: |
| Windows 和 Linux 之间的集成                    | :ballot_box_with_check: | :ballot_box_with_check: |
| 启动时间短                                     | :ballot_box_with_check: | :ballot_box_with_check: |
| 与传统虚拟机相比，占用的资源量少               | :ballot_box_with_check: | :ballot_box_with_check: |
| 可以与当前版本的 VMware 和 VirtualBox 一起运行 | :ballot_box_with_check: | :ballot_box_with_check: |
| 托管 VM                                        |           :x:           | :ballot_box_with_check: |
| 完整的 Linux 内核                              |           :x:           | :ballot_box_with_check: |
| 完全的系统调用兼容性                           |           :x:           | :ballot_box_with_check: |
| 跨 OS 文件系统的性能                           | :ballot_box_with_check: |           :x:           |

<!-- ![wsl1vswsl2](https://gitee.com/SoraShu/image/raw/master/image_0/wsl1vswsl2.png) -->

摘自[微软官方文档](https://docs.microsoft.com/zh-cn/windows/wsl/compare-versions#comparing-features)

其中”可以与当前版本的 VMware 和 VirtualBox 一起运行“纯粹扯淡。有使用虚拟机需求的同学谨慎安装 WSL。其中包括安卓模拟器。

可以看出除了跨操作系统文件系统的性能外，WSL 2 体系结构在多个方面都比 WSL 1 更具优势。若无具体说明，以下 WSL 专指 WSL 2。

## WSL 的安装

> ~~说了半天，这人终于是回到正题了~~

### 1.确认 windows 版本

- 只有 Windows 10 才能安装使用 WSL。Windows 7、8 或之前的任何版本都无法使用，请及时将系统版本更新至最新。
- 只有 Windows 10 版本 16215 或以后的版本才能够正常运行 WSL。你可以在「Windows 设置 > 系统 > 关于」处找到你的 Windows 10 操作系统版本。
- 只有 Windows 10 版本 18362 或 18363 以及以后的版本，或小版本号为 1049 的版本，才能够正常运行 WSL 2。需要明确，WSL 2 目前只能在 Windows 10 版本 1903、1909 和 2004 中使用（其中 1903 和 1909 仅支持 x64 系统），因此你需要将自己的 Windows 系统进行升级至合适的版本，才能使用正确的 Windows 10 版本安装 WSL 2。

### 2.启用系统功能

#### 启用适用于 Linux 的 Windows 子系统

以管理员身份运行 PowerShell，输入：

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
```

并回车执行。

按照提示重启电脑。

若报错，检查 PowerShell 是否是已管理员身份运行，若不是请重新以管理员身份运行 PowerShell，因为 PowerShell 的提权比较复杂。

#### 启用虚拟机功能

以管理员身份运行 PowerShell，输入：

```powershell
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

并回车执行。

按照提示重启电脑。

若报错，检查 PowerShell 是否是已管理员身份运行，若不是请重新以管理员身份运行 PowerShell，因为 PowerShell 的提权比较复杂。

### 3.下载运行 Linux 内核更新包

- [微软官方下载地址](https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi)
<!-- - []() -->

### 4.将 WSL 2 设置为默认版本

运行 PowerShell，输入：

```powershell
wsl --set-default-version 2
```

并回车执行。

### 5.下载并安装 Linux 发行版

打开微软商店，下载 Linux 发行版。这里我们选择被很多人所喜爱的 Ubuntu。

![Ubuntu-in-MSstore.png](https://gitee.com/SoraShu/image/raw/master/image_0/Ubuntu-in-MSstore.png)

点击下载。

> 其他发行版的下载见[微软官方文档](https://docs.microsoft.com/zh-cn/windows/wsl/install-manual#step-6---install-your-linux-distribution-of-choice)。

下载完成后，在开始菜单中点击下载好的 Linux 发行版进行安装，然后等待初始化完成即可。

初始化后，将提示你输入用户名与密码。接下去就可以愉快地使用 WSL 玩耍了。

## WSL 的管理

在 powershell 执行以下命令可查看安装的 Linux 发行版以及 WSL 版本。

```powershell
wsl -l -v
```

若你下载的发行版是 Ubuntu，则最终结果应该和我这边类似。

![wsl-l-v](https://gitee.com/SoraShu/image/raw/master/image_0/wsl-l-v.png)

~~请自行忽略背景~~

## 结语

至此，我们成功地在 windows 系统下跑起了 Linux，虽然只是个子系统，但我们离 Linux 更近了一步，争取早日从 wsl user group 加入 Linux user group。
