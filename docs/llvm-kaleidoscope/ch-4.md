# 第 4 章：增加 JIT 与优化

## 4.1 前言

欢迎来到 [我的第一个基于 LLVM 的语言前端](ch-0.md) 教程第四章。前三章描述了一门简单语言的实现与生成 LLVM IR 的过程，而本章将要描述两种新技术 -- 增加优化器 (optimizer) 与 JIT 编译器支持 -- 它们将会告诉你如何生成优雅高效的中间代码。

## 4.2 平凡的常数折叠

我们在第三章的实现非常优雅，且易于扩展。但美中不足的是，它并不会生成很好的中间代码。虽然 `IRBuilder` 在我们编译简单的代码的时候确实为我们做了一些显然的优化：

```py
ready> def test(x) 1+2+x;
Read function definition:
define double @test(double %x) {
entry:
        %addtmp = fadd double 3.000000e+00, %x
        ret double %addtmp
}
```

上面的中间代码并不是解析出的 AST 的简单翻译 -- 它并不是下面这样：

```py
ready> def test(x) 1+2+x;
Read function definition:
define double @test(double %x) {
entry:
        %addtmp = fadd double 2.000000e+00, 1.000000e+00
        %addtmp1 = fadd double %addtmp, %x
        ret double %addtmp1
}
```

正如你所见，常数折叠 (Constant folding) 在实践中是一项非常普遍并且重要的优化：许多语言实现者直接在 AST 构建中就实现了常数折叠优化。

有了 LLVM，你就不必直接在 AST 中支持它了。因为构造 LLVM IR 的所有调用都经过了 `IRBuilder`, 所以 `IRBuilder` 便能自行确定它是否有进行常数折叠优化的可能。如果可能，它就直接折叠并且返回那个常数，而不是创建新的指令。

好吧，上面的优化是很简单的 :). 实践中，我们建议你总是使用 `IRBuilder` 来生成中间代码。以它来实现常数折叠可以避免在 AST 构造的代码中引入过多的 "语法杂音" (你肯定不希望你的编译器里到处都是丑陋的常数判断！), 并且 `IRBuilder` 在特定情况下可以令人惊叹地减少要生成的 IR 数量 (特别是对那些有预处理宏的或者是大量使用常量的语言而言). 

另一方面，`IRBuilder` 只能分析它当下那一条指令，如果你使用下面这个复杂一点的例子：

```py
ready> def test(x) (1+2+x)*(x+(1+2));
ready> Read function definition:
define double @test(double %x) {
entry:
        %addtmp = fadd double 3.000000e+00, %x
        %addtmp1 = fadd double %x, 3.000000e+00
        %multmp = fmul double %addtmp, %addtmp1
        ret double %multmp
}
```

在这种情况下，我们可以看到乘法的左右子表达式是相同的。比起计算 `x+3` 两次，我们更想看见我们的编译器生成 `tmp = x+3; result = tmp * tmp` 这样的代码。

不幸的是，没有任何一种 (指令层面的) 局部分析可以检测并优化这种情况。我们至少需要两种变换才能消除冗余的加法指令：重新关联表达式 (以便使加法在词法上等同) 与公共子表达式消除 (Common Subexpression Elimination, CSE). 幸运的是，LLVM 以 "pass" 的形式提供了广泛的优化选择。

## 4.3 LLVM 优化过程 (pass)

> **Warning**: 因为 `PassManager` 正转向新架构，本教程使用的 `PassManager` 将会基于 [LegacyPassManager.h](https://llvm.org/doxygen/classllvm_1_1legacy_1_1FunctionPassManager.html) 中的 `llvm::legacy::FunctionPassManager`. 基于教程的目的，老的 `PassManager` 将被继续使用直到新 `PassManager` 彻底转向新架构为止。

LLVM 提供了许多优化过程，它们都有不同的作用，也有不一样的折中。LLVM 并不会奢望某一套特定的优化可以适用于所有语言的所有情况，所以它允许编译器实现者来完全决定要在何处以何种顺序使用何种特定优化。

作为一个具体的例子，LLVM 既支持 "全模块" 优化过程 ("whole module" passes) -- 这种过程可以搜集一大块中间代码的信息 (一般而言，这指一整份代码文件; 但如果此种优化在链接时发生，其包含代码的范围可能是一整个程序的一大部分), 也支持函数内优化过程 ("per-function" passes) -- 这种过程专注于单个函数内部的优化。如果你对优化过程的细节与运行方式很感兴趣，请查阅 [How to Write a Pass](https://llvm.org/docs/WritingAnLLVMPass.html) 与 [List of LLVM Passes](https://llvm.org/docs/Passes.html).

对 Kaleidoscope 而言，我们现在只能动态地一个一个在用户输入的时候生成函数的中间代码，我们不会指望在这种情况下获得最好的优化体验，但至少我们希望我们的编译器可以执行一些快速简单的可能优化。所以，当用户键入函数时，我们将只执行一小些函数内优化过程。如果我们想要创造一个 "静态 Kaleidoscope 编译器", 我们将会拥有整份我们将要编译的代码的信息，在那时我们就可以在解析完整个文件之后再运行优化器了。

为了让函数内优化过程正常工作，我们需要初始化一个 [FunctionPassManager](https://llvm.org/docs/WritingAnLLVMPass.html#what-passmanager-doesr) 来管理与组织我们想要运行的优化过程。一旦它被初始化了，我们就可以向它加入一系列的优化并加以运行。对每一个我们想要优化的模块，我们都需要一个新的 FunctionPassManager，所以我们编写一个函数来为每一个模块创建并初始化一个优化过程管理器：

```cpp
void InitializeModuleAndPassManager(void) {
  // Open a new module.
  // 打开一个新的模块
  TheModule = std::make_unique<Module>("my cool jit", TheContext);

  // Create a new pass manager attached to it.
  // 创建一个新的优化过程管理器并将其关联到模块上
  TheFPM = std::make_unique<legacy::FunctionPassManager>(TheModule.get());

  // Do simple "peephole" optimizations and bit-twiddling optzns.
  // 做一些简单的窥孔优化与位运算黑魔法
  TheFPM->add(createInstructionCombiningPass());
  // Reassociate expressions.
  // 重关联表达式
  TheFPM->add(createReassociatePass());
  // Eliminate Common SubExpressions.
  // 消除公共子表达式
  TheFPM->add(createGVNPass());
  // Simplify the control flow graph (deleting unreachable blocks, etc).
  // 简化控制流图 (比如删除不可达的基本块)
  TheFPM->add(createCFGSimplificationPass());

  TheFPM->doInitialization();
}
```

上面的代码初始化了一个全局模块 `TheModule` 和与之关联的函数优化过程管理器 `TheFPM`, 随后通过一系列 `add` 方法加入了一些了 LLVM 优化过程。

我们挑选的这四个优化过程基本上就是一组被广泛使用的通用 "打扫" 级优化。本教程不会深入去将它们都干了什么，但这可以成为你学习优化技术的一个起点。

一旦优化过程管理器好了，我们就可以使用它了。我们在构造完函数的中间代码并返回之前 (`FunctionAST::codegen()` 中) 使用这个优化过程管理器：

```cpp
if (Value *RetVal = Body->codegen()) {
  // Finish off the function.
  // 结束函数
  Builder.CreateRet(RetVal);

  // Validate the generated code, checking for consistency.
  // 检查生成的中间代码的有效性
  verifyFunction(*TheFunction);

  // Optimize the function.
  // 优化函数
  TheFPM->run(*TheFunction);

  return TheFunction;
}
``` 

正如你所见，上面的代码相当直观。`FunctionPassManager` 优化并原地更新了 LLVM Function*, (可能) 优化了其函数体。改好之后，我们可以再次尝试我们上面的测试：

```py
ready> def test(x) (1+2+x)*(x+(1+2));
ready> Read function definition:
define double @test(double %x) {
entry:
        %addtmp = fadd double %x, 3.000000e+00
        %multmp = fmul double %addtmp, %addtmp
        ret double %multmp
}
```

如我们期待的那样，我们现在拥有了非常棒的优化后的中间代码，为每一次函数调用节省了一句浮点加法指令。

LLVM 提供了许多在特定情况下可以使用的优化过程，有 [一些优化过程的文档](https://llvm.org/docs/Passes.html)，但并非非常完整。另一个了解它们的使用方法的好方式是去看看 Clang 是如何使用它们的。工具 `opt` 允许你在命令行实验这些优化过程，看看它们都干了什么。

现在我们自前端获得了合理的中间代码，让我们来讨论如何执行它吧。

## 4.4 加入即时编译器 (JIT Compiler)

对于 LLVM IR，我们有一系列的处理工具。比如，你可以在其上运行优化 (正如上面一样), 或者是将其输出为文本或二进制格式。你也可以将其编译为某个目标平台上的汇编语言，或是即时编译它。LLVM IR 的好处就在于它是编译器里不同部分的"通用货币"。

在本节，我们将会为我们的解释器加入即时编译支持。我们对于 Kaleidoscope 的基本需要就是，当用户输入了一个函数体之后，其马上求值其顶层表达式。比如，假设用户输入了 "1+2", 我们应该求值该表达式并输出 "3". 如果用户定义了一个函数，在之后它们应该可以被调用。

为了做到上面这些，我们首先要准备好为当前平台生成机器码的环境，声明并初始化 JIT. 通过调用一些 `InitializeNativeTarget` 开头的函数与初始化一个全局变量 `TheJIT`, 我们便能完成这些：

```cpp
static std::unique_ptr<KaleidoscopeJIT> TheJIT;
...
int main() {
  InitializeNativeTarget();
  InitializeNativeTargetAsmPrinter();
  InitializeNativeTargetAsmParser();

  // Install standard binary operators.
  // 1 is lowest precedence.
  BinopPrecedence['<'] = 10;
  BinopPrecedence['+'] = 20;
  BinopPrecedence['-'] = 20;
  BinopPrecedence['*'] = 40; // highest.

  // Prime the first token.
  fprintf(stderr, "ready> ");
  getNextToken();

  TheJIT = std::make_unique<KaleidoscopeJIT>();

  // Run the main "interpreter loop" now.
  MainLoop();

  return 0;
}
``` 

我们还需要设置 JIT 的数据布局 (data layout): 

```cpp
void InitializeModuleAndPassManager(void) {
  // Open a new module.
  TheModule = std::make_unique<Module>("my cool jit", TheContext);
  TheModule->setDataLayout(TheJIT->getTargetMachine().createDataLayout());

  // Create a new pass manager attached to it.
  TheFPM = std::make_unique<legacy::FunctionPassManager>(TheModule.get());
  ...
```

类 `KaleidoscopeJIT` 是一个简单的专为本教程定义的 JIT. 其位于头文件 `llvm-src/examples/Kaleidoscope/include/KaleidoscopeJIT.h` 中。在之后的章节我们将会探索其运行的方法并扩展它的功能，但现在我们就先直接使用它了。它的 API 十分简单：`addModule` 增加一个 LLVM IR 模块到 JIT，使其中的函数得以运行; `removeModule` 从 JIT 中删除一个模块并释放其所有内存; `findSymbol` 则让我们可以查找编译后的代码的指针。

利用上面的 API，我们可以修改我们对于顶层表达式的解析代码如下：

```cpp
static void HandleTopLevelExpression() {
  // Evaluate a top-level expression into an anonymous function.
  if (auto FnAST = ParseTopLevelExpr()) {
    if (FnAST->codegen()) {

      // JIT the module containing the anonymous expression, keeping a handle so
      // we can free it later.
      // 即时编译包含了匿名表达式的模块，保留其一个句柄以便于之后我们释放它
      auto H = TheJIT->addModule(std::move(TheModule));
      InitializeModuleAndPassManager();

      // Search the JIT for the __anon_expr symbol.
      // 在 JIT 中查找 __anon_expr 符号
      auto ExprSymbol = TheJIT->findSymbol("__anon_expr");
      assert(ExprSymbol && "Function not found");

      // Get the symbol's address and cast it to the right type (takes no
      // arguments, returns a double) so we can call it as a native function.
      // 获得那个符号对应的表达式的地址，将其转换为正确的类型 (不接受参数，返回一个 double)
      // 随后我们便能像调用一个本地函数一样调用它。
      double (*FP)() = (double (*)())(intptr_t)ExprSymbol.getAddress();
      fprintf(stderr, "Evaluated to %f\n", FP());

      // Delete the anonymous expression module from the JIT.
      // 在 JIT 中删除包含匿名表达式的那个模块
      TheJIT->removeModule(H);
    }
```

如果解析与代码生成都成功了，下一步便是向 JIT 中加入包含顶层表达式的模块。我们通过 `addModule` 来实现它，其将会编译该模块内所有函数并且返回一个句柄以供我们后续删除它。模块被加入 JIT 以后它就不能再被修改了，所以我们要重新初始化全局的模块与优化过程管理器。

将模块加入 JIT 后，我们通过调用 `findSymbol` 方法得到最终生成的目标代码。因为我们刚刚才把 `__anon_expr` 加入到 JIT，我们就直接断言我们找到了那个函数。

接下来，我们通过 `getAddress` 方法获得 `__anon_expr` 生成的目标代码的指针并将其转换为合适的类型。因为 JIT 编译器可以自动使用目标平台上的二进制接口 (ABI), 所以我们直接转换并调用那个函数指针就可以了。这意味着 JIT 编译出的代码与你自己编写并编译出的本地代码在使用上是没有区别的。

最后，因为我们不支持对于顶层表达式的重新求值，我们直接从 JIT 中删除编译好的模块来释放其内存。因为我们在之前已经重新初始化了全局的模块了，所以整个程序仍能继续运行并等待用户键入新的函数。

修改上述两个地方之后，让我们来看看现在 Kaleidoscope 的运行结果吧！

```py
ready> 4+5;
Read top-level expression:
define double @0() {
entry:
  ret double 9.000000e+00
}

Evaluated to 9.000000
``` 

它看起来好像是基本能跑了。函数的输出显示我们将输入的顶层表达式解析为一个没有参数，返回 double 的函数。上面测试了基本的工具，让我们来试试更加复杂的：

```py
ready> def testfunc(x y) x + y*2;
Read function definition:
define double @testfunc(double %x, double %y) {
entry:
  %multmp = fmul double %y, 2.000000e+00
  %addtmp = fadd double %multmp, %x
  ret double %addtmp
}

ready> testfunc(4, 10);
Read top-level expression:
define double @1() {
entry:
  %calltmp = call double @testfunc(double 4.000000e+00, double 1.000000e+01)
  ret double %calltmp
}

Evaluated to 24.000000

ready> testfunc(5, 10);
ready> LLVM ERROR: Program used external function 'testfunc' which could not be resolved!
```

函数定义与调用看起来如期工作，但在最后一行发生了意想不到的事！这个函数调用看起来是有效的，所以发生了什么呢？你也许从 API 里猜到了，模块是 JIT 分配与删除的单位，并且 testfunc 与匿名的顶层表达式包含在同一个模块里。当我们从 JIT 里删除包含着顶层表达式的模块的时候，我们也顺带把 testfunc 的定义删除了。所以，当我们第二次调用 testfunc 时，JIT 就再也找不到它了。

修正这个 bug 的简单方法是将匿名表达式放在与其他函数不同的另一个模块之中。只要被调用的函数的原型在调用前被加入到 JIT, JIT 就可以跨越模块搜寻被调用的函数。将匿名表达式放入不同的模块之中可以让我们在不影响其他函数的前提下删除匿名表达式。

事实上，我们将更进一步：我们将把每一个函数都放入一个单独的模块。这样做利用了 KaleidoscopeJIT 的一个有用的性质：当查找一个被多次加入的函数时，JIT 将会返回其最后一次定义。利用这个性质，我们的程序就更加地 "REPL" 化了：

```py
ready> def foo(x) x + 1;
Read function definition:
define double @foo(double %x) {
entry:
  %addtmp = fadd double %x, 1.000000e+00
  ret double %addtmp
}

ready> foo(2);
Evaluated to 3.000000

ready> def foo(x) x + 2;
define double @foo(double %x) {
entry:
  %addtmp = fadd double %x, 2.000000e+00
  ret double %addtmp
}

ready> foo(2);
Evaluated to 4.000000
```

为了让每一个函数都存在于其自己的模块内，我们需要在每一个我们新建的模块内加入之前定义的所有函数的声明：

```cpp
static std::unique_ptr<KaleidoscopeJIT> TheJIT;

...

Function *getFunction(std::string Name) {
  // First, see if the function has already been added to the current module.
  // 首先，看看这个函数是不是已经加入过这个模块了
  if (auto *F = TheModule->getFunction(Name))
    return F;

  // If not, check whether we can codegen the declaration from some existing
  // prototype.
  // 若否，则查找已经有的函数声明，生成其声明的中间代码并返回
  auto FI = FunctionProtos.find(Name);
  if (FI != FunctionProtos.end())
    return FI->second->codegen();

  // If no existing prototype exists, return null.
  // 如果这个函数的原型不存在，我们就返回 null
  return nullptr;
}

...

Value *CallExprAST::codegen() {
  // Look up the name in the global module table.
  // 在全局模块表中查找这个名字
  Function *CalleeF = getFunction(Callee);

...

Function *FunctionAST::codegen() {
  // Transfer ownership of the prototype to the FunctionProtos map, but keep a
  // reference to it for use below.
  // 将函数原型的所有权转移到 FunctionProtos map 中去，但保留一个引用以供后续使用
  auto &P = *Proto;
  FunctionProtos[Proto->getName()] = std::move(Proto);
  Function *TheFunction = getFunction(P.getName());
  if (!TheFunction)
    return nullptr;
``` 

为了实现它，我们首先加入一个新的全局变量 `FunctionProtos`, 它保存着每一个函数最后一次定义的原型。我们还加入了一个便利的函数 `getFunction` 来代替对 `TheModule->getFunction()` 的直接调用。我们的便利函数查找 `TheModule` 尝试获取本模块可能存在的该函数的原型中间代码，如果没有就尝试从原型表中查找原型并生成对应的中间代码。在 `CallExprAST::codegen()` 中我们直接将 `TheModule->getFunction()` 替换为 `getFunction()` 即可。在 `FunctionAST::codegen()` 中，我们首先更新全局的函数原型表 `FunctionProtos`, 然后再调用 `getFunction()` 来获取它。做完这些以后，我们就可以在当前模块中获取任何之前声明过的函数了 (即使它们可能并不在同一模块). 

我们还需要更新 `HandleDefinition` 与 `HandleExtern`: 

```cpp
static void HandleDefinition() {
  if (auto FnAST = ParseDefinition()) {
    if (auto *FnIR = FnAST->codegen()) {
      fprintf(stderr, "Read function definition:");
      FnIR->print(errs());
      fprintf(stderr, "\n");
      TheJIT->addModule(std::move(TheModule));
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
```

在 `HandleDefinition` 中，我们增加两行来将新定义的函数放入 JIT 中并打开一个新的全局模块。在 `HandleExtern`, 我们只需要增加一行来将函数的原型放入全局原型表中 `FunctionProtos`. 

改完后，让我们再次来试试我们的 REPL 吧：(匿名函数的输出被省略了)

```py
ready> def foo(x) x + 1;
ready> foo(2);
Evaluated to 3.000000

ready> def foo(x) x + 2;
ready> foo(2);
Evaluated to 4.000000
```

它好了！

即便只写了这么一点简单的代码，我们的 Kaleidoscope 已经拥有了强大的能力：

```py
ready> extern sin(x);
Read extern:
declare double @sin(double)

ready> extern cos(x);
Read extern:
declare double @cos(double)

ready> sin(1.0);
Read top-level expression:
define double @2() {
entry:
  ret double 0x3FEAED548F090CEE
}

Evaluated to 0.841471

ready> def foo(x) sin(x)*sin(x) + cos(x)*cos(x);
Read function definition:
define double @foo(double %x) {
entry:
  %calltmp = call double @sin(double %x)
  %multmp = fmul double %calltmp, %calltmp
  %calltmp2 = call double @cos(double %x)
  %multmp4 = fmul double %calltmp2, %calltmp2
  %addtmp = fadd double %multmp, %multmp4
  ret double %addtmp
}

ready> foo(4.0);
Read top-level expression:
define double @3() {
entry:
  %calltmp = call double @foo(double 4.000000e+00)
  ret double %calltmp
}

Evaluated to 1.000000
```

JIT 怎么认得 `sin` 跟 `cos` 呢？答案很简单，KaleidoscopeJIT 拥有简单直接的未定义符号解析规则：首先它依加入次序从新自旧去查找所有已经被加入了 JIT 的模块，如果找不到，那么它就会在 Kaleidoscope 的进程空间调用 `dlsym("sin")`. 因为 `sin` 在 JIT 的地址空间中已经被定义了，于是 JIT 在遇到 `sin` 时就会直接调用 `libm` 里面的 `sin` 函数。但在一些情况下它甚至能做更多事情：因为 `sin` 与 `cos` 是标准数学函数的名字，所以常数折叠会直接将函数调用优化为计算后的值，就像上面的 `sin(1.0)` 一样。

在后续的章节我们将看到如何改进符号解析规则以使用更多更有用的特性。从安全性 (限制 JIT 生成的代码能访问的符号集合) 到基于符号名称的代码生成，甚至可以做到懒惰计算 (lazy compilation). 

符号解析规则的一个直接的好处就是我们可以通过直接定义新的 C++ 函数来扩展我们的语言。举个例子，如果我们在我们的代码中加入：

```cpp
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
```

注意在 Windows 上我们还需要导出这个函数，因为动态符号加载器 (the dynamic symbol loader) 会使用 `GetProcAddress` 来查找符号。

现在通过使用形如 `extern putchard(x); putchard(120);` 的 Kaleidoscope 代码，我们可以往终端输出简单的文字了。(它将输出小写 `x`, 因为 120 是 `x` 的 ASCII 码). 类似的代码可以用于实现文件 IO，终端输入，以及其他各种能力。

本章到这里就结束了。现在，我们可以编译一门非图灵完备的语言，优化并即时编译用户输入的代码。下一步我们将[为语言加入控制流结构](ch-4.md), 同时探索一些有趣的 LLVM IR 相关问题。

## 4.5 全部代码

这里是本章例子的完整代码，包括 LLVM JIT 与优化器。若要构建这个例子，请使用：

```bash
# Compile
clang++ -g toy.cpp `llvm-config --cxxflags --ldflags --system-libs --libs core orcjit native` -O3 -o toy
# Run
./toy
```

如果您在 Linux 上编译这个程序，确保您加入了 `-rdynamic` 选项，这使得外部函数 (external function) 在运行时得以被正确解析。

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
#include <algorithm>
#include <cassert>
#include <cctype>
#include <cstdint>
#include <cstdio>
#include <cstdlib>
#include <map>
#include <memory>
#include <string>
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
  tok_number = -5
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

/// PrototypeAST - This class represents the "prototype" for a function,
/// which captures its name, and its argument names (thus implicitly the number
/// of arguments the function takes).
class PrototypeAST {
  std::string Name;
  std::vector<std::string> Args;

public:
  PrototypeAST(const std::string &Name, std::vector<std::string> Args)
      : Name(Name), Args(std::move(Args)) {}

  Function *codegen();
  const std::string &getName() const { return Name; }
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

/// primary
///   ::= identifierexpr
///   ::= numberexpr
///   ::= parenexpr
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
  }
}

/// binoprhs
///   ::= ('+' primary)*
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

    // Parse the primary expression after the binary operator.
    auto RHS = ParsePrimary();
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
///   ::= primary binoprhs
///
static std::unique_ptr<ExprAST> ParseExpression() {
  auto LHS = ParsePrimary();
  if (!LHS)
    return nullptr;

  return ParseBinOpRHS(0, std::move(LHS));
}

/// prototype
///   ::= id '(' id* ')'
static std::unique_ptr<PrototypeAST> ParsePrototype() {
  if (CurTok != tok_identifier)
    return LogErrorP("Expected function name in prototype");

  std::string FnName = IdentifierStr;
  getNextToken();

  if (CurTok != '(')
    return LogErrorP("Expected '(' in prototype");

  std::vector<std::string> ArgNames;
  while (getNextToken() == tok_identifier)
    ArgNames.push_back(IdentifierStr);
  if (CurTok != ')')
    return LogErrorP("Expected ')' in prototype");

  // success.
  getNextToken(); // eat ')'.

  return std::make_unique<PrototypeAST>(FnName, std::move(ArgNames));
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
static std::map<std::string, Value *> NamedValues;
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

Value *NumberExprAST::codegen() {
  return ConstantFP::get(*TheContext, APFloat(Val));
}

Value *VariableExprAST::codegen() {
  // Look this variable up in the function.
  Value *V = NamedValues[Name];
  if (!V)
    return LogErrorV("Unknown variable name");
  return V;
}

Value *BinaryExprAST::codegen() {
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
    return LogErrorV("invalid binary operator");
  }
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

  // Create a new basic block to start insertion into.
  BasicBlock *BB = BasicBlock::Create(*TheContext, "entry", TheFunction);
  Builder->SetInsertPoint(BB);

  // Record the function arguments in the NamedValues map.
  NamedValues.clear();
  for (auto &Arg : TheFunction->args())
    NamedValues[std::string(Arg.getName())] = &Arg;

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
  return nullptr;
}

//===----------------------------------------------------------------------===//
// Top-Level parsing and JIT Driver
//===----------------------------------------------------------------------===//

static void InitializeModuleAndPassManager() {
  // Open a new context and module.
  TheContext = std::make_unique<LLVMContext>();
  TheModule = std::make_unique<Module>("my cool jit", *TheContext);
  TheModule->setDataLayout(TheJIT->getDataLayout());

  // Create a new builder for the module.
  Builder = std::make_unique<IRBuilder<>>(*TheContext);

  // Create a new pass manager attached to it.
  TheFPM = std::make_unique<legacy::FunctionPassManager>(TheModule.get());

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