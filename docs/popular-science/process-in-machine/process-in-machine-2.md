---
title: RISC-V 的常见指令
authors:
    - CH3CHOHCH3
date: 2022-04-09 15:00:00
---
# RISC-V 的常见指令

!!! info ""
    这是“程序的机器级表示”系列的第二篇文章。本篇介绍 RISC-V 的常用指令，帮助建立汇编编程的初步印象。


我们已经知道，处理器执行的大部分指令都是在存取、运算、比较寄存器和主存中的数据，或是确定下一条指令的位置（PC 的值）以实现循环、分支等功能。下面，我们来介绍 RISC-V 指令集中的各种指令，了解如何用 RISC-V 编写简单的程序。

!!! info ""
    历史上，指令集的设计有 RISC（Reduced Instruction Set Computer）和 CISC（Complex Instruction Set Computer）两种理念，从名字中可以看出，RISC 强调指令集的精简，CISC 则强调指令集的复杂。CISC 中的指令种类要多得多，单条指令的功能也往往比 RISC 更复杂，同样的指令在 RISC 中可能需要多条指令实现。因此 CISC 完成任务需要的指令数更少，但代码的编写、CPU 的设计也更为复杂。Intel 的 x86 是 CISC 的代表，而移动端和嵌入式设备中常见的 arm（Advanced RISC Machine）则是 RISC 的代表。


!!! info ""
    RISC-V 读作“risk-five”，是加州大学伯克利分校设计的一种开源 ISA（x86 是闭源的），其目标是称为 ISA 领域的 Linux。该项目发起于 2010 年，为解决常见指令集的诸多问题：闭源，扼制创新；过于复杂，不利于学术研究，且很多复杂性源于历史或设计问题；针对性强，如 arm 面向移动端，x86 面向服务器，缺少统一性架构；商业指令集易受企业发展的影响。


# 运算指令

RISC-V 中的运算指令包括算术运算，逻辑运算，移位运算。运算只在寄存器之间运行，即想要对内存中的数据进行运算，需要先将其取至寄存器。

这里介绍立即数的概念。汇编中立即数即为常数，一般在运算时会对其作 **符号扩展** ，关于符号扩展和零扩展会在介绍 RISC-V 的机器码表示时介绍。

值得注意的是，RISC-V 中有一个寄存器 x0 被硬编码为 0，其值无法修改，作为常数存在。

## 算术运算

+ `add rd,rs1,rs2`：将寄存器 rs1 与 rs2 的值相加并写入寄存器 rd。
+ `sub rd,rs1,rs2`：将寄存器 rs1 与 rs2 的值相减并写入寄存器 rd。
+ `addi rd,rs1,imm`：将寄存器 rs1 的值与立即数 imm 相加并存入寄存器 rd。
+ `mul rd,rs1,rs2`：将寄存器 rs1 与 rs2 的值相乘并写入寄存器 rd。
+ `div rd,rs1,rs2`：将寄存器 rs1 除以寄存器 rs2 的值，向零舍入并写入寄存器 rd。
+ `rem rd,rs1,rs2`：将寄存器 rs1 模寄存器 rs2 的值并写入寄存器 rd。

以上运算发生溢出时会自动截断高位。乘法可以用`mulh`，`mulhu`获得两个 32 位数乘积的高 32 位，细节不赘述。

## 逻辑运算

+ `and rd,rs1,rs2`：将寄存器 rs1 与 rs2 的值按位与并写入寄存器 rd。
+ `andi rd,rs1,imm`：将寄存器 rs1 的值与立即数 imm 的值按位与并写入寄存器 rd。
+ `or rd,rs1,rs2`：将寄存器 rs1 与 rs2 的值按位或并写入寄存器 rd。
+ `ori rd,rs1,imm`：将寄存器 rs1 的值与立即数 imm 的值按位或并写入寄存器 rd。
+ `xor rd,rs1,rs2`：将寄存器 rs1 与 rs2 的值按位异或并写入寄存器 rd。
+ `xori rd,rs1,imm`：将寄存器 rs1 的值与立即数 imm 的值按位异或并写入寄存器 rd。

## 移位运算

+ `sll rd,rs1,rs2`：将寄存器 rs1 的值左移寄存器 rs2 的值这么多位，并写入寄存器 rd。
+ `slli rd,rs1,imm`：将寄存器 rs1 的值左移立即数 imm 的值这么多位，并写入寄存器 rd。
+ `srl rd,rs1,rs2`：将寄存器 rs1 的值逻辑右移寄存器 rs2 的值这么多位，并写入寄存器 rd。
+ `srli rd,rs1,imm`：将寄存器 rs1 的值逻辑右移立即数 imm 的值这么多位，并写入寄存器 rd。
+ `sra rd,rs1,rs2`：将寄存器 rs1 的值算数右移寄存器 rs2 的值这么多位，并写入寄存器 rd。
+ `srai rd,rs1,imm`：将寄存器 rs1 的值算数右移立即数 imm 的值这么多位，并写入寄存器 rd。

左移会在右边补 0，逻辑右移会在最高位添 0，算数右移在最高位添加符号位。

区分算数右移和逻辑右移，是从计算的角度考虑的：左移一位等于乘 2，右移一位等于除 2 是算数的规律；无论正数负数，在右边补 0 都等于乘 2；而负数进行逻辑右移的结果不等于除以 2，需要用算数右移；而若只有算术右移，则无符号数的运算又会受影响。

# 数据传输指令

前面讲到，想要对主存中的数据进行运算，需要先将其取至寄存器，数据传输指令实现了这个目的。

现代计算机以 **字节** (byte，1 byte=8 bit) 为基本单位，而内存本身可被视作由 byte 组成的一维数组，地址从 0 开始。**字** (word) 则是存取数据的另一个单位，在 RISC-V 中 1 word=4 byte=32 bit，在其他体系结构中可能会发生变化。

+ `lb rd,offset(rs1)`：从地址为寄存器 rs1 的值加 offset 的主存中读一个字节，符号扩展后存入 rd
+ `lh rd,offset(rs1)`：从地址为寄存器 rs1 的值加 offset 的主存中读半个字，符号扩展后存入 rd
+ `lw rd,offset(rs1)`：从地址为寄存器 rs1 的值加 offset 的主存中读一个字，符号扩展后存入 rd
+ `lbu rd,offset(rs1)`：从地址为寄存器 rs1 的值加 offset 的主存中读一个无符号的字节，零扩展后存入 rd
+ `lhu rd,offset(rs1)`：从地址为寄存器 rs1 的值加 offset 的主存中读半个无符号的字，零扩展后存入 rd
+ `lwu rd,offset(rs1)`：从地址为寄存器 rs1 的值加 offset 的主存中读一个无符号的字，零扩展后存入 rd
+ `sb rs1,offset(rs2)`：把寄存器 rs1 的值存入地址为寄存器 rs2 的值加 offset 的主存中，保留最右端的 8 位
+ `sh rs1,offset(rs2)`：把寄存器 rs1 的值存入地址为寄存器 rs2 的值加 offset 的主存中，保留最右端的 16 位
+ `sw rs1,offset(rs2)`：把寄存器 rs1 的值存入地址为寄存器 rs2 的值加 offset 的主存中，保留最右端的 32 位

!!! info ""
    l 是 load 的首字母，即加载数据；s 是 store 的缩写，即存储数据。b，h，w 分别是 byte，half word，word 的首字母，除此之外还有存取双字的 d，即 double word。


举例：

```C
long long A[100];
A[10] = A[3] + a;
```

假设数组 A 首地址在寄存器 x3，a 在 x2：

```
ld x10,24(x3)       # long long 占 64 bit =8 byte，A[3] 的地址为 A[0]+3*8
add x10,x2,x10
sd x10,80(x3)
```

# 比较指令

有符号数：

+ `slt rd,rs1,rs2`：若 rs1 的值小于 rs2 的值，rd 置为 1，否则置为 0
+ `slti rd,rs1,imm`：若 rs1 的值小于立即数 imm，rd 置为 1，否则置为 0

无符号数：

+ `sltu rd,rs1,rs2`：若 rs1 的值小于 rs2 的值，rd 置为 1，否则置为 0
+ `sltiu rd,rs1,imm`：若 rs1 的值小于立即数 imm，rd 置为 1，否则置为 0

# 条件分支指令

这部分用来实现控制流，即 if 语句，循环等。汇编中没有 C 等高级语言中的`{}`语句块，而是用`Label:`的形式，下面会举例说明。

+ `beq rs1,rs2,label`：若 rs1 的值等于 rs2 的值，程序跳转到 label 处继续执行
+ `bne rs1,rs2,label`：若 rs1 的值不等于 rs2 的值，程序跳转到 label 处继续执行
+ `blt rs1,rs2,label`：若 rs1 的值小于 rs2 的值，程序跳转到 label 处继续执行
+ `bge rs1,rs2,label`：若 rs1 的值大于等于 rs2 的值，程序跳转到 label 处继续执行

`blt`和`bge`也有无符号版本`bltu`，`bgeu`。举例：

```C
int i = 0;
do{
    i++;
}while(i<10)
```

```
add x2,x0,10        # x2 = 10
add x3,x0,0         # i = 0 存储在 x3
Loop:
    add x3,x3,1     # i++
    blt x3,x2,Loop  # i<10 则继续循环
```

# 无条件跳转指令

+ `j label`：程序直接跳转到 label 处继续执行
+ `jal rd,label`：用于调用函数，把下一条指令的地址保存在 rd 中（通常用 x1），然后跳转到 label 处继续执行
+ `jalr rd,offset(rs)`：可用于函数返回，把下一条指令的地址存到 rd 中，然后跳转到 rs+offset 地址处的指令继续执行。若 rd=x0 就是单纯的跳转（x0 不能被修改）

这里详细解释一下`jal`和`jalr`。在调用函数时，我们希望函数返回后，继续执行下一条指令，所以要把这下一条指令的地址存起来，再跳转到函数的代码块。函数执行完之后，根据先前存起来的指令地址，再跳回到调用处继续执行。
