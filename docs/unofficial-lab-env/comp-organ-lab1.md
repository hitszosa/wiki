# 计算机组成原理实验 1 环境指南（非官方）

以下指南适合 **WSL** /已有 Linux 虚拟机/已有远程 Linux 环境/已有双系统的同学使用，可以避免编译工具链（11G）或下载虚拟机镜像

## 实验需要的工具

- riscv64 GCC: 用于编译、汇编和链接
- spike: riscv 模拟器，用于执行 riscv 程序
- spike-pk: ProxyKernel 用于处理 spike 模拟器内的 IO

## 安装依赖

这些依赖基本上是个 Linux 项目都需要使用

实验工具安装需要

- make
- gcc
- g++
- git

通过以下命令可以安装

```sh
# Debian
sudo apt install build-essential git device-tree-compiler
# Ubuntu
sudo apt install build-essential git device-tree-compiler
# ArchLinux
sudo pacman -Sy base-devel git dtc
```

如果配置环境过程中出现“command not found.”可能是有依赖的工具没装，此时可以利用搜索引擎。

## 安装 riscv64 GCC

常用发行版有预编译的 gcc-riscv64-linux-gnu

```sh
# Debian
sudo apt install gcc-riscv64-linux-gnu
# Ubuntu
sudo apt install gcc-riscv64-linux-gnu
# ArchLinux
sudo pacman -Sy riscv64-linux-gnu-gcc
```

## 安装 spike-pk

在某个你想要的目录下使用 `git clone` 下载源码

```sh
git clone https://github.com/riscv-software-src/riscv-pk
# 如果连接 github 有问题可用国内镜像
git clone https://hub.fastgit.xyz/riscv-software-src/riscv-pk

cd riscv-pk
mkdir build
cd build
mkdir dist
../configure --prefix=$(pwd)/dist --host=riscv64-linux-gnu
make -j $(nproc)
make install
echo "export PATH=\$PATH:$(pwd)/dist/riscv64-linux-gnu/bin" >> ~/.bashrc # 如果使用别的 Shell 自行类比更改
```

## 安装 spike

使用 Arch Linux 的同学可以直接包管理器安装：`sudo pacman -Sy spike`

其他发行版需要编译安装

同样在你想要的目录下使用 git 获取源码

```sh
git clone https://github.com/riscv-software-src/riscv-isa-sim
# 如果连接 github 有问题可用国内镜像
git clone https://hub.fastgit.xyz/riscv-software-src/riscv-isa-sim

cd riscv-isa-sim
mkdir build
cd build
mkdir dist
../configure --prefix=$(pwd)/dist
make -j $(nproc)
make install
echo "export PATH=\$PATH:$(pwd)/dist/bin" >> ~/.bashrc # 如果使用别的 Shell 自行类比更改
```

接下来重启你的 shell

或使用任何方法使刚才我们对 PATH 环境变量的修改生效

## 验证安装正常

```sh
# GCC
riscv64-linux-gnu-gcc --version
# 输出类似
riscv64-linux-gnu-gcc (GCC) 11.1.0
Copyright (C) 2021 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

# spike
spike
# 输出类似
Spike RISC-V ISA Simulator 1.1.0

usage: spike [host options] <target program> [target options]
Host Options:

...

--dm-no-hasel         Debug module supports hasel
--dm-no-abstract-csr  Debug module won't support abstract to authenticate
--dm-no-halt-groups   Debug module won't support halt groups
--dm-no-impebreak     Debug module won't support implicit ebreak in program buffer

# spike-pk
spike $(which pk)
# 输出类似
bbl loader
tell me what ELF to load!
```

## 实验注意事项

1. 实验中使用 `riscv64-unknown-elf-gcc` 的地方全部改用 `riscv64-linux-gnu-gcc`，两者的区别在于使用的 ABI 不同，也就是 calling-convention 不同，对实验没有影响
2. `riscv64-linux-gnu-gcc` 默认是使用动态链接链接标准库的，因此如果需要在 spike 内运行，需要使用 `riscv64-linux-gnu-gcc -static` 来链接最后的可执行文件
3. 如果 `spike pk` 报找不到文件错误，可能是因为 spike 没有在 PATH 中找，简单解决办法是使用 `spike $(which pk)` 把绝对路径传给 spike

## 环境卸载清理

首先不建议卸载工具链，因为操作系统实验还要用，如果要卸载使用包管理器卸载即可。

卸载 spike，spike-pk 只需两个步骤

- 删除 riscv-pk, riscv-isa-sim 目录
- 在 `~/.bashrc` 删除安装时插入的 `export PATH=...`
