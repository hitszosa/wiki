---
authors:
  - shirok1
date: 2021-12-23
---

# 从零开始的 VirtualBox + Ubuntu 生活（雾

> 一点科普，一点吐槽

本文目的是让读者通过较为安全的虚拟机技术学习基础的 Linux 发行版安装能力 ~~管生不管养~~。

## 关于 VirtualBox

### 什么是虚拟机？

在计算机科学发展的过程中，出于调试等的需求，出现了模拟计算机硬件的软件程序，这样的程序被称为 “模拟机”。模拟机有只模拟 CPU 的，也有模拟完整的机器的。你可能猜到了，一些老游戏机（如 Nintendo Famicon、Gameboy）的模拟器也可以归入模拟机的范畴。当然，也有在 PC（此处特指 x86 架构下的微型计算机）上模拟一台 PC 的，最知名的两个例子是 Bochs 和 QEMU（jk 们应该会在操作系统课接触到）。进入新世纪后，x86 微机架构以其相对低廉的价格杀入服务器市场，抢下了许多原本小型机或大型机的市场份额。然而随着数据规模的扩大，出现了在 x86 微机上同时运行多个独立的操作系统的需求，而模拟机性能过于缓慢，无法适用于这种场景。后来出现了如 Xen 这样的半虚拟化方案，一定程度上实现了直接运行虚拟机的指令。在 2005 年，Intel 和 AMD 相继推出了基于硬件的虚拟化技术（hardware-assisted virtualization），分别称为 Intel VT-x 和 AMD-V。这项技术使得 x86 微机上的虚拟化软件可以使虚拟机直接访问一些硬件资源，虚拟化性能大幅提升。

!!! note ""
    另外，虚拟机管理程序还可分为 Type-1（直接管理虚拟化的操作系统也被虚拟化，如 WSL2、Xbox 360 等使用的 Hyper-V）和 Type-2（虚拟化系统运行在应用层，如本教程使用的 VirtualBox）两种 Hypervisor。

为方便叙述，下文统一将管理虚拟机的操作系统称为 **宿主操作系统（Host Operating System）**，将 **被虚拟化** 运行的操作系统称为 **客户操作系统（Guest Operating System）**

### 什么是 VirtualBox？

[**VirtualBox**](https://www.virtualbox.org/) 最初由 innotek GmbH 开发，后来该公司被 Sun Microsystems 收购（然后 Sun 又被收购），现在由 Oracle Corp.（即甲骨文公司，~~开源社区眼中的带恶人~~）开发，是一个以 GPLv2 协议（一个 “自由” 软件协议）开源的虚拟机软件（扩展包除外）。

!!! note "另外……"
    值得一提的是，Windows 平台上的大部分 “安卓模拟器”  *基于 VirtualBox 开发而来* ，本质上也是虚拟机。

### 下载 VirtualBox

[官网在此](https://www.virtualbox.org/wiki/Downloads)。根据你正在使用的操作系统（“宿主操作系统”）来选择对应的安装包。

### 安装 VirtualBox

由于在 Linux 下安装的方法在不同发行版之间较难通用（~~并且我没有 macOS 设备~~），这里仅给出 Windows 平台下安装的方法。

在官网点击 “Windows hosts” 来下载安装包。***TODO***

!!! warning "注意"
    VirtualBox 6.0 前的所有模式、6.0 后的默认模式 **与 WSL2 不兼容**，因为 WSL2 基于 Hyper-V（为 Type-1 Hypervisor）实现虚拟化，此时 VirtualBox 实际上运行在虚拟机中，因此需要通过特殊设置使 VirtualBox 使用 Hyper-V 作为虚拟化内核。

### 启用硬件虚拟化

现代虚拟机软件几乎全部基于硬件虚拟化开发，如果这一功能没有启用虚拟机性能会大打折扣。但通常这项功能并不会在出厂时开启。可以通过查看任务管理器 - 性能 - CPU 的详细信息中的 “虚拟化” 一栏来确定是否已开启硬件虚拟化。如显示 “无”，请自行百度开启方法。

## 关于 Ubuntu

### 什么是 Ubuntu?

[**Ubuntu**](https://ubuntu.com/) 由 Canonical Ltd. 基于 Debian 开发。Ubuntu *可能是* 目前在中国网络资源最丰富、最易于检索的 Linux 发行版。

### 下载 Ubuntu

Ubuntu 根据不同用途分为多个版本，其中只有 Ubuntu Desktop 自带图形化用户接口（GUI）。此外，Ubuntu 每隔 6 个月会发布一个新版本，其中被标记为 “长期支持”（LTS, Long-term Support）的版本有 **长达 5 年的官方支持**，但其他版本的官方支持时间通常只有 9 个月。本教程选用 Ubuntu 20.04.3 LTS 进行演示。可以从[这里](https://ubuntu.com/download/desktop)直达下载页面。

!!! tip "下载速度慢？"
    实际上我校 LUG 有自己的镜像站，对内网开放，下载速度更快，其中 Ubuntu 20.04 的光盘镜像可以在[这里](http://10.249.12.85/ubuntu-releases/20.04/)下载。不想记住这一长串地址的话，可以从 <https://lug.pub/> 跳转。

下载得到的文件后缀名为 `.iso`，是一个光盘镜像。本教程使用的 VirtualBox 支持直接在虚拟机的光驱中加载 `.iso` 文件。

!!! info ""
    由于历史原因，操作系统的安装程序往往以光盘镜像的形式发布（即使体积实际上已经超过了 DVD 的最大容量），但通常也会支持写入到 U 盘后从 U 盘启动安装程序。如果你希望将 Ubuntu 安装到物理机上，可以使用 [Rufus](https://rufus.ie/en/) 和 [balenaEtcher](https://www.balena.io/etcher/) 来完成写入到 U 盘的操作。（也有可以直接从 U 盘上存储的 `.iso` 文件启动的方案，如 [Ventoy](https://www.ventoy.net/cn/)）

---

## 正片

### 创建虚拟机

在安装之前，需要先点击 “新建” 使用向导创建虚拟机的配置文件。需要注意以下几点：

- 选择 “操作系统” 的时候要选择标有 “64-bit” 的版本

  ![操作系统设置界面](assets/vu/wizard_01_os_type.png)

- 内存最小设置为 4 GB，建议设置为 8 GB 或以上。

  ![内存设置界面](assets/vu/wizard_02_ram.png)

- 硬盘大小建议设置为 25 GB 以上，不建议选择立即分配硬盘空间（当然如果你想直接分配好空间防止后面别的东西占用也行）

  ![硬盘设置界面](assets/vu/wizard_03_disk.png)

向导结束后，打开虚拟机设置，在系统 - 主板选项卡中勾选 “启用 EFI”，在系统 - 处理器选项卡中将 CPU 核心数设置为 2 或以上，在显示选项卡中将显存调整至 128 MB，不要勾选 “启用 3D 加速”。

!!! info ""
    EFI 为 2014 年后生产的电脑的默认启动方式，与传统的启动方式差异极大。勾选该选项更贴近现代物理机的环境。

### 安装程序

接下来，点击 “启动” 启动这台虚拟机，会弹出选择启动盘的对话框。点击下拉菜单旁边的按钮，选择 Ubuntu 的安装镜像，再点击启动。

此时你会收到一条消息，告诉你此时你的键盘鼠标输入会被虚拟机独占。要想在控制宿主机和控制虚拟机之间切换，你需要输入指定的 “主机组合键”（默认是 ++right-ctrl++，也会在右下角显示）。

如果成功使用光盘引导，会显示如下图的启动菜单。显示这个菜单的程序叫 GRUB，未来也会使用这个程序来启动硬盘上的系统。

![Ubuntu 光碟镜像的 GRUB 菜单](assets/vu/bootup_02_grub.png)

每个选项代表的含义已经标注在截图上。此时不点击方向键便会进入第一个选项，开始验证安装媒介（installation medium）的完整性。待验证结束后就会正式进入安装程序。

安装程序的第一个界面是语言的选择。建议选择默认的 “English”。

??? note "原因"
    此处选择的语言会同时影响图形界面和终端中显示的语言。但 Linux 在纯文本模式下缺少对中文的支持。此处如果选择了中文，会使用中文来为个人目录下的 “音乐”、“下载” 等文件夹命名，还会使一些命令的输出无法正常显示，一旦系统出现故障将会很难在纯文本模式下进行修复。

![Ubuntu 安装菜单的语言页](assets/vu/setup_01_language.png)

另外，此时 VirtualBox 提示可以使用鼠标自动切换了，意思就是鼠标可以直接移出移入虚拟机，可以不用靠按 “主机组合键” ++right-ctrl++ 了。

窗口里还有一个 “Try Ubuntu” 的选项，此为 Linux 的 LiveCD 机制，类似于 Windows PE，将整个系统加载进内存，这里不用管，点击 “Install Ubuntu” 继续。

接下来是安装选项。Keyboard Layout 直接默认即可。注意 Normal installation 中多出来的软件 **都可以手动安装**，且在虚拟机上会 **大幅延长安装的时间**，此处建议选择 Minimal installation。Download updates while installing Ubuntu 可能会导致直接从国外服务器下载更新，建议安装完毕后选择国内镜像站再进行更新。下面的 third-party software 包括 MP3 解码器、无线网卡驱动等非自由开源软件，在物理机上安装时最好勾选，但虚拟机上没有必要安装。

![安装选项](assets/vu/setup_02_installation.png)

接下来是安装磁盘的选择。在虚拟机中，选择 “Erase disk and install Ubuntu” 即可。

!!! info ""
    在物理机下分区要考虑的因素较多，如果有详细讲解 Linux 下分区逻辑的需求可以单独开一篇讲解。

接下来是时区的选择。点击地图上的中国即可将时区设置为 “China/Shanghai”。

![设置时区](assets/vu/setup_03_timezone.png)

!!! info ""
    据说 Linux 时区有上海没有北京的原因要追溯到 1912 年北京天文台将中国分为五个时区，其中对应现在北京时间（东八区）的是 “中原时区”，代表城市即为上海。

接下来是初始用户设定。***username 不要出现中文！***（虽然好像也输不了中文）因为是虚拟机所以 “computer name” 可以随意。下面的密码必须要填写。因为初始用户为系统管理员（wheel 用户组成员），此处设定的密码在涉及系统操作（如安装软件）的时候也会用到，务必牢记。下面的自动登录仅影响进入桌面前是否要输入密码。

![初始用户设置](assets/vu/setup_04_user.png)

接下来就是漫长的等待了……嗯。

![安装中](assets/vu/setup_05_waiting.png)

安装结束后会提示 “remove installation medium”。如果是使用光盘安装的物理机，此时光驱托盘会自己弹出来，让你把光盘取走；用 U 盘安装就把 U 盘拔掉。当然对于虚拟机来讲，弹出光驱等于卸载光盘镜像，这里直接按 ++enter++ 继续重启即可。

![提示移除安装媒体](assets/vu/setup_06_remove_medium.png)

假如你能看到这样的登录界面，那就说明你成功的安装了 Ubuntu！点击用户名，输入你安装时设置的密码即可登录。

![登录界面](assets/vu/login_screen.png)

!!! info "Wayland"
    经测试，以 Wayland 为后端的桌面环境与 VirtualBox 兼容性更好，可以在登录界面点击用户名后在右下角更改桌面环境。

    ![更改桌面环境](assets/vu/login_screen_wayland.png)

### 增强工具配置

为了让虚拟机用起来更舒服，需要额外安装一些组件来让虚拟机可以和 VirtualBox 通信，完成诸如分辨率跟随窗口缩放、剪贴板共享、文件拖放的功能。

安装完成后首先会弹出来这样一个窗口，一直点击右上角的按钮即可。

![账户设定](assets/vu/config_01_oobe.png)

接下来，点击左下角的按钮打开程序列表，打开 Software & Updates（注意是左边这个圆圈连续的）。

![Software & Updates](assets/vu/config_02_software_and_updates.png)

在这个下拉菜单里选择 Other。

![选择 Others](assets/vu/config_03_other.png)

接着会弹出来一个窗口，让你直接选择下载自哪个镜像站。点击右上角的 Select Best Server 对这些服务器进行测速。

![服务器列表](assets/vu/config_04_server_list.png)

经过一段时间后，会选中最快的服务器。

![正在进行测速](assets/vu/config_05_testing_speed.png)

![选中最快服务器](assets/vu/config_06_fastest_server.png)

点击下面的 Choose Server，此时会提示你输入密码授权这个操作。切换完毕后点击 Close，此时会提示你当前本地软件索引已经过期，点击 Reload 刷新软件包缓存。

![提示刷新缓存](assets/vu/config_07_reload_cache.png)

因为 VirtualBox 的增强功能使用了定制的驱动，而 Linux 系统上的驱动模块需要针对不同版本单独编译和安装，因此接下来我们要配置编译和安装驱动模块的环境。点击左下角的按钮打开程序列表，翻到第二页打开 Terminal。

![打开终端](assets/vu/config_08_terminal.png)

输入以下命令：

``` sh
sudo apt upgrade -y # 更新所有软件包
sudo reboot # 可选，重启虚拟机，确保使用的是最新的内核
sudo apt install build-essential dkms linux-headers-$(uname -r) -y
```

注意执行这些指令时会要求你输入密码，并且这个密码不会显示出来（包括以 \* 的形式）。

![sudo 输入密码](assets/vu/config_09_sudo.png)

??? info "关于命令每一部分的含义"
    - `sudo` 是 Linux 命令行环境中的“以管理员身份运行”，它会让后面接着的命令有管理系统的权限。在 Linux 下几乎任何权限都要明切给予。
    - `apt` 是 Debian/Ubuntu 自带的软件包管理系统，`upgrade` 命令表示升级所有软件，`install` 命令表示安装后面跟着的软件。
    - `build-essential` 是 `apt` 的一个“元”软件包（即软件组合），包含了在 Ubuntu 上编译软件所需的软件，如 GCC。
    - `dkms` 是一个内核模块编译系统。Linux 不同版本内核模块不通用，dkms 可以自动为不同的 Linux 内核版本编译内核模块，生成对应的二进制文件。
    - `linux-headers-$(uname -r)` 代表当前所运行的内核的头文件，编译内核模块需要。后面的 `$(uname -r)` 实际上是 shell 的语法，`uname -r` 命令的输出是当前内核版本，通过这个语法将内核版本接到 `linux-headers-` 后面。
    - `-y` 表示自动同意安装，`apt` 默认会打断安装征求同意。

安装结束后，点击右上角，点击 Power Off / Log Out，然后点击 Power Off...。

![Power Off](assets/vu/config_10_power_off.png)

再点击 Restart 重启虚拟机。

![Restart](assets/vu/config_11_restart.png)

重新登录虚拟机后点击设备菜单里的安装增强功能，VirtualBox 会向虚拟机挂载含有增强功能安装程序的光盘镜像。

![安装增强功能](assets/vu/config_12_install_additions.png)

接下来会弹出提示自动运行光盘镜像里的安装程序。

![自动运行](assets/vu/config_13_auto_run.png)

点击 Run，然后验证密码后就会进行安装，期间不需要干涉。安装完成后会显示这样的提示，重启虚拟机。

![安装完成](assets/vu/config_14_install_complete.png)

??? info "假如没有自动运行的提示"
    点击左边的光盘图标会开始浏览光驱的内容，点击右上角的“Run Software”按钮就能重新显示这个提示。

    ![重新打开提示](assets/vu/config_13_alt.png)

重启之后，尝试改变虚拟机窗口的大小，你会发现虚拟机显示的画面会跟随着变化。这样就装完了增强功能。你可以在上面的设备菜单里开启对应的功能后自行尝试共享剪贴板和拖放。

??? tip "显示的字体太小看不清？"
    如果觉得显示的界面太小，可以在 Settings - Displays 里面更改 Scale（缩放比例）。如果觉得 100% 和 200% 都不合适，可以勾选下面的 Fractional Scaling 解锁 125% 等非整数的缩放比例。

    ![缩放设置](assets/vu/config_15_display_scale.png)
