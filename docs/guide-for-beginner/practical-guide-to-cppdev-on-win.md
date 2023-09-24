# Windows C/C++ 开发环境配置实用指南

## 写在前面

HITsz 的各种 C/C++ 实验都使用 GNU GCC Toolchain 进行编译（C 语言实验，数据结构/算法实验，计网实验，数据库实验等等），学校提供的方案是使用 CodeBlock 进行开发，实际上体验很差，所以很多人会选择 VS Code 来开发，但是不少人对 VS Code 在 Windows 上的 C/C++ 环境配置感到头疼。

```txt
“是因为你们选择了离 C 最远的 OS，所以 C 也抛弃了你们”
———— 某 OSA 成员
```

GNU GCC 顾名思义就是运行在 GNU/Linux 上的编译器套件，其实 VS Code 在 Linux 上的 C/C++ 开发体验是即开即用的，并不需要 Windows 上这么繁琐的配置。

然而 GCC 并不是本来就支持在 Windows 上运行的，而是经过了一系列改造才能在 Windows 上正常执行编译出 Windows 可运行的执行程序二进制，这种改造来源于项目 MinGW（维护更新较慢，最初并不支持 x64），目前这个项目的最出名的继承者是 [MinGW-w64](https://www.mingw-w64.org/)，本篇将使用 MSYS2 进行 MinGW-w64 环境的配置

## 什么是 MSYS2

以及和 MinGW-w64 有何区别

### MinGW-w64 

 - 旨在支持 Windows 系统上的 GCC
 - 将 Linux 上 GCC 提供系统 API 转化，提供兼容 Windows 系统 API 的实现
 - 提供在 Windows 上链接和运行代码所需的一切

### MSYS2

MSYS2 提供了 GCC、mingw-w64、CPython、CMake、Meson、OpenSSL、FFmpeg、Rust、Ruby 等的最新版本的开发工具和环境，使用 `pacman` 包管理器来进行快速环境安装和配置

提供非常类 Unix 的开发体验

## 安装 MSYS2

点此 [MSYS2](https://msys2.org/) 进入官网或使用浏览器打开下面的链接

```
https://msys2.org/
```

在中间的 Installation 处点击链接下载安装即可


!!! info "记住你的安装位置"
    安装的位置默认在`C:\msys64`，如果进行了更改，待会请自行更换教程中的路径

## MSYS2 提供的环境（选读）

此章有兴趣的同学可以阅读 [MSYS2 Environment](https://www.msys2.org/docs/environments/)

该文档详细解释了不同环境的区别和使用的C 标准库有什么区别

本教程将使用 ucrt64 环境，能在编译期和运行期提供与 MSVC 编译出的二进制更好的兼容性

## 添加 MSYS2 到环境变量

++"鼠标右键"++ 点击你的开始菜单，打开一个管理员权限的 PowerShell，输入以下命令将 MSYS2 加入环境变量


!!! warning "注意"
    一定要使用 PowerShell ，而且是管理员权限，使用命令提示符会出现环境变量拿不到的问题，因为这两个东西语法有区别
    前面自定义路径的同学务必修改下面的命令，如果有其它 MinGW GCC 在环境变量里，请务必删掉

```powershell
SETX /M Path "$Env:Path;C:\msys64\usr\bin;C:\msys64\ucrt64\bin"
```

## 安装 GCC 等开发工具

!!! info
    这部分可能需要使用魔法上网的同学，可以参考下列命令在 msys2 bash 中设置代理
    ```
        export http_proxy=http://127.0.0.1:7890
        export https_proxy=http://127.0.0.1:7890
    ```

1. 在开始/所有程序中找到 UCRT64 环境的 msys2 , 如下图所示

![ucrt64](https://gitee.com/villard/wiki-images/raw/master/vscode-mingw/msys2ucrt64.webp)

2. 打开后输入下面命令更新 msys2 sourcelist，可能会需要重启 msys2

```bash
pacman -Syyu
```

3. 然后安装 GCC GDB Make

```bash
pacman -S mingw-w64-ucrt-x86_64-make mingw-w64-ucrt-x86_64-gcc mingw-w64-ucrt-x86_64-gdb
```

4. 安装完后，在右键开始菜单打开一个终端（不需要管理员权限）

输入`gcc --version`出现如下提示则为正常

![gcc](https://gitee.com/villard/wiki-images/raw/master/vscode-mingw/gccversion.webp)

!!! warning "注意"
    请务必注意命令返回里有 `Built by MSYS2 project`
    若不是如此，可能存在mingw gcc在环境变量里，请务必检查环境变量并且删除不必要的gcc

## VS Code 相关配置

### 安装 C/C++ 插件

如图，插件中搜索 C，安装 C/C++ 插件

![cplugin](https://gitee.com/villard/wiki-images/raw/master/vscode-mingw/cplugin.webp)

### 配置 VS Code

1. 用 VS Code 打开一个新的空文件夹

2. 新建一个 test.c 文件

3. 写一个 hello world

4. 点击右上角的调试运行（或使用快捷键 ++f5++）

5. 根据下图指示选择 GCC，然后你的 VS Code 就配置好了

6. 打个断点，调试运行你的程序


![debugger](https://gitee.com/villard/wiki-images/raw/master/vscode-mingw/debugger.webp)



![choosegcc](https://gitee.com/villard/wiki-images/raw/master/vscode-mingw/choosegcc.webp)

## 使用 Clangd 带来更美妙的 C/C++ 开发体验

VS Code 自带的微软 C/C++ 开发组件并非十分好用，他的错误提示不是很明确而且会有时延，对工程的难以进行 IntelliSence 等一系列问题，在此不详细列举

想要拥有更好的体验，在这里我们安利使用 Clangd 作为 C/C++ 的 Language Server 代替 C/C++ 插件对代码进行更好的补全和感知

1. 首先先关闭 C/C++ 插件的 IntelliSence

    ![intellisense](https://gitee.com/villard/wiki-images/raw/master/vscode-mingw/intellisense.webp)

2. 安装 Clangd 

    如前文所述，打开 MSYS2（UCRT64），输入下面命令安装 clangd
    
    ```bash
    pacman -S mingw-w64-ucrt-x86_64-clang-tools-extra
    ```

    !!! warning "注意"
        右键开始打开终端，输入`clangd --version`确认此时 Clangd 在你的环境变量中

3. 安装 Clangd 插件

    ![clangd](https://gitee.com/villard/wiki-images/raw/master/vscode-mingw/clangd.webp)

4. 确认 Clangd 运行

    ![clangdrunning](https://gitee.com/villard/wiki-images/raw/master/vscode-mingw/clangdrunning.webp)