# 第 7 章：可变量

## 7.1 前言

欢迎来到 [我的第一个基于 LLVM 的语言前端](ch-0.md) 教程第七章。在前六章我们构建了一门小而强的[函数式编程语言](http://en.wikipedia.org/wiki/Functional_programming), 从中我们学到了如何去解析程序文本，构建与表示 AST，生成 LLVM IR 以及优化冗余代码和实现即时编译。

尽管 Kaleidoscope 作为一门函数式语言已经足够优秀，但事实上其函数式特性使得为其生成 LLVM IR 显得 "过于简单" 了。特别是，函数式的风格令我们特别容易直接构建出 SSA 形式的 LLVM IR -- 而 SSA 经常将那些想用 LLVM 实现一门包含变量的命令式语言的初学者挡在门外。

幸运的是，你并不需要直接在前端生成 SSA form: 尽管偶尔结果出人意料，但 LLVM 提供了非常健壮的 SSA 构建支持。

## 7.2 那么问题是什么？

为了理解为什么变量会阻碍我们生成 SSA form，考虑下面一段 C 语言代码： 

```c
int G, H;
int test(_Bool Condition) {
  int X;
  if (Condition)
    X = G;
  else
    X = H;
  return X;
}
```

在这里，我们有一个变量 `X`, 它的值依赖于程序的执行路径。对于返回指令而言，`X` 的值有两种可能，所以我们需要插入一个 phi 节点来合并这两个值。大体上，我们希望编译器生成这样的 LLVM IR: 

```llvm
@G = weak global i32 0   ; type of @G is i32*
@H = weak global i32 0   ; type of @H is i32*

define i32 @test(i1 %Condition) {
entry:
  br i1 %Condition, label %cond_true, label %cond_false

cond_true:
  %X.0 = load i32, i32* @G
  br label %cond_next

cond_false:
  %X.1 = load i32, i32* @H
  br label %cond_next

cond_next:
  %X.2 = phi i32 [ %X.1, %cond_false ], [ %X.0, %cond_true ]
  ret i32 %X.2
}
```

> 译者注：关于上面的 LLVM 的一些解释
> 
> LLVM IR 作为一种中间表示，其本身很少见地是强类型的。LLVM IR 内的所有 "变量" 依 SSA 定义只能被赋值 ("定义") 一次且仅一次。而如果我们想要实现真实的 "变量", 我们需要一块内存和一个指向该内存的指针，随后通过指针修改/读取内存的内容到变量中去。对于 C 的全局变量而言，其本身就不可能符合 LLVM IR 关于变量的 SSA 约束，于是 C 的全局变量在翻译成 LLVM IR 的 "变量" 的时候只能是作为一个指向它的指针。
> 
> 关于 LLVM IR 指针的快速介绍：
> 
> - 一个指针的类型是 `<type>*`
> - 使用 `alloca <类型>` 来分配空间
> - 使用 `load <想读取的值的类型>, <指针类型> <指针变量>` 指令来读取
> - 使用 `store <想写入的值的类型> <想写入的变量>, <指针类型> <指针变量>` 来写入。
> 
> 在随后对 LLVM IR 的优化过程中，LLVM 负责将对 "内存" 的读取优化为局部变量，同时构造出合适的 SSA Form. 这样做便避免了要求前端生成 SSA Form，极大地减轻了前端的工作量。

在上面的例子里，对全局变量 `G` 和 `H` 的求值需要显式加载 (load), 随后它们分别存活于 (live in) if 语句的 then 与 else 分支之中。为了合并两个分支中 `X` 的不同值，在 `cond_next` 块内的 `X.2` 使用了 phi 节点来依控制流选择值。当控制流经过 `cond_false` 块时，`X.2` 获得 `X.1` 的值; 当控制流经过 `cond_true` 块时，`X.2` 获得 `X.1` 的值。本章的主要目的不在于讲解 SSA Form 的细节，如需更多信息，请参阅此 [参考](http://en.wikipedia.org/wiki/Static_single_assignment_form). 

> 译者注：SSA 术语解释：定义 (definition)，存活 (live) 与杀死 (kill)
> 
> 在 SSA 里，我们将对某变量的一次赋值称为该变量的定义。所谓一个变量的某个值在程序的某个位置存活，是指从那个值的某个赋值出发，经过所有可能的控制流到达该点时，其路径上都没有该变量的其他定义。一个变量的某个值被杀死，如果该变量被赋予了另外一个值。

---

> 译者注：SSA 术语解释：phi "节点" (phi node)
> 
> 一件值得注意的事情是，为什么 phi 指令要叫 "phi 节点"?
> 
> 事实上，LLVM IR 中基本块与其跳转组成了一张图 (称为 CFG，控制流图)，而图中控制流交会的那个节点 (即基本块) 一般都是要插入 phi 的，名词顺用下来就叫 phi 节点了。

本章的主要问题是，"谁在变量的赋值之前放下 phi 节点？" LLVM IR 要求中间代码必须满足 SSA，但构造 SSA Form 需要实现比较难的算法与数据结构，让所有前端都去被折磨一遍显然是得不偿失的。

## 7.3 LLVM 中的内存

LLVM 虽然要求所有的寄存器变量 (就是普通的 LLVM IR 中的变量) 满足 SSA，但并不要求所有的内存对象满足 SSA. 在上面的例子中，我们通过对 `G` 和 `H` 的 Load 指令直接访问了它们 -- `G` 和 `H` 并不需要被 **重命名/版本化**。与其他编译系统不同，LLVM 不强制要求版本化内存中的对象，而是将对内存的数据流分析综合进 LLVM IR 里 -- 它将在[分析类优化过程](https://llvm.org/docs/WritingAnLLVMPass.html)中被按需计算，随后处理。

> 译者注：粗体小句英文原文：they are not **renamed or versioned**. 

---

> 译者注：SSA 术语解释：重命名 (rename)/版本化 (version)
> 
> 在普通的命令式程序中，我们经常对一个变量多次赋值。在 SSA 中，我们需要区别这多次赋值。比如说上面例子中的 `X` 变量，我们就将其重命名为 `X.0`, `X.1`, `X.2` 等来区分其包含不同值的时候。这种重命名有时候又叫版本化 (version，作动词), 指的是类似这种直接在变量名后面加数字的重命名方法。

---

> 译者注：LLVM 中的寄存器变量 (register variable) 与内存对象 (memory object)
> 
> 所谓寄存器变量，就是指正常的那些 `%` 开头的，可以被赋值的变量。在 LLVM IR 中我们拥有无限多的寄存器。寄存器变量要求满足 SSA 约束，可以直接塞进各种指令里当参数，或者是存放指令的结果。
> 
> 所谓内存对象，就是通过指针指向的某个对象。内存对象可以在堆上或者在栈上，需要使用 load/store 指令来访问/写入，不能直接作为指令参数/存放指令结果。
> 
> 正如上文所提到的，所有的全局变量 (以 `@` 开头) 都是指针，指向某个内存对象。

知道了这个，我们就有实现变量的基本思路了：我们把所有变量都放在函数的栈里 (从而使其是一个内存对象), 然后我们就可以 "绕过" SSA 的约束了。为了实现它，我们需要了解 LLVM 是如何处理栈变量 (stack variable) 的。

在 LLVM，所有的内存访问 (包括读取和写入) 都需要显式使用 `load` / `store` 指令。归功于 LLVM 的优秀设计，我们并不需要一个 "取地址" 运算符。注意全局变量 `@G` / `@H` 的实际类型是 `i32*`, 即便我们在声明它们的时候写的是 `i32`. `@G` 声明中的 `i32` 实际上意味着 `@G` 为全局数据区保留了一个大小为 `i32` 的内存空间，而名字 `@G` 实际上只是对那个空间的地址的一个引用。我们的栈变量以相同方式工作，除了我们这次使用 [alloca 指令](https://llvm.org/docs/LangRef.html#alloca-instruction)来声明它们：

```llvm
define i32 @example() {
entry:
  %X = alloca i32           ; type of %X is i32*.
  ...
  %tmp = load i32, i32* %X  ; load the stack value %X from the stack.
  %tmp2 = add i32 %tmp, 1   ; increment it
  store i32 %tmp2, i32* %X  ; store it back
  ...
```

上面的例子展示了如何在 LLVM IR 中声明与维护栈变量。被 alloca 指令分配的栈内存可以被任意使用：你可以将该地址传递给其他函数，或者用它储存另一个变量。对于我们之前的实例代码，我们可以用 alloca 指令重写它以避免使用 phi 节点：

```llvm
@G = weak global i32 0   ; type of @G is i32*
@H = weak global i32 0   ; type of @H is i32*

define i32 @test(i1 %Condition) {
entry:
  %X = alloca i32           ; type of %X is i32*.
  br i1 %Condition, label %cond_true, label %cond_false

cond_true:
  %X.0 = load i32, i32* @G
  store i32 %X.0, i32* %X   ; Update X
  br label %cond_next

cond_false:
  %X.1 = load i32, i32* @H
  store i32 %X.1, i32* %X   ; Update X
  br label %cond_next

cond_next:
  %X.2 = load i32, i32* %X  ; Read X
  ret i32 %X.2
}
```

现在，我们可以按下面的方式处理任意变量而无需插入任何 phi 节点了：

1. 对每一个变量都分配一份栈空间
2. 对变量的每一次读取都使用一次 load 指令
3. 对变量的每一次赋值都使用一次 store 指令
4. 要取变量的地址，只需要直接使用栈对象的地址就可以了 (也就是直接使用存放栈空间指针的那个寄存器变量)

刚解决了不可变性的问题，现在我们又有了新的问题：即使是对于非常简单且常见的操作，我们也引入了过多的对栈空间的访问，这将导致极大的性能问题。幸运的是，LLVM 优化器有一个非常好用的优化过程，"mem2reg", 来帮助我们解决这个问题。这个优化过程可以将对内存空间的访问转化为对寄存器变量的使用，并且在合适的地方插入 phi 节点。如果你对上面的例子使用这个优化过程，你将得到：

```llvm
$ llvm-as < example.ll | opt -mem2reg | llvm-dis
@G = weak global i32 0
@H = weak global i32 0

define i32 @test(i1 %Condition) {
entry:
  br i1 %Condition, label %cond_true, label %cond_false

cond_true:
  %X.0 = load i32, i32* @G
  br label %cond_next

cond_false:
  %X.1 = load i32, i32* @H
  br label %cond_next

cond_next:
  %X.01 = phi i32 [ %X.1, %cond_false ], [ %X.0, %cond_true ]
  ret i32 %X.01
}
```

mem2reg 过程实现了用于构造 SSA 的标准 "迭代式支配边界"(iterated dominance frontier) 算法，同时针对一系列常见的 "前端偷懒" 情况 (比如仅仅是因为懒得放 phi 节点而使用内存) 做了针对性的优化。mem2reg 可谓是在语言中实现变量的标准方式，前端应该总是使用 mem2reg. 值得注意的是，mem2reg 只优化特定情况下的内存使用：

1. 它只优化由 alloca 分配的内存，所以它并不会优化到全局变量或者是在堆上的内存对象。
2. 它只优化在函数的 `entry` 块分配的内存。在起始块的 alloca 指令只会执行一次，这简化了分析。
3. 它只优化那些仅被 load/store 指令使用的内存。如果你将内存的地址传递给别的函数或是将指针偏移，那么 mem2reg 就不会优化这个 alloca. 
4. 它只优化包含[第一类值](https://llvm.org/docs/LangRef.html#first-class-types)(像是指针，标量或者向量) 或是长度为 1 的数组的内存。mem2reg 并不能将对内存中结构体或者是数组的访问优化为对 (多个) 寄存器的访问，但另一个优化过程 "sroa" 可以做到这一点 -- 它甚至还能优化对联合体 (union) 的访问等更多情况。

对大部分的命令式语言而言，在将简单的变量使用转化为内存访问时，上面的情况都很少发生 -- 我们将通过 Kaleidoscope 证明这一点。最后，你可能会觉得这样做太麻烦了，为什么不干脆直接在前端写写算法，构造好 SSA Form 呢？原因有三：

1. mem2reg 有良好的测试，并且已经在 clang 中证明了自己：clang 同样使用 mem2reg 来实现变量，假如 mem2reg 出现了 bug，那么它很快就会被 clang 的大量用户发现，进而被开发者修复。
2. mem2reg 跑得飞快：它能快速地处理一般情况，同时也对大量的特殊情况做了特别优化：它能检测到只在一个基本块内被使用的变量或是只被赋值了一次的变量，同时还使用了启发式方法来避免插入无用的 phi 节点; 而这只是 mem2reg 优化的一小部分。
3. mem2reg 能生成调试信息：在 LLVM 中，我们需要先暴露变量的地址，随后才能为该地址附加[调试信息](https://llvm.org/docs/SourceLevelDebugging.html). 而使用 mem2reg 可以非常自然地与 LLVM 生成调试信息的技术相嵌合。

不出意外的话，mem2reg 能极大地简化前端设计与实现。现在，让我们开始为 Kaleidoscope 加入变量吧！

## 7.4 Kaleidoscope 中的变量

解决了理论问题之后，我们现在来决定 Kaleidoscope 里的变量写法。我们主要加入两个特性：

1. 以 `=` 运算符修改变量
2. 定义新变量

尽管只有第一项特性能体现出变量之"变", 但鉴于 Kaleidoscope 现在只能通过函数形参引入变量，所以我们还得添上第二项特性。事实上定义新变量是非常有用的，就算是不可变的变量也能极大提升程序的可读性。语法上，我们的程序将类似下面这样：

```py
# Define ':' for sequencing: as a low-precedence operator that ignores operands
# and just returns the RHS.
def binary : 1 (x y) y;

# Recursive fib, we could do this before.
def fib(x)
  if (x < 3) then
    1
  else
    fib(x-1)+fib(x-2);

# Iterative fib.
def fibi(x)
  var a = 1, b = 1, c in
  (for i = 3, i < x in
     c = a + b :
     a = b :
     b = c) :
  b;

# Call it.
fibi(10);
```

为了修改变量，我们首先需要为变量生成使用 alloca 的中间代码，随后加入新的运算符，最后再来为 Kaleidoscope 加入变量定义语法。

## 7.5 修改之前的变量实现

Kaleidoscope 的符号表之前是在中间代码生成阶段使用 'NamedValues' map 维护的。该 map 以 LLVM 的 `Value*` 的形式保存每一个名字对应的 double 值。为了使之可变，我们现在改其为存放名字对应的内存地址。实际上这是一种重构，(就本身而言) 其不改变编译器的行为，只是修改了编译器的实现。

目前 Kaleidoscope 只支持两种形式的变量定义：函数形参与 for 循环循环变量。为了看起来整齐，我们会让它们也变得可变。这意味着它们的使用也需要改为对内存的读写。

首先，我们将 `NamedValues` 中的 `Value*` 修改为 `AllocaInst*`. 改完后，C++ 编译器就会开始向我们抱怨哪里需要更新了：

```cpp
static std::map<std::string, AllocaInst*> NamedValues;
```

同时，我们创建一个函数来帮助我们在函数的起始块 (entry block) 插入 alloca 指令。

```cpp
/// CreateEntryBlockAlloca - Create an alloca instruction in the entry block of
/// the function.  This is used for mutable variables etc.
static AllocaInst *CreateEntryBlockAlloca(Function *TheFunction,
                                          const std::string &VarName) {
  IRBuilder<> TmpB(&TheFunction->getEntryBlock(),
                 TheFunction->getEntryBlock().begin());
  return TmpB.CreateAlloca(Type::getDoubleTy(TheContext), 0,
                           VarName.c_str());
}
```

上面的代码首先创建了一个插入位置为起始块第一条指令 (`.begin()`) 的 IRBuilder，随后以 `VarName` 为名字插入 alloca 指令并返回其结果。由于 Kaleidoscope 里所有变量的类型都是 double，所以我们并不需要传入变量的类型。

基于上面的函数，我们首先来更改变量求值的代码。现在变量在栈空间中，所以求值变量就需要一次 load: 

```cpp
Value *VariableExprAST::codegen() {
  // Look this variable up in the function.
  // 根据名字找到函数中的变量 (其为一指针)
  Value *V = NamedValues[Name];
  if (!V)
    return LogErrorV("Unknown variable name");

  // Load the value.
  // 使用 load 指令获取其值
  return Builder.CreateLoad(V, Name.c_str());
}
```

上面的代码相当直观。随后我们需要修改变量定义的实现，在定义时插入 alloca 指令。首先修改 `ForExprAST::codegen()` (完整代码参见[后文]()): 

```cpp
Function *TheFunction = Builder.GetInsertBlock()->getParent();

// Create an alloca for the variable in the entry block.
// 为变量在起始块插入一条 alloca 指令
AllocaInst *Alloca = CreateEntryBlockAlloca(TheFunction, VarName);

// Emit the start code first, without 'variable' in scope.
// 首先在循环变量不在作用域内的情况下生成初始值
Value *StartVal = Start->codegen();
if (!StartVal)
  return nullptr;

// Store the value into the alloca.
// 将该初始值存入分配出的内存空间中
Builder.CreateStore(StartVal, Alloca);
...

// Compute the end condition.
// 求值循环终止条件
Value *EndCond = End->codegen();
if (!EndCond)
  return nullptr;

// Reload, increment, and restore the alloca.  This handles the case where
// the body of the loop mutates the variable.
// 重新读取循环变量的值，加一，然后写入回去。
// 不使用之前循环变量的值是为了防止循环体内修改了循环变量的情况。
Value *CurVar = Builder.CreateLoad(Alloca);
Value *NextVar = Builder.CreateFAdd(CurVar, StepVal, "nextvar");
Builder.CreateStore(NextVar, Alloca);
...
```

代码做的事跟[之前](ch-5.md#535-中间代码生成)是一样的，除了我们现在使用 load/store 而不是手动构造 phi 节点来更新它的值。

为了让函数形参可变，我们也需要为它们生成 alloca 指令：

> 译者注：此处原文多用 argument，但综合上下文意思，其实际应指函数形参 (parameter), 故译文全部使用"形参". 

```cpp
Function *FunctionAST::codegen() {
  ...
  Builder.SetInsertPoint(BB);

  // Record the function arguments in the NamedValues map.
  NamedValues.clear();
  for (auto &Arg : TheFunction->args()) {
    // Create an alloca for this variable.
    // 为每个变量生成一条 alloca 指令
    AllocaInst *Alloca = CreateEntryBlockAlloca(TheFunction, Arg.getName());

    // Store the initial value into the alloca.
    // 将其初始值存入内存中
    Builder.CreateStore(&Arg, Alloca);

    // Add arguments to variable symbol table.
    // 将参数加入符号表中
    NamedValues[Arg.getName()] = Alloca;
  }

  if (Value *RetVal = Body->codegen()) {
    ...
```

对于每一个形参，我们插入一条 alloca 指令，然后将实参值存入刚刚分配出的内存对象中，然后将形参变量 (的地址) 加入符号表; 这个过程将在函数的起始块被插入之后发生。

最后我们还需要在优化过程中加入 mem2reg 以生成优秀的中间代码：

```cpp
// Promote allocas to registers.
TheFPM->add(createPromoteMemoryToRegisterPass());
// Do simple "peephole" optimizations and bit-twiddling optzns.
TheFPM->add(createInstructionCombiningPass());
// Reassociate expressions.
TheFPM->add(createReassociatePass());
...
```

让我们来比较一下被 mem2reg 优化前后的代码吧。对于先前的递归斐波那契函数而言，优化前的代码是这样的：

```llvm
define double @fib(double %x) {
entry:
  %x1 = alloca double
  store double %x, double* %x1
  %x2 = load double, double* %x1
  %cmptmp = fcmp ult double %x2, 3.000000e+00
  %booltmp = uitofp i1 %cmptmp to double
  %ifcond = fcmp one double %booltmp, 0.000000e+00
  br i1 %ifcond, label %then, label %else

then:       ; preds = %entry
  br label %ifcont

else:       ; preds = %entry
  %x3 = load double, double* %x1
  %subtmp = fsub double %x3, 1.000000e+00
  %calltmp = call double @fib(double %subtmp)
  %x4 = load double, double* %x1
  %subtmp5 = fsub double %x4, 2.000000e+00
  %calltmp6 = call double @fib(double %subtmp5)
  %addtmp = fadd double %calltmp, %calltmp6
  br label %ifcont

ifcont:     ; preds = %else, %then
  %iftmp = phi double [ 1.000000e+00, %then ], [ %addtmp, %else ]
  ret double %iftmp
}
```

尽管代码非常简单，只有一个变量 (参数`x`),  但它还是很好地说明我们实现变量的方法。在起始块，我们为 `x` 生成了一条 alloca 指令并放入初始值; 函数内每一个对它的引用都被转换成了对栈空间的访问。作为对比，我们没有改写 if/then/else 表达式的代码生成方式 -- 事实上，手动插入 phi 节点来实现它反而更加简单。

mem2reg 执行完毕后，代码变成了这样：

```llvm
define double @fib(double %x) {
entry:
  %cmptmp = fcmp ult double %x, 3.000000e+00
  %booltmp = uitofp i1 %cmptmp to double
  %ifcond = fcmp one double %booltmp, 0.000000e+00
  br i1 %ifcond, label %then, label %else

then:
  br label %ifcont

else:
  %subtmp = fsub double %x, 1.000000e+00
  %calltmp = call double @fib(double %subtmp)
  %subtmp5 = fsub double %x, 2.000000e+00
  %calltmp6 = call double @fib(double %subtmp5)
  %addtmp = fadd double %calltmp, %calltmp6
  br label %ifcont

ifcont:     ; preds = %else, %then
  %iftmp = phi double [ 1.000000e+00, %then ], [ %addtmp, %else ]
  ret double %iftmp
}
```

对 mem2reg 来说，这不过是一种平凡情况 -- `x` 根本没有被重新定义。这个例子只是为了缓解你在生成了 "低效" 代码之后的紧张感而已，放轻松:)！

执行完剩余的优化后，我们得到：

```llvm
define double @fib(double %x) {
entry:
  %cmptmp = fcmp ult double %x, 3.000000e+00
  %booltmp = uitofp i1 %cmptmp to double
  %ifcond = fcmp ueq double %booltmp, 0.000000e+00
  br i1 %ifcond, label %else, label %ifcont

else:
  %subtmp = fsub double %x, 1.000000e+00
  %calltmp = call double @fib(double %subtmp)
  %subtmp5 = fsub double %x, 2.000000e+00
  %calltmp6 = call double @fib(double %subtmp5)
  %addtmp = fadd double %calltmp, %calltmp6
  ret double %addtmp

ifcont:
  ret double 1.000000e+00
}
```

simplifycfg 优化过程选择将返回指令复制一份插入到 `else` 块中，这将消除一些跳转与 phi 指令。

本小节我们修改了符号表实现，令其储存栈变量 (地址) 而非值。现在是时候来实现赋值运算符了。

## 7.6 崭新的赋值运算符

凭我们现在的代码，加入新的赋值运算符非常简单，可以像解析普通二元运算符一样解析它。尽管如此，我们还是选择将其实现为内置运算符以方便后续的代码生成。第一步是为其设置优先级：

```cpp
int main() {
  // Install standard binary operators.
  // 1 is lowest precedence.
  BinopPrecedence['='] = 2;
  BinopPrecedence['<'] = 10;
  BinopPrecedence['+'] = 20;
  BinopPrecedence['-'] = 20;
```

随后我们便可以直接来为赋值运算符生成中间代码了：

```cpp
Value *BinaryExprAST::codegen() {
  // Special case '=' because we don't want to emit the LHS as an expression.
  // 作为特殊情况处理：我们不想求值左边的 "表达式"
  if (Op == '=') {
    // Assignment requires the LHS to be an identifier.
    // 左表达式必须是一个标识符以充当变量名
    VariableExprAST *LHSE = dynamic_cast<VariableExprAST*>(LHS.get());
    if (!LHSE)
      return LogErrorV("destination of '=' must be a variable");
```

赋值运算符的求值并不像普通的二元运算符，后者先求值左边，再求值右边，然后计算。所以我们将其列为特殊情况优先处理。其另一个不同之处在于左 "表达式" 并不能是任意的表达式 -- `(x+1) = expr` 是不合法的，我们只允许形如 `x = expr` 的表达式。

```cpp
  // Codegen the RHS.
  // 求值右表达式
  Value *Val = RHS->codegen();
  if (!Val)
    return nullptr;

  // Look up the name.
  // 查找名字是否存在
  Value *Variable = NamedValues[LHSE->getName()];
  if (!Variable)
    return LogErrorV("Unknown variable name");

  Builder.CreateStore(Val, Variable);
  return Val;
}
...
```

一旦找到名字对应的变量，中间代码的生成就很简单了：先求值右表达式，然后将其存入内存，最后返回其值。最后返回赋予的值是为了支持形如 `X = (Y = Z)` 的链式赋值。

有了赋值运算符，我们就可以修改循环变量和参数了。比如说，我们可以写出下面的代码：

```py
# Function to print a double.
extern printd(x);

# Define ':' for sequencing: as a low-precedence operator that ignores operands
# and just returns the RHS.
def binary : 1 (x y) y;

def test(x)
  printd(x) :
  x = 4 :
  printd(x);

test(123);
```

上面的代码先输出 "123", 然后输出 "4" -- 我们现在可以修改变量的值了！本章的主要目标业已完成，接下来我们将支持局部变量的定义，这将令 "变量" 这个特性变得更加实用。

## 7.7 自定义局部变量

支持 `var/in` 表达式就像我们之前支持其他表达式一样：先修改 lexer，然后是 parser，接着 AST，最后为其生成代码。我们首先来修改 lexer，代码很平凡，没什么好解释的了：

```cpp
enum Token {
  ...
  // var definition
  tok_var = -13
...
}
...
static int gettok() {
...
    if (IdentifierStr == "in")
      return tok_in;
    if (IdentifierStr == "binary")
      return tok_binary;
    if (IdentifierStr == "unary")
      return tok_unary;
    if (IdentifierStr == "var")
      return tok_var;
    return tok_identifier;
...
```

接下来为其定义新的 AST 节点：

```cpp
/// VarExprAST - Expression class for var/in
class VarExprAST : public ExprAST {
  std::vector<std::pair<std::string, std::unique_ptr<ExprAST>>> VarNames;
  std::unique_ptr<ExprAST> Body;

public:
  VarExprAST(std::vector<std::pair<std::string, std::unique_ptr<ExprAST>>> VarNames,
             std::unique_ptr<ExprAST> Body)
    : VarNames(std::move(VarNames)), Body(std::move(Body)) {}

  Value *codegen() override;
};
```

我们希望能一口气定义一串变量，并且它们分别有可选的初始值，随后在 var/in 表达式的 body 部分便可使用被定义的变量。在代码中，我们使用一个 vector `VarNames` 来储存它们。

AST 写好后，我们就可以来改 parser 了。修改主表达式的解析函数如下：

```cpp
/// primary
///   ::= identifierexpr
///   ::= numberexpr
///   ::= parenexpr
///   ::= ifexpr
///   ::= forexpr
///   ::= varexpr
static std::unique_ptr<ExprAST> ParsePrimary() {
  switch (CurTok) {
  default:
    return LogError("unknown token when expecting an expression");
  case tok_identifier:
    return ParseIdentifierExpr();
  case tok_number:
    return ParseNumberExpr();
  case '(':
    return ParseParenExpr();
  case tok_if:
    return ParseIfExpr();
  case tok_for:
    return ParseForExpr();
  case tok_var:
    return ParseVarExpr();
  }
}
```

随后是解析 VarExpr 的函数 `ParseVarExpr`:

```cpp
/// varexpr ::= 'var' identifier ('=' expression)?
//                    (',' identifier ('=' expression)?)* 'in' expression
static std::unique_ptr<ExprAST> ParseVarExpr() {
  getNextToken();  // eat the var.

  std::vector<std::pair<std::string, std::unique_ptr<ExprAST>>> VarNames;

  // At least one variable name is required.
  // 至少需要一个变量名
  if (CurTok != tok_identifier)
    return LogError("expected identifier after var");
```

首先，我们先解析标识符/表达式对并将其信息存入 `VarNames` 中。

```cpp
while (1) {
  std::string Name = IdentifierStr;
  getNextToken();  // eat identifier.

  // Read the optional initializer.
  // 读取可选的初始值
  std::unique_ptr<ExprAST> Init;
  if (CurTok == '=') {
    getNextToken(); // eat the '='.

    Init = ParseExpression();
    if (!Init) return nullptr;
  }

  VarNames.push_back(std::make_pair(Name, std::move(Init)));

  // End of var list, exit loop.
  // 变量定义列表结束了，退出循环
  if (CurTok != ',') break;
  getNextToken(); // eat the ','.

  if (CurTok != tok_identifier)
    return LogError("expected identifier list after var");
}
```

解析完变量定义之后，我们来解析函数体并构建 AST 节点：

```cpp
  // At this point, we have to have 'in'.
  // 现在这里应该是一个 in
  if (CurTok != tok_in)
    return LogError("expected 'in' keyword after 'var'");
  getNextToken();  // eat 'in'.

  auto Body = ParseExpression();
  if (!Body)
    return nullptr;

  return std::make_unique<VarExprAST>(std::move(VarNames),
                                       std::move(Body));
}
```

解析并用 AST 表示出代码之后，我们来为其生成 LLVM IR. 首先是：

```cpp
Value *VarExprAST::codegen() {
  std::vector<AllocaInst *> OldBindings;

  Function *TheFunction = Builder.GetInsertBlock()->getParent();

  // Register all variables and emit their initializer.
  // 为所有变量注册并求值其初始值
  for (unsigned i = 0, e = VarNames.size(); i != e; ++i) {
    const std::string &VarName = VarNames[i].first;
    ExprAST *Init = VarNames[i].second.get();
```

上面的循环遍历所有定义的变量，将其一个个加入变量表中，同时将先前同名变量的值保存于表 `OldBindings` 中。


```cpp
  // Emit the initializer before adding the variable to scope, this prevents
  // the initializer from referencing the variable itself, and permits stuff
  // like this:
  // 我们在变量加入作用域之前便求值初始值表达式。这样会阻止初始值表达式中对该变量的引用，
  // 但仍允许对已存在的同名外层变量的引用，比如：
  //  var a = 1 in
  //    var a = a in ...   # refers to outer 'a'.
  Value *InitVal;
  if (Init) {
    InitVal = Init->codegen();
    if (!InitVal)
      return nullptr;
  } else { // If not specified, use 0.0.
    InitVal = ConstantFP::get(TheContext, APFloat(0.0));
  }

  AllocaInst *Alloca = CreateEntryBlockAlloca(TheFunction, VarName);
  Builder.CreateStore(InitVal, Alloca);

  // Remember the old variable binding so that we can restore the binding when
  // we unrecurse.
  OldBindings.push_back(NamedValues[VarName]);

  // Remember this binding.
  NamedValues[VarName] = Alloca;
}
```

除了注释里提到的问题之外，我们做的主要就是求值初始值表达式，生成 alloca 指令，然后更新符号表使名字被绑定到对应的栈变量上。完成后，我们就可以求值 body 部分了：

```cpp
// Codegen the body, now that all vars are in scope.
// 现在所有的新变量都在作用域内，可以开始为 body 生成中间代码了
Value *BodyVal = Body->codegen();
if (!BodyVal)
  return nullptr;
```

最后，在返回之前，我们将之前被遮盖的同名外层变量的绑定恢复：

```cpp
  // Pop all our variables from scope.
  // 将新定义的变量全都从作用域中删去
  for (unsigned i = 0, e = VarNames.size(); i != e; ++i)
    NamedValues[VarNames[i].first] = OldBindings[i];

  // Return the body computation.
  // 返回函数体求出的值
  return BodyVal;
}
```

到这里，我们实现了带作用域的局部变量定义，并且变量 (显然地) 都是可变的 :). 终于，我们完成了本章开头设下的目标。我们举例用的迭代式斐波那契计算函数也在新实现下良好地运行; mem2reg 优化过程将我们所有的栈变量都优化为 SSA 寄存器变量; phi 节点被按需插入，而我们的前端实现依旧简洁 -- 没有什么 "迭代式支配边界计算" 给彻底整乱。

## 7.8 完整代码

这里是本章例子的完整代码，具备完全的变量与 var/in 支持。使用下面的命令构建这个例子：

```bash
# Compile
clang++ -g toy.cpp `llvm-config --cxxflags --ldflags --system-libs --libs core orcjit native` -O3 -o toy
# Run
./toy
```

完整代码如下：

```cpp
#include "../include/KaleidoscopeJIT.h"
#include "llvm/ADT/APFloat.h"
#include "llvm/ADT/STLExtras.h"
#include "llvm/IR/BasicBlock.h"
#include "llvm/IR/Constants.h"
#include "llvm/IR/DerivedTypes.h"
#include "llvm/IR/Function.h"
#include "llvm/IR/IRBuilder.h"
#include "llvm/IR/Instructions.h"
#include "llvm/IR/LLVMContext.h"
#include "llvm/IR/LegacyPassManager.h"
#include "llvm/IR/Module.h"
#include "llvm/IR/Type.h"
#include "llvm/IR/Verifier.h"
#include "llvm/Support/TargetSelect.h"
#include "llvm/Target/TargetMachine.h"
#include "llvm/Transforms/InstCombine/InstCombine.h"
#include "llvm/Transforms/Scalar.h"
#include "llvm/Transforms/Scalar/GVN.h"
#include "llvm/Transforms/Utils.h"
#include <algorithm>
#include <cassert>
#include <cctype>
#include <cstdint>
#include <cstdio>
#include <cstdlib>
#include <map>
#include <memory>
#include <string>
#include <utility>
#include <vector>

using namespace llvm;
using namespace llvm::orc;

//===----------------------------------------------------------------------===//
// Lexer
//===----------------------------------------------------------------------===//

// The lexer returns tokens [0-255] if it is an unknown character, otherwise one
// of these for known things.
enum Token {
  tok_eof = -1,

  // commands
  tok_def = -2,
  tok_extern = -3,

  // primary
  tok_identifier = -4,
  tok_number = -5,

  // control
  tok_if = -6,
  tok_then = -7,
  tok_else = -8,
  tok_for = -9,
  tok_in = -10,

  // operators
  tok_binary = -11,
  tok_unary = -12,

  // var definition
  tok_var = -13
};

static std::string IdentifierStr; // Filled in if tok_identifier
static double NumVal;             // Filled in if tok_number

/// gettok - Return the next token from standard input.
static int gettok() {
  static int LastChar = ' ';

  // Skip any whitespace.
  while (isspace(LastChar))
    LastChar = getchar();

  if (isalpha(LastChar)) { // identifier: [a-zA-Z][a-zA-Z0-9]*
    IdentifierStr = LastChar;
    while (isalnum((LastChar = getchar())))
      IdentifierStr += LastChar;

    if (IdentifierStr == "def")
      return tok_def;
    if (IdentifierStr == "extern")
      return tok_extern;
    if (IdentifierStr == "if")
      return tok_if;
    if (IdentifierStr == "then")
      return tok_then;
    if (IdentifierStr == "else")
      return tok_else;
    if (IdentifierStr == "for")
      return tok_for;
    if (IdentifierStr == "in")
      return tok_in;
    if (IdentifierStr == "binary")
      return tok_binary;
    if (IdentifierStr == "unary")
      return tok_unary;
    if (IdentifierStr == "var")
      return tok_var;
    return tok_identifier;
  }

  if (isdigit(LastChar) || LastChar == '.') { // Number: [0-9.]+
    std::string NumStr;
    do {
      NumStr += LastChar;
      LastChar = getchar();
    } while (isdigit(LastChar) || LastChar == '.');

    NumVal = strtod(NumStr.c_str(), nullptr);
    return tok_number;
  }

  if (LastChar == '#') {
    // Comment until end of line.
    do
      LastChar = getchar();
    while (LastChar != EOF && LastChar != '\n' && LastChar != '\r');

    if (LastChar != EOF)
      return gettok();
  }

  // Check for end of file.  Don't eat the EOF.
  if (LastChar == EOF)
    return tok_eof;

  // Otherwise, just return the character as its ascii value.
  int ThisChar = LastChar;
  LastChar = getchar();
  return ThisChar;
}

//===----------------------------------------------------------------------===//
// Abstract Syntax Tree (aka Parse Tree)
//===----------------------------------------------------------------------===//

namespace {

/// ExprAST - Base class for all expression nodes.
class ExprAST {
public:
  virtual ~ExprAST() = default;

  virtual Value *codegen() = 0;
};

/// NumberExprAST - Expression class for numeric literals like "1.0".
class NumberExprAST : public ExprAST {
  double Val;

public:
  NumberExprAST(double Val) : Val(Val) {}

  Value *codegen() override;
};

/// VariableExprAST - Expression class for referencing a variable, like "a".
class VariableExprAST : public ExprAST {
  std::string Name;

public:
  VariableExprAST(const std::string &Name) : Name(Name) {}

  Value *codegen() override;
  const std::string &getName() const { return Name; }
};

/// UnaryExprAST - Expression class for a unary operator.
class UnaryExprAST : public ExprAST {
  char Opcode;
  std::unique_ptr<ExprAST> Operand;

public:
  UnaryExprAST(char Opcode, std::unique_ptr<ExprAST> Operand)
      : Opcode(Opcode), Operand(std::move(Operand)) {}

  Value *codegen() override;
};

/// BinaryExprAST - Expression class for a binary operator.
class BinaryExprAST : public ExprAST {
  char Op;
  std::unique_ptr<ExprAST> LHS, RHS;

public:
  BinaryExprAST(char Op, std::unique_ptr<ExprAST> LHS,
                std::unique_ptr<ExprAST> RHS)
      : Op(Op), LHS(std::move(LHS)), RHS(std::move(RHS)) {}

  Value *codegen() override;
};

/// CallExprAST - Expression class for function calls.
class CallExprAST : public ExprAST {
  std::string Callee;
  std::vector<std::unique_ptr<ExprAST>> Args;

public:
  CallExprAST(const std::string &Callee,
              std::vector<std::unique_ptr<ExprAST>> Args)
      : Callee(Callee), Args(std::move(Args)) {}

  Value *codegen() override;
};

/// IfExprAST - Expression class for if/then/else.
class IfExprAST : public ExprAST {
  std::unique_ptr<ExprAST> Cond, Then, Else;

public:
  IfExprAST(std::unique_ptr<ExprAST> Cond, std::unique_ptr<ExprAST> Then,
            std::unique_ptr<ExprAST> Else)
      : Cond(std::move(Cond)), Then(std::move(Then)), Else(std::move(Else)) {}

  Value *codegen() override;
};

/// ForExprAST - Expression class for for/in.
class ForExprAST : public ExprAST {
  std::string VarName;
  std::unique_ptr<ExprAST> Start, End, Step, Body;

public:
  ForExprAST(const std::string &VarName, std::unique_ptr<ExprAST> Start,
             std::unique_ptr<ExprAST> End, std::unique_ptr<ExprAST> Step,
             std::unique_ptr<ExprAST> Body)
      : VarName(VarName), Start(std::move(Start)), End(std::move(End)),
        Step(std::move(Step)), Body(std::move(Body)) {}

  Value *codegen() override;
};

/// VarExprAST - Expression class for var/in
class VarExprAST : public ExprAST {
  std::vector<std::pair<std::string, std::unique_ptr<ExprAST>>> VarNames;
  std::unique_ptr<ExprAST> Body;

public:
  VarExprAST(
      std::vector<std::pair<std::string, std::unique_ptr<ExprAST>>> VarNames,
      std::unique_ptr<ExprAST> Body)
      : VarNames(std::move(VarNames)), Body(std::move(Body)) {}

  Value *codegen() override;
};

/// PrototypeAST - This class represents the "prototype" for a function,
/// which captures its name, and its argument names (thus implicitly the number
/// of arguments the function takes), as well as if it is an operator.
class PrototypeAST {
  std::string Name;
  std::vector<std::string> Args;
  bool IsOperator;
  unsigned Precedence; // Precedence if a binary op.

public:
  PrototypeAST(const std::string &Name, std::vector<std::string> Args,
               bool IsOperator = false, unsigned Prec = 0)
      : Name(Name), Args(std::move(Args)), IsOperator(IsOperator),
        Precedence(Prec) {}

  Function *codegen();
  const std::string &getName() const { return Name; }

  bool isUnaryOp() const { return IsOperator && Args.size() == 1; }
  bool isBinaryOp() const { return IsOperator && Args.size() == 2; }

  char getOperatorName() const {
    assert(isUnaryOp() || isBinaryOp());
    return Name[Name.size() - 1];
  }

  unsigned getBinaryPrecedence() const { return Precedence; }
};

/// FunctionAST - This class represents a function definition itself.
class FunctionAST {
  std::unique_ptr<PrototypeAST> Proto;
  std::unique_ptr<ExprAST> Body;

public:
  FunctionAST(std::unique_ptr<PrototypeAST> Proto,
              std::unique_ptr<ExprAST> Body)
      : Proto(std::move(Proto)), Body(std::move(Body)) {}

  Function *codegen();
};

} // end anonymous namespace

//===----------------------------------------------------------------------===//
// Parser
//===----------------------------------------------------------------------===//

/// CurTok/getNextToken - Provide a simple token buffer.  CurTok is the current
/// token the parser is looking at.  getNextToken reads another token from the
/// lexer and updates CurTok with its results.
static int CurTok;
static int getNextToken() { return CurTok = gettok(); }

/// BinopPrecedence - This holds the precedence for each binary operator that is
/// defined.
static std::map<char, int> BinopPrecedence;

/// GetTokPrecedence - Get the precedence of the pending binary operator token.
static int GetTokPrecedence() {
  if (!isascii(CurTok))
    return -1;

  // Make sure it's a declared binop.
  int TokPrec = BinopPrecedence[CurTok];
  if (TokPrec <= 0)
    return -1;
  return TokPrec;
}

/// LogError* - These are little helper functions for error handling.
std::unique_ptr<ExprAST> LogError(const char *Str) {
  fprintf(stderr, "Error: %s\n", Str);
  return nullptr;
}

std::unique_ptr<PrototypeAST> LogErrorP(const char *Str) {
  LogError(Str);
  return nullptr;
}

static std::unique_ptr<ExprAST> ParseExpression();

/// numberexpr ::= number
static std::unique_ptr<ExprAST> ParseNumberExpr() {
  auto Result = std::make_unique<NumberExprAST>(NumVal);
  getNextToken(); // consume the number
  return std::move(Result);
}

/// parenexpr ::= '(' expression ')'
static std::unique_ptr<ExprAST> ParseParenExpr() {
  getNextToken(); // eat (.
  auto V = ParseExpression();
  if (!V)
    return nullptr;

  if (CurTok != ')')
    return LogError("expected ')'");
  getNextToken(); // eat ).
  return V;
}

/// identifierexpr
///   ::= identifier
///   ::= identifier '(' expression* ')'
static std::unique_ptr<ExprAST> ParseIdentifierExpr() {
  std::string IdName = IdentifierStr;

  getNextToken(); // eat identifier.

  if (CurTok != '(') // Simple variable ref.
    return std::make_unique<VariableExprAST>(IdName);

  // Call.
  getNextToken(); // eat (
  std::vector<std::unique_ptr<ExprAST>> Args;
  if (CurTok != ')') {
    while (true) {
      if (auto Arg = ParseExpression())
        Args.push_back(std::move(Arg));
      else
        return nullptr;

      if (CurTok == ')')
        break;

      if (CurTok != ',')
        return LogError("Expected ')' or ',' in argument list");
      getNextToken();
    }
  }

  // Eat the ')'.
  getNextToken();

  return std::make_unique<CallExprAST>(IdName, std::move(Args));
}

/// ifexpr ::= 'if' expression 'then' expression 'else' expression
static std::unique_ptr<ExprAST> ParseIfExpr() {
  getNextToken(); // eat the if.

  // condition.
  auto Cond = ParseExpression();
  if (!Cond)
    return nullptr;

  if (CurTok != tok_then)
    return LogError("expected then");
  getNextToken(); // eat the then

  auto Then = ParseExpression();
  if (!Then)
    return nullptr;

  if (CurTok != tok_else)
    return LogError("expected else");

  getNextToken();

  auto Else = ParseExpression();
  if (!Else)
    return nullptr;

  return std::make_unique<IfExprAST>(std::move(Cond), std::move(Then),
                                      std::move(Else));
}

/// forexpr ::= 'for' identifier '=' expr ',' expr (',' expr)? 'in' expression
static std::unique_ptr<ExprAST> ParseForExpr() {
  getNextToken(); // eat the for.

  if (CurTok != tok_identifier)
    return LogError("expected identifier after for");

  std::string IdName = IdentifierStr;
  getNextToken(); // eat identifier.

  if (CurTok != '=')
    return LogError("expected '=' after for");
  getNextToken(); // eat '='.

  auto Start = ParseExpression();
  if (!Start)
    return nullptr;
  if (CurTok != ',')
    return LogError("expected ',' after for start value");
  getNextToken();

  auto End = ParseExpression();
  if (!End)
    return nullptr;

  // The step value is optional.
  std::unique_ptr<ExprAST> Step;
  if (CurTok == ',') {
    getNextToken();
    Step = ParseExpression();
    if (!Step)
      return nullptr;
  }

  if (CurTok != tok_in)
    return LogError("expected 'in' after for");
  getNextToken(); // eat 'in'.

  auto Body = ParseExpression();
  if (!Body)
    return nullptr;

  return std::make_unique<ForExprAST>(IdName, std::move(Start), std::move(End),
                                       std::move(Step), std::move(Body));
}

/// varexpr ::= 'var' identifier ('=' expression)?
//                    (',' identifier ('=' expression)?)* 'in' expression
static std::unique_ptr<ExprAST> ParseVarExpr() {
  getNextToken(); // eat the var.

  std::vector<std::pair<std::string, std::unique_ptr<ExprAST>>> VarNames;

  // At least one variable name is required.
  if (CurTok != tok_identifier)
    return LogError("expected identifier after var");

  while (true) {
    std::string Name = IdentifierStr;
    getNextToken(); // eat identifier.

    // Read the optional initializer.
    std::unique_ptr<ExprAST> Init = nullptr;
    if (CurTok == '=') {
      getNextToken(); // eat the '='.

      Init = ParseExpression();
      if (!Init)
        return nullptr;
    }

    VarNames.push_back(std::make_pair(Name, std::move(Init)));

    // End of var list, exit loop.
    if (CurTok != ',')
      break;
    getNextToken(); // eat the ','.

    if (CurTok != tok_identifier)
      return LogError("expected identifier list after var");
  }

  // At this point, we have to have 'in'.
  if (CurTok != tok_in)
    return LogError("expected 'in' keyword after 'var'");
  getNextToken(); // eat 'in'.

  auto Body = ParseExpression();
  if (!Body)
    return nullptr;

  return std::make_unique<VarExprAST>(std::move(VarNames), std::move(Body));
}

/// primary
///   ::= identifierexpr
///   ::= numberexpr
///   ::= parenexpr
///   ::= ifexpr
///   ::= forexpr
///   ::= varexpr
static std::unique_ptr<ExprAST> ParsePrimary() {
  switch (CurTok) {
  default:
    return LogError("unknown token when expecting an expression");
  case tok_identifier:
    return ParseIdentifierExpr();
  case tok_number:
    return ParseNumberExpr();
  case '(':
    return ParseParenExpr();
  case tok_if:
    return ParseIfExpr();
  case tok_for:
    return ParseForExpr();
  case tok_var:
    return ParseVarExpr();
  }
}

/// unary
///   ::= primary
///   ::= '!' unary
static std::unique_ptr<ExprAST> ParseUnary() {
  // If the current token is not an operator, it must be a primary expr.
  if (!isascii(CurTok) || CurTok == '(' || CurTok == ',')
    return ParsePrimary();

  // If this is a unary operator, read it.
  int Opc = CurTok;
  getNextToken();
  if (auto Operand = ParseUnary())
    return std::make_unique<UnaryExprAST>(Opc, std::move(Operand));
  return nullptr;
}

/// binoprhs
///   ::= ('+' unary)*
static std::unique_ptr<ExprAST> ParseBinOpRHS(int ExprPrec,
                                              std::unique_ptr<ExprAST> LHS) {
  // If this is a binop, find its precedence.
  while (true) {
    int TokPrec = GetTokPrecedence();

    // If this is a binop that binds at least as tightly as the current binop,
    // consume it, otherwise we are done.
    if (TokPrec < ExprPrec)
      return LHS;

    // Okay, we know this is a binop.
    int BinOp = CurTok;
    getNextToken(); // eat binop

    // Parse the unary expression after the binary operator.
    auto RHS = ParseUnary();
    if (!RHS)
      return nullptr;

    // If BinOp binds less tightly with RHS than the operator after RHS, let
    // the pending operator take RHS as its LHS.
    int NextPrec = GetTokPrecedence();
    if (TokPrec < NextPrec) {
      RHS = ParseBinOpRHS(TokPrec + 1, std::move(RHS));
      if (!RHS)
        return nullptr;
    }

    // Merge LHS/RHS.
    LHS =
        std::make_unique<BinaryExprAST>(BinOp, std::move(LHS), std::move(RHS));
  }
}

/// expression
///   ::= unary binoprhs
///
static std::unique_ptr<ExprAST> ParseExpression() {
  auto LHS = ParseUnary();
  if (!LHS)
    return nullptr;

  return ParseBinOpRHS(0, std::move(LHS));
}

/// prototype
///   ::= id '(' id* ')'
///   ::= binary LETTER number? (id, id)
///   ::= unary LETTER (id)
static std::unique_ptr<PrototypeAST> ParsePrototype() {
  std::string FnName;

  unsigned Kind = 0; // 0 = identifier, 1 = unary, 2 = binary.
  unsigned BinaryPrecedence = 30;

  switch (CurTok) {
  default:
    return LogErrorP("Expected function name in prototype");
  case tok_identifier:
    FnName = IdentifierStr;
    Kind = 0;
    getNextToken();
    break;
  case tok_unary:
    getNextToken();
    if (!isascii(CurTok))
      return LogErrorP("Expected unary operator");
    FnName = "unary";
    FnName += (char)CurTok;
    Kind = 1;
    getNextToken();
    break;
  case tok_binary:
    getNextToken();
    if (!isascii(CurTok))
      return LogErrorP("Expected binary operator");
    FnName = "binary";
    FnName += (char)CurTok;
    Kind = 2;
    getNextToken();

    // Read the precedence if present.
    if (CurTok == tok_number) {
      if (NumVal < 1 || NumVal > 100)
        return LogErrorP("Invalid precedence: must be 1..100");
      BinaryPrecedence = (unsigned)NumVal;
      getNextToken();
    }
    break;
  }

  if (CurTok != '(')
    return LogErrorP("Expected '(' in prototype");

  std::vector<std::string> ArgNames;
  while (getNextToken() == tok_identifier)
    ArgNames.push_back(IdentifierStr);
  if (CurTok != ')')
    return LogErrorP("Expected ')' in prototype");

  // success.
  getNextToken(); // eat ')'.

  // Verify right number of names for operator.
  if (Kind && ArgNames.size() != Kind)
    return LogErrorP("Invalid number of operands for operator");

  return std::make_unique<PrototypeAST>(FnName, ArgNames, Kind != 0,
                                         BinaryPrecedence);
}

/// definition ::= 'def' prototype expression
static std::unique_ptr<FunctionAST> ParseDefinition() {
  getNextToken(); // eat def.
  auto Proto = ParsePrototype();
  if (!Proto)
    return nullptr;

  if (auto E = ParseExpression())
    return std::make_unique<FunctionAST>(std::move(Proto), std::move(E));
  return nullptr;
}

/// toplevelexpr ::= expression
static std::unique_ptr<FunctionAST> ParseTopLevelExpr() {
  if (auto E = ParseExpression()) {
    // Make an anonymous proto.
    auto Proto = std::make_unique<PrototypeAST>("__anon_expr",
                                                 std::vector<std::string>());
    return std::make_unique<FunctionAST>(std::move(Proto), std::move(E));
  }
  return nullptr;
}

/// external ::= 'extern' prototype
static std::unique_ptr<PrototypeAST> ParseExtern() {
  getNextToken(); // eat extern.
  return ParsePrototype();
}

//===----------------------------------------------------------------------===//
// Code Generation
//===----------------------------------------------------------------------===//

static std::unique_ptr<LLVMContext> TheContext;
static std::unique_ptr<Module> TheModule;
static std::unique_ptr<IRBuilder<>> Builder;
static std::map<std::string, AllocaInst *> NamedValues;
static std::unique_ptr<legacy::FunctionPassManager> TheFPM;
static std::unique_ptr<KaleidoscopeJIT> TheJIT;
static std::map<std::string, std::unique_ptr<PrototypeAST>> FunctionProtos;
static ExitOnError ExitOnErr;

Value *LogErrorV(const char *Str) {
  LogError(Str);
  return nullptr;
}

Function *getFunction(std::string Name) {
  // First, see if the function has already been added to the current module.
  if (auto *F = TheModule->getFunction(Name))
    return F;

  // If not, check whether we can codegen the declaration from some existing
  // prototype.
  auto FI = FunctionProtos.find(Name);
  if (FI != FunctionProtos.end())
    return FI->second->codegen();

  // If no existing prototype exists, return null.
  return nullptr;
}

/// CreateEntryBlockAlloca - Create an alloca instruction in the entry block of
/// the function.  This is used for mutable variables etc.
static AllocaInst *CreateEntryBlockAlloca(Function *TheFunction,
                                          StringRef VarName) {
  IRBuilder<> TmpB(&TheFunction->getEntryBlock(),
                   TheFunction->getEntryBlock().begin());
  return TmpB.CreateAlloca(Type::getDoubleTy(*TheContext), nullptr, VarName);
}

Value *NumberExprAST::codegen() {
  return ConstantFP::get(*TheContext, APFloat(Val));
}

Value *VariableExprAST::codegen() {
  // Look this variable up in the function.
  AllocaInst *A = NamedValues[Name];
  if (!A)
    return LogErrorV("Unknown variable name");

  // Load the value.
  return Builder->CreateLoad(A->getAllocatedType(), A, Name.c_str());
}

Value *UnaryExprAST::codegen() {
  Value *OperandV = Operand->codegen();
  if (!OperandV)
    return nullptr;

  Function *F = getFunction(std::string("unary") + Opcode);
  if (!F)
    return LogErrorV("Unknown unary operator");

  return Builder->CreateCall(F, OperandV, "unop");
}

Value *BinaryExprAST::codegen() {
  // Special case '=' because we don't want to emit the LHS as an expression.
  if (Op == '=') {
    // Assignment requires the LHS to be an identifier.
    // This assume we're building without RTTI because LLVM builds that way by
    // default.  If you build LLVM with RTTI this can be changed to a
    // dynamic_cast for automatic error checking.
    VariableExprAST *LHSE = static_cast<VariableExprAST *>(LHS.get());
    if (!LHSE)
      return LogErrorV("destination of '=' must be a variable");
    // Codegen the RHS.
    Value *Val = RHS->codegen();
    if (!Val)
      return nullptr;

    // Look up the name.
    Value *Variable = NamedValues[LHSE->getName()];
    if (!Variable)
      return LogErrorV("Unknown variable name");

    Builder->CreateStore(Val, Variable);
    return Val;
  }

  Value *L = LHS->codegen();
  Value *R = RHS->codegen();
  if (!L || !R)
    return nullptr;

  switch (Op) {
  case '+':
    return Builder->CreateFAdd(L, R, "addtmp");
  case '-':
    return Builder->CreateFSub(L, R, "subtmp");
  case '*':
    return Builder->CreateFMul(L, R, "multmp");
  case '<':
    L = Builder->CreateFCmpULT(L, R, "cmptmp");
    // Convert bool 0/1 to double 0.0 or 1.0
    return Builder->CreateUIToFP(L, Type::getDoubleTy(*TheContext), "booltmp");
  default:
    break;
  }

  // If it wasn't a builtin binary operator, it must be a user defined one. Emit
  // a call to it.
  Function *F = getFunction(std::string("binary") + Op);
  assert(F && "binary operator not found!");

  Value *Ops[] = {L, R};
  return Builder->CreateCall(F, Ops, "binop");
}

Value *CallExprAST::codegen() {
  // Look up the name in the global module table.
  Function *CalleeF = getFunction(Callee);
  if (!CalleeF)
    return LogErrorV("Unknown function referenced");

  // If argument mismatch error.
  if (CalleeF->arg_size() != Args.size())
    return LogErrorV("Incorrect # arguments passed");

  std::vector<Value *> ArgsV;
  for (unsigned i = 0, e = Args.size(); i != e; ++i) {
    ArgsV.push_back(Args[i]->codegen());
    if (!ArgsV.back())
      return nullptr;
  }

  return Builder->CreateCall(CalleeF, ArgsV, "calltmp");
}

Value *IfExprAST::codegen() {
  Value *CondV = Cond->codegen();
  if (!CondV)
    return nullptr;

  // Convert condition to a bool by comparing non-equal to 0.0.
  CondV = Builder->CreateFCmpONE(
      CondV, ConstantFP::get(*TheContext, APFloat(0.0)), "ifcond");

  Function *TheFunction = Builder->GetInsertBlock()->getParent();

  // Create blocks for the then and else cases.  Insert the 'then' block at the
  // end of the function.
  BasicBlock *ThenBB = BasicBlock::Create(*TheContext, "then", TheFunction);
  BasicBlock *ElseBB = BasicBlock::Create(*TheContext, "else");
  BasicBlock *MergeBB = BasicBlock::Create(*TheContext, "ifcont");

  Builder->CreateCondBr(CondV, ThenBB, ElseBB);

  // Emit then value.
  Builder->SetInsertPoint(ThenBB);

  Value *ThenV = Then->codegen();
  if (!ThenV)
    return nullptr;

  Builder->CreateBr(MergeBB);
  // Codegen of 'Then' can change the current block, update ThenBB for the PHI.
  ThenBB = Builder->GetInsertBlock();

  // Emit else block.
  TheFunction->getBasicBlockList().push_back(ElseBB);
  Builder->SetInsertPoint(ElseBB);

  Value *ElseV = Else->codegen();
  if (!ElseV)
    return nullptr;

  Builder->CreateBr(MergeBB);
  // Codegen of 'Else' can change the current block, update ElseBB for the PHI.
  ElseBB = Builder->GetInsertBlock();

  // Emit merge block.
  TheFunction->getBasicBlockList().push_back(MergeBB);
  Builder->SetInsertPoint(MergeBB);
  PHINode *PN = Builder->CreatePHI(Type::getDoubleTy(*TheContext), 2, "iftmp");

  PN->addIncoming(ThenV, ThenBB);
  PN->addIncoming(ElseV, ElseBB);
  return PN;
}

// Output for-loop as:
//   var = alloca double
//   ...
//   start = startexpr
//   store start -> var
//   goto loop
// loop:
//   ...
//   bodyexpr
//   ...
// loopend:
//   step = stepexpr
//   endcond = endexpr
//
//   curvar = load var
//   nextvar = curvar + step
//   store nextvar -> var
//   br endcond, loop, endloop
// outloop:
Value *ForExprAST::codegen() {
  Function *TheFunction = Builder->GetInsertBlock()->getParent();

  // Create an alloca for the variable in the entry block.
  AllocaInst *Alloca = CreateEntryBlockAlloca(TheFunction, VarName);

  // Emit the start code first, without 'variable' in scope.
  Value *StartVal = Start->codegen();
  if (!StartVal)
    return nullptr;

  // Store the value into the alloca.
  Builder->CreateStore(StartVal, Alloca);

  // Make the new basic block for the loop header, inserting after current
  // block.
  BasicBlock *LoopBB = BasicBlock::Create(*TheContext, "loop", TheFunction);

  // Insert an explicit fall through from the current block to the LoopBB.
  Builder->CreateBr(LoopBB);

  // Start insertion in LoopBB.
  Builder->SetInsertPoint(LoopBB);

  // Within the loop, the variable is defined equal to the PHI node.  If it
  // shadows an existing variable, we have to restore it, so save it now.
  AllocaInst *OldVal = NamedValues[VarName];
  NamedValues[VarName] = Alloca;

  // Emit the body of the loop.  This, like any other expr, can change the
  // current BB.  Note that we ignore the value computed by the body, but don't
  // allow an error.
  if (!Body->codegen())
    return nullptr;

  // Emit the step value.
  Value *StepVal = nullptr;
  if (Step) {
    StepVal = Step->codegen();
    if (!StepVal)
      return nullptr;
  } else {
    // If not specified, use 1.0.
    StepVal = ConstantFP::get(*TheContext, APFloat(1.0));
  }

  // Compute the end condition.
  Value *EndCond = End->codegen();
  if (!EndCond)
    return nullptr;

  // Reload, increment, and restore the alloca.  This handles the case where
  // the body of the loop mutates the variable.
  Value *CurVar =
      Builder->CreateLoad(Alloca->getAllocatedType(), Alloca, VarName.c_str());
  Value *NextVar = Builder->CreateFAdd(CurVar, StepVal, "nextvar");
  Builder->CreateStore(NextVar, Alloca);

  // Convert condition to a bool by comparing non-equal to 0.0.
  EndCond = Builder->CreateFCmpONE(
      EndCond, ConstantFP::get(*TheContext, APFloat(0.0)), "loopcond");

  // Create the "after loop" block and insert it.
  BasicBlock *AfterBB =
      BasicBlock::Create(*TheContext, "afterloop", TheFunction);

  // Insert the conditional branch into the end of LoopEndBB.
  Builder->CreateCondBr(EndCond, LoopBB, AfterBB);

  // Any new code will be inserted in AfterBB.
  Builder->SetInsertPoint(AfterBB);

  // Restore the unshadowed variable.
  if (OldVal)
    NamedValues[VarName] = OldVal;
  else
    NamedValues.erase(VarName);

  // for expr always returns 0.0.
  return Constant::getNullValue(Type::getDoubleTy(*TheContext));
}

Value *VarExprAST::codegen() {
  std::vector<AllocaInst *> OldBindings;

  Function *TheFunction = Builder->GetInsertBlock()->getParent();

  // Register all variables and emit their initializer.
  for (unsigned i = 0, e = VarNames.size(); i != e; ++i) {
    const std::string &VarName = VarNames[i].first;
    ExprAST *Init = VarNames[i].second.get();

    // Emit the initializer before adding the variable to scope, this prevents
    // the initializer from referencing the variable itself, and permits stuff
    // like this:
    //  var a = 1 in
    //    var a = a in ...   # refers to outer 'a'.
    Value *InitVal;
    if (Init) {
      InitVal = Init->codegen();
      if (!InitVal)
        return nullptr;
    } else { // If not specified, use 0.0.
      InitVal = ConstantFP::get(*TheContext, APFloat(0.0));
    }

    AllocaInst *Alloca = CreateEntryBlockAlloca(TheFunction, VarName);
    Builder->CreateStore(InitVal, Alloca);

    // Remember the old variable binding so that we can restore the binding when
    // we unrecurse.
    OldBindings.push_back(NamedValues[VarName]);

    // Remember this binding.
    NamedValues[VarName] = Alloca;
  }

  // Codegen the body, now that all vars are in scope.
  Value *BodyVal = Body->codegen();
  if (!BodyVal)
    return nullptr;

  // Pop all our variables from scope.
  for (unsigned i = 0, e = VarNames.size(); i != e; ++i)
    NamedValues[VarNames[i].first] = OldBindings[i];

  // Return the body computation.
  return BodyVal;
}

Function *PrototypeAST::codegen() {
  // Make the function type:  double(double,double) etc.
  std::vector<Type *> Doubles(Args.size(), Type::getDoubleTy(*TheContext));
  FunctionType *FT =
      FunctionType::get(Type::getDoubleTy(*TheContext), Doubles, false);

  Function *F =
      Function::Create(FT, Function::ExternalLinkage, Name, TheModule.get());

  // Set names for all arguments.
  unsigned Idx = 0;
  for (auto &Arg : F->args())
    Arg.setName(Args[Idx++]);

  return F;
}

Function *FunctionAST::codegen() {
  // Transfer ownership of the prototype to the FunctionProtos map, but keep a
  // reference to it for use below.
  auto &P = *Proto;
  FunctionProtos[Proto->getName()] = std::move(Proto);
  Function *TheFunction = getFunction(P.getName());
  if (!TheFunction)
    return nullptr;

  // If this is an operator, install it.
  if (P.isBinaryOp())
    BinopPrecedence[P.getOperatorName()] = P.getBinaryPrecedence();

  // Create a new basic block to start insertion into.
  BasicBlock *BB = BasicBlock::Create(*TheContext, "entry", TheFunction);
  Builder->SetInsertPoint(BB);

  // Record the function arguments in the NamedValues map.
  NamedValues.clear();
  for (auto &Arg : TheFunction->args()) {
    // Create an alloca for this variable.
    AllocaInst *Alloca = CreateEntryBlockAlloca(TheFunction, Arg.getName());

    // Store the initial value into the alloca.
    Builder->CreateStore(&Arg, Alloca);

    // Add arguments to variable symbol table.
    NamedValues[std::string(Arg.getName())] = Alloca;
  }

  if (Value *RetVal = Body->codegen()) {
    // Finish off the function.
    Builder->CreateRet(RetVal);

    // Validate the generated code, checking for consistency.
    verifyFunction(*TheFunction);

    // Run the optimizer on the function.
    TheFPM->run(*TheFunction);

    return TheFunction;
  }

  // Error reading body, remove function.
  TheFunction->eraseFromParent();

  if (P.isBinaryOp())
    BinopPrecedence.erase(P.getOperatorName());
  return nullptr;
}

//===----------------------------------------------------------------------===//
// Top-Level parsing and JIT Driver
//===----------------------------------------------------------------------===//

static void InitializeModuleAndPassManager() {
  // Open a new module.
  TheContext = std::make_unique<LLVMContext>();
  TheModule = std::make_unique<Module>("my cool jit", *TheContext);
  TheModule->setDataLayout(TheJIT->getDataLayout());

  // Create a new builder for the module.
  Builder = std::make_unique<IRBuilder<>>(*TheContext);

  // Create a new pass manager attached to it.
  TheFPM = std::make_unique<legacy::FunctionPassManager>(TheModule.get());

  // Promote allocas to registers.
  TheFPM->add(createPromoteMemoryToRegisterPass());
  // Do simple "peephole" optimizations and bit-twiddling optzns.
  TheFPM->add(createInstructionCombiningPass());
  // Reassociate expressions.
  TheFPM->add(createReassociatePass());
  // Eliminate Common SubExpressions.
  TheFPM->add(createGVNPass());
  // Simplify the control flow graph (deleting unreachable blocks, etc).
  TheFPM->add(createCFGSimplificationPass());

  TheFPM->doInitialization();
}

static void HandleDefinition() {
  if (auto FnAST = ParseDefinition()) {
    if (auto *FnIR = FnAST->codegen()) {
      fprintf(stderr, "Read function definition:");
      FnIR->print(errs());
      fprintf(stderr, "\n");
      ExitOnErr(TheJIT->addModule(
          ThreadSafeModule(std::move(TheModule), std::move(TheContext))));
      InitializeModuleAndPassManager();
    }
  } else {
    // Skip token for error recovery.
    getNextToken();
  }
}

static void HandleExtern() {
  if (auto ProtoAST = ParseExtern()) {
    if (auto *FnIR = ProtoAST->codegen()) {
      fprintf(stderr, "Read extern: ");
      FnIR->print(errs());
      fprintf(stderr, "\n");
      FunctionProtos[ProtoAST->getName()] = std::move(ProtoAST);
    }
  } else {
    // Skip token for error recovery.
    getNextToken();
  }
}

static void HandleTopLevelExpression() {
  // Evaluate a top-level expression into an anonymous function.
  if (auto FnAST = ParseTopLevelExpr()) {
    if (FnAST->codegen()) {
      // Create a ResourceTracker to track JIT'd memory allocated to our
      // anonymous expression -- that way we can free it after executing.
      auto RT = TheJIT->getMainJITDylib().createResourceTracker();

      auto TSM = ThreadSafeModule(std::move(TheModule), std::move(TheContext));
      ExitOnErr(TheJIT->addModule(std::move(TSM), RT));
      InitializeModuleAndPassManager();

      // Search the JIT for the __anon_expr symbol.
      auto ExprSymbol = ExitOnErr(TheJIT->lookup("__anon_expr"));

      // Get the symbol's address and cast it to the right type (takes no
      // arguments, returns a double) so we can call it as a native function.
      double (*FP)() = (double (*)())(intptr_t)ExprSymbol.getAddress();
      fprintf(stderr, "Evaluated to %f\n", FP());

      // Delete the anonymous expression module from the JIT.
      ExitOnErr(RT->remove());
    }
  } else {
    // Skip token for error recovery.
    getNextToken();
  }
}

/// top ::= definition | external | expression | ';'
static void MainLoop() {
  while (true) {
    fprintf(stderr, "ready> ");
    switch (CurTok) {
    case tok_eof:
      return;
    case ';': // ignore top-level semicolons.
      getNextToken();
      break;
    case tok_def:
      HandleDefinition();
      break;
    case tok_extern:
      HandleExtern();
      break;
    default:
      HandleTopLevelExpression();
      break;
    }
  }
}

//===----------------------------------------------------------------------===//
// "Library" functions that can be "extern'd" from user code.
//===----------------------------------------------------------------------===//

#ifdef _WIN32
#define DLLEXPORT __declspec(dllexport)
#else
#define DLLEXPORT
#endif

/// putchard - putchar that takes a double and returns 0.
extern "C" DLLEXPORT double putchard(double X) {
  fputc((char)X, stderr);
  return 0;
}

/// printd - printf that takes a double prints it as "%f\n", returning 0.
extern "C" DLLEXPORT double printd(double X) {
  fprintf(stderr, "%f\n", X);
  return 0;
}

//===----------------------------------------------------------------------===//
// Main driver code.
//===----------------------------------------------------------------------===//

int main() {
  InitializeNativeTarget();
  InitializeNativeTargetAsmPrinter();
  InitializeNativeTargetAsmParser();

  // Install standard binary operators.
  // 1 is lowest precedence.
  BinopPrecedence['='] = 2;
  BinopPrecedence['<'] = 10;
  BinopPrecedence['+'] = 20;
  BinopPrecedence['-'] = 20;
  BinopPrecedence['*'] = 40; // highest.

  // Prime the first token.
  fprintf(stderr, "ready> ");
  getNextToken();

  TheJIT = ExitOnErr(KaleidoscopeJIT::Create());

  InitializeModuleAndPassManager();

  // Run the main "interpreter loop" now.
  MainLoop();

  return 0;
}
```
