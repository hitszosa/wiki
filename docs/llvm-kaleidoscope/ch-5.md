# 第 5 章：扩展语言：控制流

## 5.1 前言

欢迎来到 [我的第一个基于 LLVM 的语言前端](ch-0.md) 教程第五章。前四章描述了一门简单语言的实现，生成 LLVM IR 并优化，随后即时编译到本地代码的过程。不幸的是，至少到目前为止，Kaleidoscope 还不堪大用：它没有控制流结构 (除了函数调用与返回). 这意味着你不能用它写出带条件分支的代码，这显著地影响了 Kaleidoscope 的威力。在本章，我们将扩展 Kaleidoscope，使其支持 if/then/else 表达式和一个简单的 for 循环。

## 5.2 If/Then/Else

实现 if/then/else 比较简单。要做的事情主要是把这个"新概念" 教给 lexer, parser, AST 与 LLVM 中间代码生成器。本节同时也是一个极好的展示如何 "养大" 一门语言，用新点子扩展它的例子。

在我们动手之前，先来看看讨论一下我们 **想要** 什么。我们想要的基本上就是这样：

```py
def fib(x)
  if x < 3 then
    1
  else
    fib(x-1)+fib(x-2);
```

在 Kaleidoscope，每一个结构都是一个表达式：没有语句。于是，就像其他表达式一样，if/the/else 表达式需要返回一个值。由于我们所采用的语言特性大部分都是函数式的，所以我们会让 if/then/else 表达式先求值其条件，随后视条件再求值 then 或 else 中的一个子表达式，这与 C 中的 "?:" 三元操作符表达式非常相似。

if/then/else 表达式的语义是先求值其条件，值 `0.0` 被认为是逻辑假，而其他任何值被认为是逻辑真。如果条件为真，那么第一个子表达式 (then 子表达式) 将会被求值; 如果条件为假，那么第二个子表达式 (else 子表达式将会被求值). 由于 Kaleidoscope 允许副作用，所以上面的行为需要被正式地确定。

现在我们知道我们想要什么了，我们来一步一步实现它。

### 5.2.1 词法扩展

对 lexer 的扩展很简单。首先我们加入几个新的词法单元 (token). 

```cpp
// control
tok_if = -6,
tok_then = -7,
tok_else = -8,
```

随后我们修改 lexer 使其能辨认这些关键字：

```cpp
...
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
return tok_identifier;
```

### 5.2.2 AST 扩展

为了表示新表达式，我们增加一个新的 AST 节点类型：

```cpp
/// IfExprAST - Expression class for if/then/else.
class IfExprAST : public ExprAST {
  std::unique_ptr<ExprAST> Cond, Then, Else;

public:
  IfExprAST(std::unique_ptr<ExprAST> Cond, std::unique_ptr<ExprAST> Then,
            std::unique_ptr<ExprAST> Else)
    : Cond(std::move(Cond)), Then(std::move(Then)), Else(std::move(Else)) {}

  Value *codegen() override;
};
```

这个 AST 节点仅仅只包含几个指向其子表达式的指针而已。

### 5.2.3 语法扩展

现在我们能从 lexer 里得到新的相关词元并且已经有其对应的 AST 了，那么我们的解析逻辑也是相对简单的了。首先我们定义一个新的解析函数：

```cpp
/// ifexpr ::= 'if' expression 'then' expression 'else' expression
static std::unique_ptr<ExprAST> ParseIfExpr() {
  getNextToken();  // eat the if.

  // condition.
  auto Cond = ParseExpression();
  if (!Cond)
    return nullptr;

  if (CurTok != tok_then)
    return LogError("expected then");
  getNextToken();  // eat the then

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
```

随后在 primary expression 里增加解析它的情况：

```cpp
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
  }
}
```

### 5.2.4 想要生成的 LLVM IR

现在我们能从代码构建出 AST 了，最后一步就是生成 LLVM 中间代码了。这将会是最有趣的一步，我们将介绍一些新的概念，而之前的几步的代码多多少少都已经在前面介绍过了。

我们先来看一个简单的例子。考虑以下代码：

```py
extern foo();
extern bar();
def baz(x) if x then foo() else bar();
```

如果你关闭优化，你得到的代码将会类似于：

```llvm
declare double @foo()

declare double @bar()

define double @baz(double %x) {
entry:
  %ifcond = fcmp one double %x, 0.000000e+00
  br i1 %ifcond, label %then, label %else

then:       ; preds = %entry
  %calltmp = call double @foo()
  br label %ifcont

else:       ; preds = %entry
  %calltmp1 = call double @bar()
  br label %ifcont

ifcont:     ; preds = %else, %then
  %iftmp = phi double [ %calltmp, %then ], [ %calltmp1, %else ]
  ret double %iftmp
}
```

为了"看见" 控制流图，我们可以使用 LLVM [opt](https://llvm.org/cmds/opt.html) 工具的一个小功能。如果你将 LLVM IR 放在一个文件 `t.ll` 里，然后执行 `llvm-as < t.ll | opt -analyze -view-cfg`, 那么你将会通过一个[弹出的窗口](https://llvm.org/docs/ProgrammersManual.html#viewing-graphs-while-debugging-code)看到下面这张图：

![CFG 例子](https://llvm.org/docs/_images/LangImpl05-cfg.png)

得到这张图的另一个方法是在编译器里或者是调试器内对 `Function *F` 调用 `F->viewCFG()` 或者 `F->viewCFGOnly()`. LLVM 具有许多优秀的可视化不同的图的功能。

谈回生成的代码，它比较简单： `entry` 块求值条件表达式 (在本例子里，指 `x`), 然后用指令 `fcmp one` 将其与 0.0 比较 (`one` 意味着 Ordered and Not Equal，有序且不等). 基于比较结果，这个代码跳转到 `then` 块或者是 `else` 块，块中包含了待求值的表达式。

当 `then`/`else` 块执行完毕之后，它们都跳转回 `ifcont` 块来执行 if/then/else 表达式之后的指令。在这个例子中这个指令就是从函数中返回。随后问题出现了：代码怎么知道哪一个表达式的值将要被返回？

这引出了一个重要的 SSA 运算：[phi 运算](http://en.wikipedia.org/wiki/Static_single_assignment_form). 如果你对 SSA 不太熟悉，这里有一篇很好的介绍：[维基百科：SSA form](http://en.wikipedia.org/wiki/Static_single_assignment_form). 简而言之，phi 运算能 "记住" 控制流从哪个块流过来。Phi 运算接受控制流可能经过的上一个块的变量，然后返回控制流真正经过的那个块里的变量。在本例子中，如果控制流从 `then` 块中流入，那么它的值就会是 `calltmp` 的值; 否则如果控制流从 `else` 块流入，那么它的值就会是 `calltmp1` 的值。

"天哪！", 你也许会想，"难道我那可怜的小小前端要开始生成 SSA Form 来使用 LLVM 了吗？!" 幸运的是，它并不需要，并且我们强烈建议你不要让前端生成 SSA form，除非你有非常非常充足的理由。实践上，对于一般的命令式编程语言而言，会产生 phi 运算的代码只有两类：

1. 包含了用户定义的变量的运算，如 `x = 1`, `x = x + 1`;
2. 值并不由 AST 直接给出 (而是依赖于运行时控制流的流向), 比如这个例子。

在本教程的[第七章：可变量](ch-7.md)里，我们将会深入地讨论第一种情况; 在现在，你并不需要构造 SSA 来应付第一种情况。对于情况二，你可以使用我们用来应付情况一的技术，或者如果方便的话，你可以直接手动插入 phi 节点。简单起见，我们选择后者，即我们手动插入 phi 节点。

好了，介绍跟总览已经结束了，开始写代码吧！

### 5.2.5 中间代码生成

我们来实现 `IfExprAST::codegen()`: 

```cpp
Value *IfExprAST::codegen() {
  Value *CondV = Cond->codegen();
  if (!CondV)
    return nullptr;

  // Convert condition to a bool by comparing non-equal to 0.0.
  // 将条件表达式的值与 0.0 比较以获得一个 bool 值 (i1)
  CondV = Builder.CreateFCmpONE(
      CondV, ConstantFP::get(TheContext, APFloat(0.0)), "ifcond");
```

第一块代码跟上面的代码类似，我们先生成条件表达式的代码，随后将其的值与 0.0 比较，获得一个类型为 1-bit 的整数 (bool) 的值。

```cpp
Function *TheFunction = Builder.GetInsertBlock()->getParent();

// Create blocks for the then and else cases.  Insert the 'then' block at the
// end of the function.
// 为 then 和 else 创建基本块，将 then 块插入到函数的结尾。
BasicBlock *ThenBB =
    BasicBlock::Create(TheContext, "then", TheFunction);
BasicBlock *ElseBB = BasicBlock::Create(TheContext, "else");
BasicBlock *MergeBB = BasicBlock::Create(TheContext, "ifcont");

Builder.CreateCondBr(CondV, ThenBB, ElseBB);
```

上面的代码构造了与 if/then/else 相关的基本块 (basic block), 它们跟上面的例子直接对应。第一行通过获取当前插入块的 "父母" 获得了当前正在构造的函数对象。随后其创建了三个基本块，注意其只将 TheFunction 传入了第一个块 (`then` 块) 的构造函数之中，这会使得构造函数将新构造出的基本块插入到该函数的最后去。最后我们便可以生成条件分支的代码。值得注意的是，创建新的基本块并不会隐式地改变 IRBuilder 插入指令的地点，所以新增的跳转指令仍然附加会在 `cond` 基本块的最后。同时我们发现，跳转指令可以直接以 ElseBB 作为参数，即使它还未被加入到哪一个函数之中 -- 事实上，这是 LLVM 支持前向引用 (forward reference) 的基本方法。

> 译者注：本文将 branch instruction 翻译为跳转指令

```cpp
// Emit then value.
// 生成 `then` 分支的代码
Builder.SetInsertPoint(ThenBB);

Value *ThenV = Then->codegen();
if (!ThenV)
  return nullptr;

Builder.CreateBr(MergeBB);
// Codegen of 'Then' can change the current block, update ThenBB for the PHI.
// 对 then 块内表达式的代码生成可能会改变当前基本块，所以我们为 phi 更新 ThenBB
ThenBB = Builder.GetInsertBlock();
```

当跳转指令被插入以后，我们将 IRBuilder 的插入点移动到新的基本块 `then` 的开头 -- 严格来讲，是其最后; 但因为 `then` 块现在是空的，所以它的最后就是它的开头 :). 

随后我们递归生成 `then` 表达式的中间代码，然后在 `then` 块的结尾加入一个无条件跳转指令到控制流合并的基本块。LLVM IR 一个有趣 (且非常重要) 的要求是：每一个基本块都被一句[控制流指令](https://llvm.org/docs/LangRef.html#terminators)(比如返回或者是跳转) 所[终结 (terminated)](https://llvm.org/docs/LangRef.html#functionstructure). 这意味着所有的控制流，包括 fall through，都必须在 LLVM IR 中被显式表达。如果你违反了这一规则，那么检验器 (verifier) 将会产生一个 error.

最后一行代码比较微妙，很容易漏掉，但十分重要。这里的主要问题是，在后面我们创建 phi 指令时，我们需要给它一并提供基本块与值。但在递归调用 codegen 的时候，`then` 子表达式可能会创建新的基本块，导致 ThenV 与目前的 ThenBB 不在同一个基本块内。因此我们在这里需要使用 Builder.GetInsertBlock() 来更新我们的 ThenBB，以确保我们能正确地将基本块与值匹配。

```cpp
// Emit else block.
// 生成 else 分支的代码
TheFunction->getBasicBlockList().push_back(ElseBB);
Builder.SetInsertPoint(ElseBB);

Value *ElseV = Else->codegen();
if (!ElseV)
  return nullptr;

Builder.CreateBr(MergeBB);
// codegen of 'Else' can change the current block, update ElseBB for the PHI.
// 同样也要更新一下 ElseBB 的值
ElseBB = Builder.GetInsertBlock();
```

生成 `else` 块的代码跟生成 `then` 块的代码基本相同，但是在最开始，我们需要将 `else` 基本块加入到函数里面去。(因为我们在构造这个块的时候没有将其加入). 现在我们可以开始处理 `merge` 块了。

```cpp
  // Emit merge block.
  // 生成 merge 块
  TheFunction->getBasicBlockList().push_back(MergeBB);
  Builder.SetInsertPoint(MergeBB);
  PHINode *PN =
    Builder.CreatePHI(Type::getDoubleTy(TheContext), 2, "iftmp");

  PN->addIncoming(ThenV, ThenBB);
  PN->addIncoming(ElseV, ElseBB);
  return PN;
}
```

前两行类似与之前：首先将 `merge` 块加入到函数内 (在这之前它是浮动的 (floating), 和 `else` 块一样), 然后改变 IRBuilder 的插入点。随后我们马上插入一个 phi 指令并设置 phi 指令的参数。

最终，codegen 函数返回构造出的 PHINode 作为对 if/then/else 表达式求值的结果。在我们这个例子里，它将会被解析顶层表达式的函数使用，后者最终会构造出返回指令。

综上，Kaleidoscope 现在可以依条件执行代码了。有了这个能力，Kaleidoscope 已经是一门相对完整的语言，可以计算许许多多的数值函数。下一步我们将会加入另一个有用的表达式，我们在其他非函数式语言里已经非常熟悉它了。

## 5.3 for 循环表达式

在学习了如何向语言中加入基本的控制流结构之后，我们就有了加入更高级功能的工具了。让我们来加入一些更激进的事物吧：一个 for 表达式：

```py
extern putchard(char);
def printstar(n)
  for i = 1, i < n, 1.0 in
    putchard(42);  # ascii 42 = '*'

# print 100 '*' characters
printstar(100);
```

这个表达式定义了一个新的变量 (`i`), 随后从一个起始值 (`1`) 开始迭代，当满足条件 (`i < n`) 时，把循环变量增加一个可选的增加值 (`1.0`). 如果增加值 (step value) 被忽略，那么它默认是 1.0. 当循环条件为真时，它执行其循环体内的表达式。由于我们没有什么特别的值值得返回，我们直接定义 for 表达式的返回值是 0.0. 当我们定义了可变量之后，它将会变得更加有用。

### 5.3.1 词法扩展

Lexer 的修改与 if/then/else 类似：

```cpp
... in enum Token ...
// control
tok_if = -6, tok_then = -7, tok_else = -8,
tok_for = -9, tok_in = -10

... in gettok ...
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
return tok_identifier;
```

### 5.3.2 AST 扩展

AST 扩展也是简单的，直接储存变量名与所属的各个子表达式即可。

```cpp
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
```

### 5.3.3 语法扩展

Parser 的代码也是相对统一的。唯一值得注意的点就是对可选的增加值的处理。在代码中，对有无增加值的检测是通过检查是否存在第二个逗号来实现的。如果没有增加值，它就向 AST 的增加值子表达式中传入 `null`.

```cpp
/// forexpr ::= 'for' identifier '=' expr ',' expr (',' expr)? 'in' expression
static std::unique_ptr<ExprAST> ParseForExpr() {
  getNextToken();  // eat the for.

  if (CurTok != tok_identifier)
    return LogError("expected identifier after for");

  std::string IdName = IdentifierStr;
  getNextToken();  // eat identifier.

  if (CurTok != '=')
    return LogError("expected '=' after for");
  getNextToken();  // eat '='.


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
  getNextToken();  // eat 'in'.

  auto Body = ParseExpression();
  if (!Body)
    return nullptr;

  return std::make_unique<ForExprAST>(IdName, std::move(Start),
                                       std::move(End), std::move(Step),
                                       std::move(Body));
}
```

最后我们在主表达式解析中加入其解析：

```cpp
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
  }
}
```

### 5.3.4 想要生成的 LLVM IR

现在我们来到值得注意的部分了：为 for 表达式生成 LLVM IR. 有了上面的例子，我们不难写出我们想要的 IR (下面的 IR 是未经优化的):

```llvm
declare double @putchard(double)

define double @printstar(double %n) {
entry:
  ; initial value = 1.0 (inlined into phi)
  br label %loop

loop:       ; preds = %loop, %entry
  %i = phi double [ 1.000000e+00, %entry ], [ %nextvar, %loop ]
  ; body
  %calltmp = call double @putchard(double 4.200000e+01)
  ; increment
  %nextvar = fadd double %i, 1.000000e+00

  ; termination test
  %cmptmp = fcmp ult double %i, %n
  %booltmp = uitofp i1 %cmptmp to double
  %loopcond = fcmp one double %booltmp, 0.000000e+00
  br i1 %loopcond, label %loop, label %afterloop

afterloop:      ; preds = %loop
  ; loop always returns 0.0
  ret double 0.000000e+00
}
```

`loop` 基本块包含了所有我们之前见过的结构：一个 phi 节点，几个表达式以及一些基本块。

最后让我们来看看它们是怎么样组合出来吧。

### 5.3.5 中间代码生成

中间代码生成的第一步比较简单，我们直接从求值初始值表达式开始：

```cpp
Value *ForExprAST::codegen() {
  // Emit the start code first, without 'variable' in scope.
  // 在没有 for 定义的变量在作用域内的前提下求值初始值表达式
  Value *StartVal = Start->codegen();
  if (!StartVal)
    return nullptr;
```

下一步是为循环体的开始创建一个新的基本块。在上小节的例子中，整个循环体在一个基本块里; 但实际上整个循环体可能包含多个基本块 (比如循环体包含了 if/then/else 或者 for/in 表达式的情况). 

```cpp
// Make the new basic block for the loop header, inserting after current
// block.
// 将新基本块作为循环表达式的头部，插入到当前基本块之后
Function *TheFunction = Builder.GetInsertBlock()->getParent();
BasicBlock *PreheaderBB = Builder.GetInsertBlock();
BasicBlock *LoopBB =
    BasicBlock::Create(TheContext, "loop", TheFunction);

// Insert an explicit fall through from the current block to the LoopBB.
// 插入一个显式的 fall through 以便掉到下一个基本块去
Builder.CreateBr(LoopBB);
```

这些代码类似于我们为 if/then/else 所写的代码。因为我们需要创建 phi 节点，我们必须记得那个 fall through 进入循环体的基本块。一旦有了那个块，我们就可以创建真正的循环头部，并且插入一个无条件跳转以进入循环体。

> 译者注：其实就是把每一次的循环变量用 phi 定义出来。循环变量的值可能来源于两处：初始值与上一次的值，后一种情况 phi 节点的参数将会包含一个在之后定义的变量。在这里我们保存开头的块是为了在 phi 里加入初始值所在的块。

```cpp
// Start insertion in LoopBB.
// 开始往循环体内加入指令
Builder.SetInsertPoint(LoopBB);

// Start the PHI node with an entry for Start.
// 将开始的 StartVal 加入到 phi 节点里 
PHINode *Variable = Builder.CreatePHI(Type::getDoubleTy(TheContext),
                                      2, VarName.c_str());
Variable->addIncoming(StartVal, PreheaderBB);
```

现在我们开始真正地生成循环体内的代码。我们首先在循环体头部为循环变量插入 phi 节点。由于我们已经知道我们将会从初始值得到循环变量的一个值，我们先将它加入 phi 的参数。我们的 phi 节点最终会得到它的第二个参数，但我们现在还不能设置它 (因为这个变量现在还不存在！). 

```cpp
// Within the loop, the variable is defined equal to the PHI node.  If it
// shadows an existing variable, we have to restore it, so save it now.
// 在循环体内部，循环变量现在是被 phi 指令定义出来的那个
// 如果我们的循环变量遮盖了一个已经存在的变量，那么我们要先把旧的变量的值保存起来
Value *OldVal = NamedValues[VarName];
NamedValues[VarName] = Variable;

// Emit the body of the loop.  This, like any other expr, can change the
// current BB.  Note that we ignore the value computed by the body, but don't
// allow an error.
// 生成循环体的代码。同理这可能改变当前基本块。
// 即使我们会忽略循环体的求值结果，但我们不会允许循环体内出现错误
if (!Body->codegen())
  return nullptr;
```

现在代码变得有趣起来了。我们的 for 循环引入了新的变量，这意味着它有可能跟某个外部变量重名。我们可以选择在重名的时候直接报错 -- 这将简化我们的处理 -- 但我们还是选择使用遮蔽 (shadowing)。为了处理这种情况，在我们开始生成循环体的代码之前，我们首先要检查可能冲突的变量并事先加以保存。在循环体执行完毕之后，我们还要恢复重名变量原先的值。

当循环变量被加载进符号表之后，我们便可以开始递归生成循环体的中间代码。这使得循环体内部可以使用循环变量：任何对该名字的引用都可以自然地在符号表内找到。

```cpp
// Emit the step value.
// 生成增加值表达式的代码
Value *StepVal = nullptr;
if (Step) {
  StepVal = Step->codegen();
  if (!StepVal)
    return nullptr;
} else {
  // If not specified, use 1.0.
  // 如果增加值没有被指定，那么使用 1.0
  StepVal = ConstantFP::get(TheContext, APFloat(1.0));
}

Value *NextVar = Builder.CreateFAdd(Variable, StepVal, "nextvar");
```

当循环体被生成了之后，我们就要计算循环变量的下一次的值。我们先求值增加值，然后再将其加到循环变量上以创建一个新的变量。接下来，`NextVar` 将会变成下一次循环中循环变量的值。

```cpp
// Compute the end condition.
// 生成循环条件表达式
Value *EndCond = End->codegen();
if (!EndCond)
  return nullptr;

// Convert condition to a bool by comparing non-equal to 0.0.
// 将循环条件表达式的值与 0.0 比较以得到一个 bool
EndCond = Builder.CreateFCmpONE(
    EndCond, ConstantFP::get(TheContext, APFloat(0.0)), "loopcond");
```

最后，我们求值循环条件来判断是否应该终止循环。这些代码与 if/then/else 中的判断是完全相同的。

```cpp
// Create the "after loop" block and insert it.
// 创建循环后基本块并将其插入
BasicBlock *LoopEndBB = Builder.GetInsertBlock();
BasicBlock *AfterBB =
    BasicBlock::Create(TheContext, "afterloop", TheFunction);

// Insert the conditional branch into the end of LoopEndBB.
// 在循环结束的最后插入跳转指令
Builder.CreateCondBr(EndCond, LoopBB, AfterBB);

// Any new code will be inserted in AfterBB.
// 让后面的代码都插入到循环之后的基本块之中
Builder.SetInsertPoint(AfterBB);
```

当循环体的中间代码被完全生成了之后，我们只需要结束其控制流就可以了。上面的代码记住了循环体最后一个块 (为了接下来的 phi 节点), 然后创建了循环的出口的基本块 (`afterloop`). 基于循环条件的值，其使用跳转指令来使控制流跳转到合适的地方。我们创建循环出口的基本块是为了做那个跳转，接下来插入的指令应该都在这个块内，所以我们需要把指令的插入点设置在此块中。

```cpp
  // Add a new entry to the PHI node for the backedge.
  // 为 phi 节点增加一对新的参数 (后向边)
  Variable->addIncoming(NextVar, LoopEndBB);

  // Restore the unshadowed variable.
  // 恢复被遮盖的变量
  if (OldVal)
    NamedValues[VarName] = OldVal;
  else
    NamedValues.erase(VarName);

  // for expr always returns 0.0.
  // 对于 for 循环，其求值结果永远是 0.0
  return Constant::getNullValue(Type::getDoubleTy(TheContext));
}
```

最后的代码进行了一些清理工作。我们首先以 `NextVar` 的值来更新循环变量 -- 将其插入到定义循环变量的 phi 节点中去。随后我们从符号表中删除了循环变量 (或者是恢复了被遮蔽的旧变量的值). 最后，我们返回 for 表达式的求值结果 -- 0.0.

就此，我们结束本章教程。在本章，我们为 Kaleidoscope 加入了两种新的控制流结构，并且在实现过程中探索了 LLVM IR 一些应为前端实现者所知的重要方面。在下一章，我们将会大胆一些，将[自定义运算符](ch-6.md)加入我们弱小可怜又无助的小 Kalei.

## 5.4 全部代码

下面是本章例子包含的完整代码，拥有 if/then/else 与 for 表达式。使用下面的命令构建这个例子：

```bash
# Compile
clang++ -g toy.cpp `llvm-config --cxxflags --ldflags --system-libs --libs core orcjit native` -O3 -o toy
# Run
./toy
```

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
  tok_number = -5,

  // control
  tok_if = -6,
  tok_then = -7,
  tok_else = -8,
  tok_for = -9,
  tok_in = -10
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

/// primary
///   ::= identifierexpr
///   ::= numberexpr
///   ::= parenexpr
///   ::= ifexpr
///   ::= forexpr
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
//   ...
//   start = startexpr
//   goto loop
// loop:
//   variable = phi [start, loopheader], [nextvariable, loopend]
//   ...
//   bodyexpr
//   ...
// loopend:
//   step = stepexpr
//   nextvariable = variable + step
//   endcond = endexpr
//   br endcond, loop, endloop
// outloop:
Value *ForExprAST::codegen() {
  // Emit the start code first, without 'variable' in scope.
  Value *StartVal = Start->codegen();
  if (!StartVal)
    return nullptr;

  // Make the new basic block for the loop header, inserting after current
  // block.
  Function *TheFunction = Builder->GetInsertBlock()->getParent();
  BasicBlock *PreheaderBB = Builder->GetInsertBlock();
  BasicBlock *LoopBB = BasicBlock::Create(*TheContext, "loop", TheFunction);

  // Insert an explicit fall through from the current block to the LoopBB.
  Builder->CreateBr(LoopBB);

  // Start insertion in LoopBB.
  Builder->SetInsertPoint(LoopBB);

  // Start the PHI node with an entry for Start.
  PHINode *Variable =
      Builder->CreatePHI(Type::getDoubleTy(*TheContext), 2, VarName);
  Variable->addIncoming(StartVal, PreheaderBB);

  // Within the loop, the variable is defined equal to the PHI node.  If it
  // shadows an existing variable, we have to restore it, so save it now.
  Value *OldVal = NamedValues[VarName];
  NamedValues[VarName] = Variable;

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

  Value *NextVar = Builder->CreateFAdd(Variable, StepVal, "nextvar");

  // Compute the end condition.
  Value *EndCond = End->codegen();
  if (!EndCond)
    return nullptr;

  // Convert condition to a bool by comparing non-equal to 0.0.
  EndCond = Builder->CreateFCmpONE(
      EndCond, ConstantFP::get(*TheContext, APFloat(0.0)), "loopcond");

  // Create the "after loop" block and insert it.
  BasicBlock *LoopEndBB = Builder->GetInsertBlock();
  BasicBlock *AfterBB =
      BasicBlock::Create(*TheContext, "afterloop", TheFunction);

  // Insert the conditional branch into the end of LoopEndBB.
  Builder->CreateCondBr(EndCond, LoopBB, AfterBB);

  // Any new code will be inserted in AfterBB.
  Builder->SetInsertPoint(AfterBB);

  // Add a new entry to the PHI node for the backedge.
  Variable->addIncoming(NextVar, LoopEndBB);

  // Restore the unshadowed variable.
  if (OldVal)
    NamedValues[VarName] = OldVal;
  else
    NamedValues.erase(VarName);

  // for expr always returns 0.0.
  return Constant::getNullValue(Type::getDoubleTy(*TheContext));
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
  // Open a new module.
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
