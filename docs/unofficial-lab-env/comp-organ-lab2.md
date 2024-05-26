# 计算机组成原理实验 2 环境指南（非官方）

以下指南适合有 linux 环境的同学使用。

笔者心态：对 verilog 感到不适，并且想折腾一些花样。

## 实验需要的工具

- verilator
- chisel
- gtkwave

## 安装依赖

实验工具安装需要

- bison
- flex
- gcc
- git
- g++
- java
- make
- sbt
- scala
- sdkman
- verilator

通过以下命令可以安装

```sh
# Debian
sudo apt install build-essential git device-tree-compiler
# Ubuntu
sudo apt install build-essential git device-tree-compiler
# ArchLinux
sudo pacman -Sy base-devel git dtc
# opensuse
sudo zypper in -t pattern devel_C_C++
```

如果配置环境过程中出现“command not found.”可能是有依赖的工具没装，此时可以利用搜索引擎。

## 安装 bison 和 flex

bison 和 flex 是 gnu 提供的两个语法解析工具。
（verilator 依赖这两个工具将 verilog 代码编译成 c++ 的 class）

```sh
#opensuse
sudo zypper in bison flex
```

## 安装 verilator

```sh
# 先 cd 到一个文件夹中，推荐 cd 到 ~/Downloads
git clone https://github.com/verilator/verilator
cd verilator
autoconf         # Create ./configure script
./configure      # Configure and create Makefile
make -j `nproc`  # Build Verilator itself (if error, try just 'make')
sudo make install  # 这会安装到 /usr 中
```

如果需要一些个性化，比方说安装到非`/usr`目录，或者想要其他支持，请查阅 verilator 的安装文档 https://verilator.org/guide/latest/install.html#git-quick-install

( 因为 verilator 在编译的时候，需要使用 gnu 方言，因此推荐使用 linux )

## 安装 sdkman

```sh
curl -s "https://get.sdkman.io" | bash
```

安装过程中，注意看提示

## 安装 java

```sh
sdk install java
```

## 安装 scala 2.12.13

```sh
sdk install scala 2.12.13
```

## 安装 gtkwave

```sh
# opensuse
sudo zypper in gtkwave
# windows
winget install gtkwave
# mac
brew install gtkwave
```

如果 windows/mac 作为宿主系统，应该是在 windows/mac 中安装 gtkwave，然后在 windows 打开 `.vcd` 文件。

但是 mac 实际上使用 gtkwave 会有点问题（无法双击打开）。但是可以在 shell 中 `gtkwave <.vcd>`打开。

如果是 ssh 连接的 linux，可以使用 CyberDuck 或者是 MountainDuck（付费）或者是 sshfs 挂载目录，然后在 宿主机上 `gtkwave <.vcd>` 或者是双击打开。（笔者就是用的这种方式）

## 创建 chisel 项目

chisel 是 scala 的一个库并且目前为止并没有一个真正的 chisel IDE。我们可以通过官方的 chisel-template 创建我们的项目。

```sh
git clone https://github.com/chipsalliance/chisel-template.git <my-chisel-project>
```

然后在 vscode 中打开我们的 chisel 就行了。当然也可以用其他的 IDE (neovim is the best ide in the world!)

## 调试 chisel 生成的 verilog

可以使用 C++ 和 verilator 调试我们的 module。

这是 verilator 的使用教程：https://itsembedded.com/dhd_list/

## 使用 gtkwave 打开波形仿真文件

```sh
gtkwave mul.vcd
```
