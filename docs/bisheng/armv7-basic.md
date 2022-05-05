# 毕昇杯专用 ARMv7 汇编急救

## 总览

> 主要参考文章: <https://thinkingeek.com/arm-assembler-raspberry-pi/>
>
> **绝大部分示例代码都是从此文章抄来的**
>
> 这篇参考文章偏向于使用 ARM 汇编进行开发的入门，期望读者水平为学过 C，还教了一些算法啥的。作者还有一本书，200 来页，书比网站上的更加系统，建议看书。但是网站上还有一些后续章节 (比如 SIMD) 是书里没有的。
>
> 当然如果我写错了什么，那必是我的问题，不关作者事。

这是一份对 ARMv7 汇编的急救指南，目的是以最快速度提供 "实现一个 target 到 ARMv7 的 C 编译器" 所需要的 ARM 汇编知识。本文章假设读者熟悉汇编语言与 C，至少看过 CSAPP 的第 4 章，并且了解一些编译技术。

本文章 **没有** :

- 任何对 **不同模式** 的介绍与说明 (也就是只考虑 User 模式)
- 任何对 **浮点运算** 的介绍与说明
- 对一些指令的完全介绍，比如 ldm/stm
- 对规范的完整解释

本文章的介绍顺序大体按照那本书里的顺序，为了方便查阅，根据主题分类索引如下：

- 总览 & 框架：[基本架构](#基本架构)
- 寄存器：[寄存器](#寄存器)
- 分支：[分支](#分支-branching), [控制结构的翻译](#实现控制流), [条件执行](#条件执行)
- 内存与取址模式：[内存](#内存), [基本取址模式](#更紧凑的指令--取址模式), [load/store 特有的取址](#数组与结构体)
- 函数：[函数基本](#函数), [递归函数](#栈), [函数局部变量](#局部变量)
- 动手实践：[用 GDB 调试汇编程序](#gdb), [用 GCC 交叉编译 C 代码到 ARMv7 可执行文件并反编译](#动手过程)

## 基本架构

### 第一个程序 & 如何运行

下面是一个返回值为 `2` 的简单的汇编程序：

```asm
/* -- first.s */
/* 这是注释 */
.global main   /* 一个程序的入口 (entry) 必须是全局的 (global) */
.func main     /* 标记 label ’main’ 为一个函数  */

main:            /* 这是 label main */
    mov r0, #2   /* 将字面量 2 放入寄存器 r0 中 */
    bx lr        /* 从 main 中 "返回" */
```

简单地将其编译，然后链接为可执行程序：

```console
$ as -g -mfpu=vfpv2 -o first.o first.s
$ gcc -o first first.o
```

执行它并查看其返回值：

```console
$ ./first ; echo $?
2
```

如果懒得每次都打命令，可以写一个简单的脚本：

```sh
#!/bin/bash
as -g -mfpu=vfpv2 -o $1.o $1.s
gcc -o $1 $1.o
rm $1.o
./$1 ; echo $?
```

### 解释

注释不可以嵌套。

`.`开头的是给 GNU Assembler (简称 GAS) 的汇编器指令 (directive), 不是 ARM 汇编指令. (类比 C 的宏)

`global` 跟 linkage 有关，是给链接器看的; 由 GCC 负责调用链接器，就不用我们手动指定库了。

每一行 ARM 汇编都长这样：

```asm
    标号：指令 参数 注释
```

> 它们的英文是 label: instruction parameters comments

它们全都是可选的，不一定都会出现。特别地，只有单独标号的一行将标号绑定到下一行，这样子就可以将多个标号绑定到同一行上：

```asm
A:  mov r0, #1 /* 标号 A 绑定到这条指令 */
B: 
    mov r0, #2 /* 标号 B 绑定到这条指令 */
C:
D:
E:
    mov r0, #2 /* 标号 C, D, E 绑定到这条指令 */
```

汇编里的空白是无所谓的，参数依逗号分隔，立即数 (immediate，类比 C 的字面量 (literal)) 以 `#` 开头，参数中用作目的地的参数常常在最左边 (如这里的 `r0`).

单行注释一般也可以用 `@` 打头，类似 bash/python 里的 `#` 跟 C++ 里的 `//`.

### 直接使用 LD

TODO

## 寄存器

对树莓派 2B 来说，除开特殊的寄存器，共有 16 个通用整数寄存器以及 32 个通用浮点寄存器。每一个通用整数寄存器 (分别叫 `r0` 到 `r15`) 都是 32 位长的。本章只讨论整数寄存器。

汇编里的数据采用补码表示，立即数同理。

有些通用寄存器并不是真的 "通用" -- 它们在特定上下文有特定用途：

- `r13` 也被称为 `sp` (Stack Pointer)
- `r14` 也被称为 `lr` (Link Register)
- `r15` 也被称为 `pc` (Program Counter)

关于 `fp`, 一般是 `r11`, 但是官方貌似没有指定，我看到的也有用 `r7` 的，所以你随便挑一个就好了。

下面的例子展示了寄存器的基本用法：

```asm
/* -- sum01.s */
.global main

main:
    mov r1, #3      /* r1 ← 3 */
    mov r2, #4      /* r2 ← 4 */
    add r0, r1, r2  /* r0 ← r1 + r2 */
    bx lr
```

## 内存

对 ARM（更广义的说，对于大多数 RISC 指令集），所有指令的参数必须是立即数或者是寄存器，而不是像 x86 一样可以直接塞内存地址。用来将内存特定位置的值读入寄存器的指令是 `ldr` (load register), 用来写回去的是 `str` (store register).

### 地址

地址是内存里特定位置的名字。在 ARMv7，地址是一个 32 位长的整数（即 1 word），标识内存中每一个字节。对汇编程序而言，内存是扁平的。

当从内存读/存数据时，地址的计算方式 (即取址模式，addressing mode) 有很多种，本章只介绍一种：通过寄存器取址。

汇编里的标号实质上被实现为这行汇编代码对应的二进制码在可执行文件里的地址。所以我们可以把标号当作 "地址的立即数" 来看。

我们可以使用汇编器指令来申请大块内存：

```asm
.balign 4       @ 对齐到 4 字节，Byte ALIGN
myvar1:         @ 标号，接下来就可以通过该标号获得数据的地址
    .word 3     @ 留出 1 个 "word" 的空白 (32bit) 并将其设为补码 3
```

程序的数据一般会跟代码分开放在内存的不同区域 (节，section). 使用 `.data` 让汇编器把下面的东西放在数据节 (data section) 里，使用 `.text` 让汇编器把下面的东西放在代码节 (text section) 里。（在某些情况下，并不需要区分代码节和数据节）

### Load

下面的程序定义两个内存中的变量 `myvar1` 和 `myvar2`, 分别赋予其初始值 3 和 4，然后取其值，相加，返回作为错误码。

```asm
/* -- load01.s */

/* -- Data section */
.data

/* 确保数据被对齐到 4 字节 */
.balign 4
/* 定义 myvar1 的存储空间 (storage) */
myvar1:
    /* 一个值为 3 的 4 byte 整数 */
    .word 3

/* 确保数据被对齐到 4 字节 */
.balign 4
/* 定义 myvar2 的存储空间 (storage) */
myvar2:
    /* 一个值为 4 的 4 byte 整数 */
    .word 4

/* -- Code section */
.text

/* 确保代码被对齐到 4 字节 */
.balign 4
.global main
main:
    ldr r1, addr_of_myvar1 /* r1 ← &myvar1 */
    ldr r1, [r1]           /* r1 ← *r1 */
    ldr r2, addr_of_myvar2 /* r2 ← &myvar2 */
    ldr r2, [r2]           /* r2 ← *r2 */
    add r0, r1, r2         /* r0 ← r1 + r2 */
    bx lr

/* 用来访问数据的标号 */
addr_of_myvar1 : .word myvar1
addr_of_myvar2 : .word myvar2
```

注意最下面的 `addr_of_*`. 即使标号 `myvar1` 已经包含了 myvar1 的地址，但是因为这个标号在 `.data` 节里，所以我们不能跨节，在 `.text` 节里使用它。我们必须在 `.text` 节里再做两个标号，分别放它们的地址，然后用标号来访问这些地址 -- 现在 `addr_of_myvar1` 是一个 "地址", 这个 "地址" 指着一个 word，这个 word 的值是 `myvar1` 的地址，类似于一个二级指针。

> 总结：**标号不能跨节使用**.

上面的 "地址" 打了引号：因为重定位 (relocation) 的缘故，它并不是一个真的地址，而是一个待填的 "坑"; 这个坑将会在链接时被链接器补上。

要把寄存器里的值当做地址传给 `ldr` 指令，我们只要用中括号把它括起来就可以了。（中括号即为 “取地址” 操作）

但是这样子很烦人，所以汇编器可以帮我们自动处理不同节的重定位：我们不需要像上面一样补上最后两行然后跳两次 -- 我们可以直接使用 `=<标号>` 的形式来得到不同节里的标号值。具体使用可见下节

### Store

下面的程序定义两个内存中的变量 `myvar1` 和 `myvar2`, 赋予其初始值 0，然后使用 str 指令赋予其值 3, 4，然后取其值，相加，返回作为错误码。

```diff
/* -- store01.s */

/* -- Data section */
.data

.balign 4
myvar1:
    .word 0

.balign 4
myvar2:
    .word 0

/* -- Code section */
.text

.balign 4
.global main
main:
+    ldr r1, =myvar1        /* r1 ← &myvar1 */
    mov r3, #3             /* r3 ← 3 */
+    str r3, [r1]           /* *r1 ← r3 */
+    ldr r2, =myvar2        /* r2 ← &myvar2 */
    mov r3, #4             /* r3 ← 4 */
+    str r3, [r2]           /*  *r2 ← r3 */ 

    /* 跟之前一样 */
    ldr r1, =myvar2        /* r1 ← &myvar1 */
    ldr r1, [r1]           /* r1 ← *r1 */
    ldr r2, =myvar2        /* r2 ← &myvar2 */
    ldr r2, [r2]           /* r2 ← *r2 */
    add r0, r1, r2
    bx lr
```

注意上面代码的两个 str 指令的参数顺序，我们写 `str A, [B]`, 是将 `A -> *B`, 而不是像普通指令一样目的地在左边。

## GDB

> If you develop C/C++ in GNU/Linux and never used gdb, shame on you.

本章介绍如何使用 GDB 方便地调试汇编程序。

启动 GDB:

```console
$ gdb --args ./store01
GNU gdb (GDB) 7.4.1-debian
Copyright (C) 2012 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later 
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "arm-linux-gnueabihf".
For bug reporting instructions, please see:
...
Reading symbols from /home/roger/asm/chapter03/store01...(no debugging symbols found)...done.
(gdb)
```

现在我们进入了 GDB 的交互模式。

退出 GDB（缩写命令为 `q`）: 

```
(gdb) quit
```

开始运行程序 -- 这会跳过所有 C 运行时库的初始化，直接在标号 `main` 前停住。

```
(gdb) start
Temporary breakpoint 1 at 0x8390
Starting program: /home/roger/asm/chapter03/store01 

Temporary breakpoint 1, 0x00008390 in main ()
```

反汇编：

> 使用 disas 作为简写也可

```
(gdb) disassemble
Dump of assembler code for function main:
=> 0x00008390 :	ldr	r1, [pc, #40]	; 0x83c0 箭头指向**将要**被执行的指令
   0x00008394 :	mov	r3, #3
   0x00008398 :	str	r3, [r1]
   0x0000839c :	ldr	r2, [pc, #32]	; 0x83c4 
   0x000083a0 :	mov	r3, #4
   0x000083a4 :	str	r3, [r2]
   0x000083a8 :	ldr	r1, [pc, #16]	; 0x83c0 
   0x000083ac :	ldr	r1, [r1]
   0x000083b0 :	ldr	r2, [pc, #12]	; 0x83c4 
   0x000083b4 :	ldr	r2, [r2]
   0x000083b8 :	add	r0, r1, r2
   0x000083bc :	bx	lr
End of assembler dump.
```

查看寄存器（缩写命令为 `i r`）：

```
(gdb) info registers r0 r1 r2 r3
r0             0x1	1
r1             0xbefff744	3204446020
r2             0xbefff74c	3204446028
r3             0x8390	33680
```

修改寄存器的值：

> p 表示 print

```
(gdb) p $r0 = 2
$1 = 2                   $1表示执行 p 时求值求出的第一个结果
(gdb) info registers r0 r1 r2 r3
r0             0x2	2
r1             0xbefff744	3204446020
r2             0xbefff74c	3204446028
r3             0x8390	33680
```

引用之前输入的值：

```
(gdb) p $1
$2 = 2
```

执行下一条指令（缩写命令为 `si`）：

> stepi 表示 step instruction

```
(gdb) stepi
0x00008394 in main ()
```

打印变量/变量地址：

```
(gdb) p &myvar1
$3 = ( *) 0x10564
```

```
(gdb) p myvar1
$4 = 0
```

执行到下一个标号（缩写命令为 `c`）：

```
(gdb) continue
Continuing.
[Inferior 1 (process 3080) exited with code 07]
```

## 分支 (Branching)

`r15` 在绝大部分情况下充当 `pc` 的作用，寄存器名称 `pc` 就是 `r15` 的别名。

ARMv7 指令集定长，32bit.

改变 `pc` 的值的过程称为分支 (Branching). 中文一般也叫跳转，虽然跳转对应的英文应该是 jump.（事实上，分支和跳转背后的设计思想有别）

### 无条件跳转

单独使用 `b <label>` 指令来进行无条件跳转。

```asm
/* -- branch01.s */
.text
.global main
main:
    mov r0, #2 /* r0 ← 2 */
    b end      /* 跳转到标号为 'end' 的位置 */
    mov r0, #3 /* r0 ← 3 */
end:
    bx lr
```

### 有条件跳转

`cpsr`(Current Program Status Register) 寄存器是一个特殊寄存器，保存了程序执行过程中一些条件信息。其保存的四个 flag 分别是：

- `N`: Negative flag: 上一条指令的结果是否为负数
- `Z`: Zero flag: 上一条指令的结果是否为 0
- `C`: Carry flag: 上一条指令的结果是否需要第 33 个位才能表示 -- 加法溢出或者非借位减法都会使其为真
- `V`: Overflow flag: 上一条指令的结果是否出现了溢出

我们可以使用 `cmp A B` 指令来 "做比较" -- 它相当于执行 `sub ? A B`(A - B), 运算中更新 cpsr，但运算结果并不会保存在任何地方。所以我们有：

- `N`: A < B
- `Z`: A == B
- `C`: A > B    (发生了非借位减法)
- `V`: 溢出

我们可以透过 cpsr 的状态来判断常见的关系：

| 缩写  | 关系                     | cpsr 状态          |
|-------|--------------------------|--------------------|
| EQ    | equal                    | `Z == 1`           |
| NE    | not equal                | `Z == 0`           |
| GE    | greater or equal than    | `N == V`           |
| LT    | lower than               | `N != V`           |
| GT    | greather than            | `N == V && Z == 0` |
| LE    | lower or equal than      | `N != V || Z == 1` |
| MI    | minus/negative           | `N == 1`           |
| PL    | plus/positive or zero    | `N == 0`           |
| VS    | overflow set             | `V == 1`           |
| VC    | overflow clear           | `V == 0`           |
| HI    | higher                   | `C == 1 && Z == 0` |
| LS    | lower or same            | `C == 0 || Z == 1` |
| CS/HS | carry set/higher or same | `C == 1`           |
| CC/LO | carry clear/lower        | `C == 0`           |

上面的这些缩写可以拿来跟 `b` 指令组合变成 `bXX` 指令，例如下面这个程序：

```asm
/* -- compare01.s */
.text
.global main
main:
    mov r1, #2       /* r1 ← 2 */
    mov r2, #2       /* r2 ← 2 */
    cmp r1, r2       /* 用 "r1 - r2" 的伪操作来更新 cpsr 寄存器 */
    beq case_equal   /* cpsr 状态为 EQ，即 Z == 1 时，就跳转 */
case_different :
    mov r0, #2       /* r0 ← 2 */
    b end            /* branch to end */
case_equal:
    mov r0, #1       /* r0 ← 1 */
end:
    bx lr
```

## 实现控制流

> 这里描述的实现不是最优的，只是能行的。
>
> 更优的实现 (具有更少跳转语句) 参见龙书

### if/then/else

```asm
if_eval: 
    /* 求值条件，并根据条件生成 cmp */
bXX else /* 合适的 bXX 用于跳转 */
then_part: 
   /* 生成 then 块 */
   b end_of_if
else:
   /* 生成 else 块 */
end_of_if:
```

具体实现与短路的实现可以看龙书。

### while

```asm
while_condition: 
    /* 生成条件 E */
    bXX end_of_loop  /* 如果 E 是假的，跳转 */
    /* 生成循环体 */
    b while_condition /* 无条件跳回开始 */
end_of_loop:
```

### for

#### 不带 continue 的

相当于：

```c
for (A; B; C)
    S

A;
while (B) {
    S;
    C;
}
```

#### 带 continue 的

```c
A;
while (B) {
    S; // continue 相当于 goto step

step:
    C;
}
```

## 更紧凑的指令 / 取址模式

### Indexing modes / 指令参数类型

通过之前的例子，我们可以发现一条 ARM 汇编指令的参数可以是不同的类型 (立即数/寄存器/`[寄存器]`/标号...). 这些类型被称为 indexing mode (这个名字没啥意义，反正你认为是参数类型就是了).

指令的语法可以总结如下：

```asm
instruction Rdest, Rsource1, source2
```

其中 `Rdest` 与 `Rsource1` 都必须是一个寄存器，而 `source2` 则可以塞入更多种类的操作数。本章主要讨论 `source2` 的其他种类。

> 有些指令是不一样的，但是总的来说就是这么个理，比如 `mov Rdest, source2`

至于 ldr 跟 str 的特殊取址模式，下章再讲。

### 位移运算数

- `LSL #n/Rsource3`: 逻辑左移，左 n 位会丢，右 n 位是 0. n 由立即数或者寄存器 Rsource3 里的值决定
- `LSR #n/Rsource3`: 逻辑右移，左 n 位是 0，右 n 位会丢
- `ASR #n/Rsource3`: 算术右移，左 n 位填充为符号位，右 n 位会丢
- `ROR #n/Rsource3`: 向右旋转，类似 `LSR`, 但右 n 位被推到左 n 位去

举几个例子：

```asm
mov r1, r2, LSL #1      @ r1 <- (r2*2)
mov r1, r2, LSL #2      @ r1 <- (r2*4)
mov r1, r3, ASR #3      @ r1 <- (r3/8)
mov r3, #4
mov r1, r2, LSL r3      @ r1 <- (r2*16)
```

更复杂的乘法：

```asm
add r1, r2, r2, LSL #1      @ r1 <- r2 + (r2*2) equivalent to r1 <- r1*3
add r1, r2, r2, LSL #2      @ r1 <- r2 + (r2*4) equivalent to r1 <- r1*5
sub r1, r2, r2, LSL #3      /* r1 <- r2 - (r2*8) equivalent to r1 <- r2*(-7) */
rsb r1, r2, r2, LSL #3      /* r1 <- (r2*8) - r2 equivalent to r1 <- r2*7 */
```

注意 rsb (Reverse SuBstract) 指令，它就是被减数跟减数顺序反过来的 sub 指令。因为位移操作数只能在最后一个参数 (`source2`) 用，所以 ARM 提供了这个指令来让我们不用用正常的 sub 减完再取负. (适合于我们想要对被减数使用位移的时候)

这个主要用来做强度约减的。

## 数组与结构体

### 定义/分配空间

C 语言里的数组/结构体定义可以翻译如下：

```c
int a[100];
struct my_struct {
    char f0;
    int f1;
} b;
char s[] = "This is a string";
```

```asm
/* -- array01.s */
.data
a:
    .skip 400       @ 预留 400 byte 的内存空间，即 sizeof(a)
b:
    .skip 8         @ 预留 8 byte 的内存空间，即 sizeof(b), 要考虑对齐

S:
    .asciz "This is a string"   @ ASCIi with Zero terminator，会自动在最后附加空字符
```

### Indexing mode / 索引模式 / 取址模式

#### 不更新寄存器 (Non-updating indexing modes)

1. `[R1, #±n]`: 取地址 R1±n
2. `[R1, #±R2]`: 取地址 R1±R2
3. `[R1, ±R2, shift_op #±n]`: shift_op 可以是上面提到过的任意位移操作，取地址 `R1 ± shift_op(R2, ±n)`
  
其中 (1) 适合类似 `a[N]`, `N` 为常数的索引，也适合于结构体里取成员. (3) 适合于 `a[i]`, `i` 是变量的索引。假如 `a` 是 i32 类型的数组，那么只要 `[a, i, LSL #4]` 就可以了。

例子：

```asm
str r2, [r1, #+12]          @ *(r1 + 12) <- r2
str r2, [r1, +r3]           @ *(r1 + r3) <- r2
str r2, [r1, +r2, LSL #2]   @ *(r1 + r2*4) <- r2
```

#### 取址后更新寄存器 (Post-indexing modes)

考虑下面的 C 代码：

```c
for (int i = 0; i < n; i++) {
    a[i] = i;
}
```

如果正常地按照字面意思来翻译的话，我们会翻译成这样：

```asm
/* 假设 r0 存了 a 的首地址，r1 存了 n 的值 */
/* 用 r2 来当 i */

mov r2, #0
loop:
    cmp r2, r1         
    bge loop_end

    str r2 [r0, +r2, LSL #2]   @ r2 -> *(r0 + r2*4)

    add r2, r2, #1             @ r2 <- r2 + 1
    b loop
loop_end:
```

但是仔细想想，我们并不需要每一次访问元素都算一次地址，我们可能可以这样实现：

```asm
/* 假设 r0 一开始存了 a 的首地址，r1 存了 n 的值 */

mov r2, #0
loop:
    cmp r2, r1     
    bge loop_end

    str r2 [r0]         @ (*) r2 -> *r0
    add r0, r0, #4      @ (*) r0 <- r0 + 4

    add r2, r2, #1      @ r2 <- r2 + 1
    b loop
loop_end:
```

而 ARM 为这种情况提供了一个快速的取址模式: "取址，然后更新". (*) 两行可被合并为 `str r2 [r0], #+4`

1. `[R1], #±n`: 取地址 R1，然后 R1 ±= n
2. `[R1], #±R2`: 取地址 R1，然后 R1 ±= R2
3. `[R1], ±R2, shift_op #±n`: 取地址 R1，然后 R1 ±= shift_op(R2, ±n)

#### 取址前更新寄存器 (Pre-indexing modes)

同样，也有先更新寄存器，再取址的取值模式：

1. `[R1, #±n]!`: R1 ±= n，然后取地址 R1
2. `[R1, #±R2]!`: R1 ±= R2，然后取地址 R1
3. `[R1, ±R2, shift_op #±n]!`: R1 ±= shift_op(R2, ±n), 然后取地址 R1

> 好怪啊，为什么不把 pre 和 post 的语法反过来

### 访问

要想访问数组的第 i 个元素，我们就直接像上面那样使用 `[a, #i LSL #2]`(`*(a+4*i)`) 就可以了。

要想访问结构体的域，我们也像上面那样，不过这次是计算出偏移量然后加上：`[p, #offset]` (`*(p + offset)`)

## 函数

### 基本知识

一段代码要想成为一个 "函数", 必须遵守一些约定。这些约定用来约束函数执行前后的 CPU 状态。在 Linux 上，一个 ARM 汇编函数必须遵守 ARM 结构过程调用规范 (ARM Architecture Procedure Call Standard, AAPCS).

寄存器 `r14` 又叫 `lr` (link register), 保存了调用本函数的指令的下一条指令的地址 (可以看作是执行完要返回哪里)

寄存器 `r13` 又叫 `sp` (stack pointer), 保存了栈顶的地址。

### 参数传递

首四个参数必须被按顺序存于 `r0`, `r1`, `r2`, `r3`.

更多的参数置于栈中 TODO

### "好"函数

- 函数不应该依赖开始执行时 cpsr 的状态
- 函数可以自由修改 `r0`, `r1`, `r2`, `r3`
- 除非被用于传递参数，否则函数不应该依赖于开始执行时 `r0`, `r1`, `r2`, `r3` 内的值
- 函数内部可以随意修改 `lr`, 但离开函数之后 `sp` 的值总应该是进入函数时 `lr` 的值
- 函数内部可以随意修改其他寄存器，但离开函数时它们的值应该与进入时相同

### 调用函数

直接 (立即) 调用：`bl <标号>`, 此标号必须在 `.text` 节定义。

间接调用：`blx Rsource1`, 其中 Rsource1 存着该函数的第一条指令的地址

### 离开函数

直接跳转即可。`bx Rsource1`, 其中 Rsource1 应该保存着进入函数时 `lr` 的值。

> blx 指令在跳转的同时还会将 lr 设为 pc + #4，而函数返回时我们只需要做一个简单跳转就好了，所以直接用 bx

### 返回值

对于 32bit 的基础类型，比如 C 的 char, short, int, long，返回值存于 `r0`; 对于 64 位的基础类型，返回值存于 `r0` 与 `r1`; 对于其他任何长度大于 32 bit 的类型，返回值存于栈上。

### 实例: Hello World (调用 C 标准库函数)

```asm
/* -- hello01.s */
.data

greeting:
 .asciz "Hello world"   /* .asciz: 以 0 结尾的字符数组 */

.balign 4
return: .word 0         /* 用于保存 lr 的值，因为函数调用可能改变 lr        */
                        /* 被调函数只负责能跳回来，**不负责**还原 lr 的值！!! */

.text

.global main
main:
    ldr r1, address_of_return     /*   r1 ← &address_of_return */
    str lr, [r1]                  /*   *r1 ← lr */

    ldr r0, address_of_greeting   /* r0 ← &address_of_greeting */
                                  /* 传给 puts 的第一个参数 */

    bl puts                       /* 调用 puts */
                                  /* lr ← address of next instruction */

    ldr r1, address_of_return     /* r1 ← &address_of_return */
    ldr lr, [r1]                  /* lr ← *r1 */
    bx lr                         /* return from main */
address_of_greeting: .word greeting
address_of_return: .word return

/* 外部函数 External */
.global puts
```

## 栈

### 基本理论

- `sp` 表示当前栈顶。根据 AAPCS，其应以 8 字节对齐。
- 离开函数时，`sp` 必须被恢复为进入函数时的值

在 Linux ARM 上，栈向低地址方向增长。

入栈/出栈理论上的操作如下：

```asm
sub sp, sp, #8  /* sp ← sp - 8. 扩大当前栈帧 8 byte */
str lr, [sp]    /* *sp ← lr */
... // 函数的其他代码 ...
ldr lr, [sp]    /* lr ← *sp */
add sp, sp, #8  /* sp ← sp + 8. /* 减小当前栈帧 8 byte
                                   将 sp 还原 */
bx lr
```

使用索引模式，代码简化如下：

```asm
str lr, [sp, #-8]!  /* preindex: sp ← sp - 8; *sp ← lr */
... // Code of the function
ldr lr, [sp], #+8   /* postindex; lr ← *sp; sp ← sp + 8 */
bx lr
```

### 阶乘程序

下面的程序实现了阶乘：

```asm
/* -- factorial01.s */
.data

message1: .asciz "Type a number: "
format:   .asciz "%d"
message2: .asciz "The factorial of %d is %d\n"

.text

factorial:
    str lr, [sp,#-4]!  /* Push lr onto the top of the stack */
    str r0, [sp,#-4]!  /* Push r0 onto the top of the stack */
                       /* Note that after that, sp is 8 byte aligned */
    cmp r0, #0         /* compare r0 and 0 */
    bne is_nonzero     /* if r0 != 0 then branch */
    mov r0, #1         /* r0 ← 1. This is the return */
    b end
is_nonzero:
                       /* Prepare the call to factorial(n-1) */
    sub r0, r0, #1     /* r0 ← r0 - 1 */
    bl factorial
                       /* After the call r0 contains factorial(n-1) */
                       /* Load r0 (that we kept in th stack) into r1 */
    ldr r1, [sp]       /* r1 ← *sp */
    mul r0, r0, r1     /* r0 ← r0 * r1 */
    
end:
    add sp, sp, #+4    /* Discard the r0 we kept in the stack */
    ldr lr, [sp], #+4  /* Pop the top of the stack and put it in lr */
    bx lr              /* Leave factorial */

.global main
main:
    str lr, [sp,#-4]!            /* Push lr onto the top of the stack */
    sub sp, sp, #4               /* Make room for one 4 byte integer in the stack */
                                 /* In these 4 bytes we will keep the number */
                                 /* entered by the user */
                                 /* Note that after that the stack is 8-byte aligned */
    ldr r0, address_of_message1  /* Set &message1 as the first parameter of printf */
    bl printf                    /* Call printf */

    ldr r0, address_of_format    /* Set &format as the first parameter of scanf */
    mov r1, sp                   /* Set the top of the stack as the second parameter */
                                 /* of scanf */
    bl scanf                     /* Call scanf */

    ldr r0, [sp]                 /* Load the integer read by scanf into r0 */
                                 /* So we set it as the first parameter of factorial */
    bl factorial                 /* Call factorial */

    mov r2, r0                   /* Get the result of factorial and move it to r2 */
                                 /* So we set it as the third parameter of printf */
    ldr r1, [sp]                 /* Load the integer read by scanf into r1 */
                                 /* So we set it as the second parameter of printf */
    ldr r0, address_of_message2  /* Set &message2 as the first parameter of printf */
    bl printf                    /* Call printf */


    add sp, sp, #+4              /* Discard the integer read by scanf */
    ldr lr, [sp], #+4            /* Pop the top of the stack and put it in lr */
    bx lr                        /* Leave main */

address_of_message1: .word message1
address_of_message2: .word message2
address_of_format: .word format
```

### 快速保存各种寄存器: ldm & stm

TODO 详细介绍

使用 `stmdb sp!, {r4, lr}` 将 r4, lr 推入栈中; 使用 `ldmia sp!, {r4, lr}` 将 r4, lr 弹出栈 (并且把对应寄存器的值设回去).

GNU as 给我们提供了两个助记符：`push {r4, lr}` 与 `pop {r4, lr}`

## 条件执行

对大部分指令，在它后面加条件后缀 (`eq`, `ne`, ...) 即可：

```asm
mp r2, #0                   @ Compare r2 and 0
moveq r1, r1, ASR #1        @ if r2 == 0, r1 <- r1 >> 1. [r1 <- r1/2]
addne r1, r1, r1, LSL #1    @ if r2 != 0, r1<-r1+(r1<<1). [r1 <- 3*r1]
addne r1, r1, #1            @ if r2 != 0, r1 <- r1 + 1
```

需要注意的是，一般的指令并不会改变 cpsr 的状态，只有 `cmp` 跟加了后缀 `s` 的指令会改变 cpsr.

```asm
/* for (int i = 100 ; i >= 0; i--) */
mov r1, #100
loop:
    /* do something */
    subs r1, r1, #1     @ r1 <- r1 - 1, update cpsr with the final r1
    bpl loop            @ branch if the previous sub computed a positive
                        @ number (N flag in cpsr is 0)
```

## 局部变量

### 快速总结

> 关于栈帧，那本书里教的貌似不是 gcc 的做法，注意对比

标准没有规定 `fp` 是哪个寄存器，但通行做法是用 `r11`, GNU as 也支持使用 `fp` 作为 `r11` 的简写。

标准对栈帧结构没啥要求，只要你能符合 AAPCS 就可以了。反汇编 `arm-linux-gnueabihf-gcc` 的编译结果可以看到 GCC 对栈帧的处理方式，总结一下就是下面这样子 (对应下面动手实践的星号行的时候的机器状态):

```
         |                                      | Low address
         |                                      |
         |                                      |
         |                                      |
         |                                      |<----------+
       + |......................................|           |
       + |                                      |           |
       + |                                      |           |
       + |  Local variable                      |           |
       + |                                      |           |
       + |......................................|           |
       + |                                      |           |
       + |                                      |           |
       + |  Local variable                      |           |
       + |                                      |           |
       + |......................................|           |
       + |                                      |           |
       + |                                      |           |
       + |  Local variable                      |           |
       + |                                      |           |
       + +--------------------------------------+           |
         |                                      |           |
         |                                      |           |
         |  (Last) Other registers(R4)          |           |
         |                                      |           |
         +--------------------------------------+           |
         |                                      |           |
         |                                      |           |
         |  (Last) FP                           |           | +----------------+
+--------+                                      |<---+      | |                |
|        +--------------------------------------+    |      +-+ Stack Pointer  |
|        |                                      |    |        |                |
|        |                                      |    |        +----------------+
|        |  (last) LR                           |    |
|        |                                      |    |        +----------------+
|        +--------------------------------------+    |        |                |
|        |                                      |    +--------+ Frame Pointer  |
|        |                                      |             |                |
|        |                                      |             +----------------+
|        |                                      |
|        |                                      |
|        |                                      |
|
|                                                   High address
+------>
```

一直通过 `fp = *fp` 就可以遍历当前整个栈，看起来就好像用 fp 把栈帧都串起来了一样。

### 动手过程

```c
int fib(int x) {
    if (x == 1 || x == 2) {
        return 1;
    } else {
        return fib(x - 1) + fib(x - 2);
    }
}
```

使用下面的命令反汇编: (我知道 gcc 可以直接出汇编，但貌似 objdump 出来的格式好看点)(输出经过了美化，可能你的原始输出会不太一样)

```sh
arm-linux-gnueabihf-gcc -march=armv7-a -O0 -static -g -c arm_stackframe.c -o arm_stackframe.O0.armv7.o
arm-linux-gnueabihf-objdump -drwCS arm_stackframe.armv7.O0.o > arm_stackframe.armv7.O0.s
```

> 参数解释参见: <https://stackoverflow.com/a/1289907>

```asm

arm_stackframe.armv7.O0.o:     file format elf32-littlearm


Disassembly of section .text:

00000000 <fib>:
@int fib(int x) {
    /* ==================== 进入函数的处理 ====================== */
   0:	e92d4810 	push	{r4, fp, lr}            @ 存 r4，因为后面会用
   4:	e28db008 	add	fp, sp, #8                  @ 让 fp 指向栈里存着的 fp
   8:	e24dd00c 	sub	sp, sp, #12                 @ 栈空间保留 3 * i32
    /* *************************************** 上面的示意图就是现在的状态 *************************************** */   

   c:	e50b0010 	str	r0, [fp, #-16]              @ 根据 AAPCS, r0 里是第一个参数，这里把输入参数放到栈上 (可能是因为开了 O0，默认假设参数都在栈上方便 codegen)
@    if (x == 1 || x == 2) {
  10:	e51b3010 	ldr	r3, [fp, #-16]              @ 读取输入参数
  14:	e3530001 	cmp	r3, #1                      @ 短路比较
  18:	0a000002 	beq	28 <fib+0x28>
  1c:	e51b3010 	ldr	r3, [fp, #-16]
  20:	e3530002 	cmp	r3, #2
  24:	1a000001 	bne	30 <fib+0x30>
@        return 1;
  28:	e3a03001 	mov	r3, #1                     @ 返回值存 r3 里，跳转到返回
  2c:	ea00000a 	b	5c <fib+0x5c>
@    } else {
@        return fib(x - 1) + fib(x - 2);
  30:	e51b3010 	ldr	r3, [fp, #-16]             @ r3 读取第一个，输入参数
  34:	e2433001 	sub	r3, r3, #1                 @ 减一
  38:	e1a00003 	mov	r0, r3                     @ 设好参数
  3c:	ebfffffe 	bl	0 <fib>	3c: R_ARM_CALL	fib @ 调函数
  40:	e1a04000 	mov	r4, r0                     @ r4 存返回值

  44:	e51b3010 	ldr	r3, [fp, #-16]
  48:	e2433002 	sub	r3, r3, #2
  4c:	e1a00003 	mov	r0, r3
  50:	ebfffffe 	bl	0 <fib>	50: R_ARM_CALL	fib
  54:	e1a03000 	mov	r3, r0                     @ r3 存返回值
  
  58:	e0843003 	add	r3, r4, r3                 @ r3 = r3 + r4
@    }

    /* ==================== 返回的处理 ====================== */
  5c:	e1a00003 	mov	r0, r3
  60:	e24bd008 	sub	sp, fp, #8                 @ 局部变量退栈
  64:	e8bd8810 	pop	{r4, fp, pc}               @ 直接把存着 lr 的内存格子给 pc (相当于 bx lr)

```

顺带一提 `O1` 优化的版本如下：

```asm

arm_stackframe.armv7.O1.o:     file format elf32-littlearm


Disassembly of section .text:

00000000 <fib>:
int fib(int x) {
    if (x == 1 || x == 2) {                    @ 对基本情况直接原地返回，不操作栈
   0:	e2403001 	sub	r3, r0, #1
   4:	e3530001 	cmp	r3, #1
   8:	8a000001 	bhi	14 <fib+0x14>
        return 1;
   c:	e3a00001 	mov	r0, #1
    } else {
        return fib(x - 1) + fib(x - 2);
    }
  10:	e12fff1e 	bx	lr
int fib(int x) {
  14:	e92d4070 	push	{r4, r5, r6, lr}
  18:	e1a04000 	mov	r4, r0                        @ 这下没干存栈里再读回来的蠢事了
        return fib(x - 1) + fib(x - 2);
  1c:	e1a00003 	mov	r0, r3
  20:	ebfffffe 	bl	0 <fib>	20: R_ARM_CALL	fib
  24:	e1a05000 	mov	r5, r0
  28:	e2440002 	sub	r0, r4, #2
  2c:	ebfffffe 	bl	0 <fib>	2c: R_ARM_CALL	fib
  30:	e0850000 	add	r0, r5, r0
  34:	e8bd8070 	pop	{r4, r5, r6, pc}

```
