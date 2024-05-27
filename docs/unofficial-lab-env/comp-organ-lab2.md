# 计算机组成原理实验 2 环境指南（非官方）

以下指南适合有 Linux/MacOS 环境的同学使用。

笔者心态：对 Verilog 感到不适，并且想折腾一些花样。

## 实验需要的工具

- Verilator
- Chisel
- GTKWave

## 安装依赖

实验工具安装需要

- Bison
- Flex
- gcc
- git
- g++
- Java
- make
- sbt
- Scala
- SDKMAN
- Verilator

通过以下命令可以安装

```sh
# Debian/Ubuntu
sudo apt install build-essential git device-tree-compiler
# Ubuntu
sudo apt install build-essential git device-tree-compiler
# ArchLinux
sudo pacman -Sy base-devel git dtc
# OpenSUSE
sudo zypper in -t pattern devel_C_C++
```

如果配置环境过程中出现 "command not found." 可能是有依赖的工具没装，此时可以利用搜索引擎。

MacOS 需要安装 Homebrew 作为包管理器，请查阅 [Tuna Mirrors 的安装指南](https://mirrors.tuna.tsinghua.edu.cn/help/homebrew) 。

## 安装 Bison 和 Flex

Bison 和 Flex 是 GNU 提供的两个语法解析工具。Verilator 依赖这两个工具将 Verilog 代码编译成 C++ 的 class。

```sh
# Debian/Ubuntu
sudo apt install bison flex
# Fedora/CentOS
sudo dnf install bison flex
# ArchLinux
sudo pacman -Sy bison flex
# OpenSUSE
sudo zypper in bison flex
# MacOS
brew install bison flex
```

## 安装 Verilator

使用包管理器安装：

```sh
# Debian/Ubuntu
sudo apt install verilator
# Fedora/CentOS
sudo dnf install verilator verilator-devel
# ArchLinux
sudo pacman -Sy verilator
# MacOS
brew install verilator
```

但是有一些 Linux 的发行版，他会将包拆的比较奇怪，导致没法找到 `verilated.mk` 之类的情况发生。亦或者是版本比较老，则可以使用编译安装：

```sh
# 先 cd 到一个文件夹中，推荐 cd 到 $HOME/Downloads
git clone https://github.com/verilator/verilator
cd verilator
autoconf # 生成 ./configure 文件
./configure --prefix=$HOME/app/verilator # 配置 makefile 和安装路径
make -j `nproc` # 多核编译
make install  # 这会安装到 $HOME/app/verilator 中
echo 'export verilator_ROOT=$HOME/app/verilator' >> .bashrc # 追加环境变量
```

如果想要其他支持，请查阅 [Verilator 的安装文档](https://verilator.org/guide/latest/install.html) 。

Verilator 在编译过程中，需要用到 GNU 方言，因此推荐使用 Linux。当然 MacOS 下的 `brew install` 就已经可以正常工作了。

## 安装 SDKMAN

注意，并不一定要安装 SDKMAN。SDKMAN 是一个用于管理 Java' 相关开发环境的软件。
如果我们发行版的包管理器可以顺利安装 java, scala@2.12.13, sbt，那么实际上也不需要 SDKMAN。

```sh
# ArchLinux
paru sbt
sudo pacman -Sy openjdk
paru scala_2.12
# Fedora
sudo dnf install sbt openjdk scala-2.12
# MacOS
brew install sbt openjdk scala@2.12
```

注意，笔者没有查到 Ubuntu/Debian/OpenSUSE/CentOS 可以使用包管理安装 sbt 的方式。
因此，推荐通过 SDKMAN 安装 sbt。当然，scala 和 openjdk 是可以通过包管理器安装的。

可以通过下面这个命令安装：

```sh
curl -s "https://get.sdkman.io" | bash
```

安装过程中，注意看提示，会要求在安装完成后配置环境变量。

### 使用 SDKMAN 安装 Java

```sh
sdk install java
```

### 使用 SDKMAN 安装 Scala 2.12.13

```sh
sdk install scala 2.12.13
```

### 使用 SDKMAN 安装 sbt

sbt 是 Scala 项目的构建软件

```sh
sdk install sbt
```

## 安装 GTKWave

```sh
# Debian/Ubuntu
sudo apt install gtkwave
# Fedora/CentOS
sudo dnf install gtkwave
# ArchLinux
sudo pacman -Sy gtkwave
# OpenSUSE
sudo zypper in gtkwave
# Windows
winget install gtkwave
```

如果 Windows/MacOS 作为宿主系统，应该是在 Windows/MacOS 中安装 GTKWave，然后在 Windows/MacOS 打开 `.vcd` 文件。
如果是 ssh 连接的 linux，可以使用 CyberDuck 或者是 MountainDuck（付费）或者是 sshfs 挂载目录，然后在 宿主机上 `gtkwave <.vcd>` 或者是双击打开，笔者就是用的这种方式。

### MacOS 下安装 GTKWave

MacOS 下安装 GTKWave 是一件非常令人感到疑惑的事情。MacOS 有着地狱一般的向下兼容问题。

通过`brew install`并不能直接安装 GTKWave,
不过可以通过`brew tap randomplum/gtkWave && brew install --HEAD randomplum/gtkwave/gtkwave`这样安装。这种安装方式并不推荐，因为`randomplum/gtkwave`这个 tap 提供的 GTKWave 并不支持 tcl 脚本。

但是我们可以使用 Nix 或者 MacPort 安装 GTKWave（支持 tcl 脚本）。

下面两种方式选一种就行了。

#### Nix

NixOS 是一个更加强大的 Linux 发行版，感兴趣有可以进一步了解。NixOS 上有一种环境管理的工具叫做 Nix，
但是 Nix 作为环境管理工具现在已经支持了 Linux（可以是不同于 NixOS 的其他发行版）和 MacOS。

通过这行命令安装 Nix。安装过程中请注意看提示。

```sh
sh <(curl -L https://nixos.org/nix/install)
```

下面是通过 Nix 安装 GTKWave。

```sh
# i 表示 install
# A 表示 attr 这告诉 nix-env 通过它的属性名来选择软件包
nix-env -iA nixpkgs.gtkwave
```

下面是卸载 Nix 下的 GTKWave。

```sh
nix-env -e gtkwave
```

当然 Nix 的用法远不止于此，但是就先介绍到这里。
Nix 可以用的很优雅，但是这超过了本文的范围了。

想要卸载 Nix？请翻阅 [Nix 文档](https://nixos.org) 。

#### MacPort

MacPort 是 MacOS 上老牌的包管理器了，现在 Homebrew 比较流行。
但是 Homebrew 在安装 GTKWave 上表现的并不顺利。

在 [这里](https://www.MacOSports.org/install.php) 下载 MacPort 并安装（注意要选择当前系统的版本）。

```sh
sudo port install gtkwave
```

想要卸载 MacPort？请翻阅 [MacPort 文档](https://guide.MacOSports.org/chunked/installing.MacOSports.uninstalling.html) 。

## Chisel(Scala) IDE 的选择

Chisel 只是 Scala 中的一个库。因此，只要 IDE 能支持 Scala，那么自然也是支持 Chisel 了。
但是一些 IDE 如 VSCode/JetBrains IDEA 会对 Chisel 语法有更加好的 highlight 支持。

经观察：对于稍微大一些的 Chisel 项目（小学期级别）, VSCode + Metals 会很卡，建议使用 JetBrain IDEA。

### VSCode 对 Chisel(Scala) 的支持

下面是笔者使用的 VSCode 插件:

- Chisel Syntax
- Scala (Metals)
- Scala Snippets
- Scala Syntax (official)

### VSCode 对 Chisel(Scala) 的支持

下面是笔者使用的 JetBrain IDEA 插件:

- Scala

## example

下面这个例子可以在 [这里](https://github.com/KINGFIOX/chisel-verilator-example) 找到。

### 创建 Chisel 项目

Chisel 是 Scala 的一个库并且目前为止并没有一个真正的 Chisel IDE。我们可以通过官方的 Chisel-template 创建我们的项目。

```sh
git clone https://github.com/chipsalliance/chisel-template.git <my-chisel-project>
```

### 编写 Chisel 代码

chisel-template 项目下天然的提供了一个 GCD 硬件 module。

```scala
/// src/main/scala/gcd/GCD.scala
package gcd

import Chisel3._
// _root_ disambiguates from package Chisel3.util.circt if user imports Chisel3.util._
import _root_.circt.stage.ChiselStage

/**
  * Compute GCD using subtraction method.
  * Subtracts the smaller from the larger until register y is zero.
  * value in register x is then the GCD
  */
class GCD extends Module {
  val io = IO(new Bundle {
    val value1        = Input(UInt(16.W))
    val value2        = Input(UInt(16.W))
    val loadingValues = Input(Bool())
    val outputGCD     = Output(UInt(16.W))
    val outputValid   = Output(Bool())
  })

  val x  = Reg(UInt())
  val y  = Reg(UInt())

  when(x > y) { x := x - y }
    .otherwise { y := y - x }

  when(io.loadingValues) {
    x := io.value1
    y := io.value2
  }

  io.outputGCD := x
  io.outputValid := y === 0.U
}

/**
 * Generate Verilog sources and save it in file GCD.v
 */
object GCD extends App {
  ChiselStage.emitSystemVerilogFile(
    new GCD,
    firtoolOpts = Array("-disable-all-randomization", "-strip-debug-info")
  )
}
```

上面这个 `class GCD extends Module` 继承了 Chisel 的 Module 抽象类。在这个类里面，我们实现了 GCD 模块。
对应的也就是 Verilog 中的 `module GCD();`。在这个 `class GCD` 中，我们描述了 GCD 的行为。

除了一个 `class GCD` ，下面还有一个 `object GCD` 单例。这个单例继承了 App, App 相当于是 Scala 中的 `class Main`。
`object GCD` 中，我们执行了 sv file 的发射。然后我们可以通过执行 `sbt "runMain gcd.GCD"` 运行 sv file 的发射。
最后，我们就可以在根目录下看到对应的 GCD.sv 文件。

```sv
// Generated by CIRCT firtool-1.62.0
module GCD(
  input         clock,
                reset,
  input  [15:0] io_value1,
                io_value2,
  input         io_loadingValues,
  output [15:0] io_outputGCD,
  output        io_outputValid
);

  reg [15:0] x;
  reg [15:0] y;
  always @(posedge clock) begin
    if (io_loadingValues) begin
      x <= io_value1;
      y <= io_value2;
    end
    else if (x > y)
      x <= x - y;
    else
      y <= y - x;
  end // always @(posedge)
  assign io_outputGCD = x;
  assign io_outputValid = y == 16'h0;
endmodule
```

- 这里是 Scala 教程：https://docs.scala-lang.org/zh-cn/
- 这里是 Chisel 教程：https://www.chisel-lang.org/docs/cookbooks/cookbook

### 使用 Verilator C++ 调试 Chisel 生成的 Verilog

下面是一个 C++ 下的 testbench。

```cxx
#include "VGCD.h"
#include "verilated.h"
#include "verilated_vcd_c.h"

#include <cstdlib>
#include <iostream>

// 一个正确的 gcd 实现
int gcd(uint16_t a, uint16_t b)
{
    while (b != 0) {
        int remainder = a % b;
        a = b;
        b = remainder;
    }
    return a;
}

int main(int argc, char** argv)
{
    Verilated::commandArgs(argc, argv);
    Verilated::traceEverOn(true); // 启用波形跟踪

    size_t fail_cnt = 0;
    size_t success_cnt = 0;

    auto dut = std::make_unique<VGCD>();
    VerilatedVcdC* vcd = new VerilatedVcdC();
    dut->trace(vcd, 99); // 设定跟踪级别
    vcd->open("gcd.vcd"); // 打开VCD文件

    // 重置设备
    dut->reset = 1;
    dut->clock = 0;
    for (int i = 0; i < 5; i++) {
        dut->clock = !dut->clock;
        dut->eval();
        vcd->dump(10 * i); // 记录时间点
    }
    dut->reset = 0;

    srand(time(NULL));
    uint16_t x = rand();
    uint16_t y = rand();

    // 主仿真循环
    for (int cycle = 0; cycle < 400; cycle++) {
        dut->io_loadingValues = (cycle == 5);
        dut->io_value1 = x;
        dut->io_value2 = y;

        dut->clock = 1;
        dut->eval();
        vcd->dump(10 * cycle + 5);

        dut->clock = 0;
        dut->eval();
        vcd->dump(10 * cycle + 10);

        dut->io_loadingValues = 0;
    }

    // 收集结果和清理
    uint16_t top_z = dut->io_outputGCD;

    dut->final();
    vcd->close(); // 关闭VCD文件

    if (uint16_t z = gcd(x, y); z == top_z) {
        std::cout << "success" << std::endl;
    } else {
        std::cout << "fail" << std::endl;
        std::cout << "x: " << (int)x << std::endl;
        std::cout << "y: " << (int)y << std::endl;
        std::cout << "gcd(x, y): " << (int)z << std::endl;
        std::cout << "dut(x, y): " << (int)top_z << std::endl;
    }

    return 0;
}
```

### 编译 C++ 和 Verilog

```sh
verilator -Wall --cc --trace -Iobj_dir -Wno-UNUSEDSIGNAL GCD.sv --exe tb.cxx # 会生成 obj_dir 文件夹，这是将 Verilog 编译成了 C++ 的 class
make -C obj_dir -f VGCD.mk
```

这样我们就用 Verilator 编译完成了 C++ 和 Verilog。我们可以在 obj_dir 文件夹下面找到 VGCD 可执行文件。
我们可以通过 `./VGCD` 执行可执行文件。执行完成以后，可以看到输出 `success` 字样并且在项目的根目录下生成了 `gcd.vcd` 文件。
我们可以使用 VSCode 的 wavetrace 插件（付费）打开 `.vcd` 文件。

详细的用法查阅 [Verilator 的使用教程](https://itsembedded.com/dhd_list) 。

## 使用 GTKWave 打开波形仿真文件

```sh
gtkwave gcd.vcd
```

### 使用 tcl 脚本

每次我们使用 GTKWave 调试波形图的时候。每次打开需要一个步骤：选中信号，然后 append/insert。
这很不优雅。尤其是：当我们需要频繁的调试。打开 GTKWave -> 选中信号 -> a/i -> 看波形 -> 改代码 -> make run -> 打开 GTKWave ->
选中信号 -> a/i -> 看波形 -> ...

但是，实际上，"选中信号 -> a/i" 这个步骤我们可以实现的更加自动化一些 —— 编写 tcl 脚本！
实现打开 GTKWave 的时候，就将所有的信号 append/insert 到屏幕上。

创建一个 `load_all_waves.tcl` 文件，写入以下内容，保存。

```tcl
# load_all_waves.tcl
# Add all signals
set nfacs [ gtkwave::getNumFacs ]
set all_facs [list]
for {set i 0} {$i < $nfacs } {incr i} {
  set facname [ gtkwave::getFacName $i ]
  lappend all_facs "$facname"
}
set num_added [ gtkwave::addSignalsFromList $all_facs ]
puts "num signals added: $num_added"

# Zoom full
gtkwave::/Time/Zoom/Zoom_Full

# Print
# set dumpname [ gtkwave::getDumpFileName ]
# gtkwave::/File/Print_To_File PDF {Letter (8.5" x 11")} Minimal $dumpname.pdf
```

然后我们可以通过下面这个命令打开`.tcl`和`.vcd`：

```sh
# 可以通过 gtkwave -h 知道参数的含义：
# -S, --script=FILE. specify Tcl command script file for execution
# -f, --dump=FILE. specify dumpfile name
gtkwave -S load_all_waves.tcl -f gcd.vcd
```
