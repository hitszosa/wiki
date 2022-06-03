# 在讲 VSC 之前...

## 一点点 Shell 与程序交互 (PowerShell)

每一个正常的、给普通人用的操作系统都会有至少一个跟用户交互的界面。在 Windows 上，最常用的界面是 "文件管理器", 第二常用的界面是 "PowerShell". 在 Linux 上最常用的应该是 "Shell" (Bash, Zsh...), 然后是各种 DE (Desktop Environment, 相当于 Windows 的文件管理器). 而在这个界面里，最重要的功能便是 **执行其他程序**。

比如说在 Windows 上，我们双击一个文件，它就 "打开" 了 -- 这个打开是什么意思呢？对于普通的文件而言，其实就是 Windows 根据这个文件的后缀名，找到了它的 "默认打开方式", 然后把路径 *传给* 那个程序并执行。然后那个程序要怎么样料理这个路径就是它自己的事了。

今天我们不过多展开 GUI 下的交互方式，我们来说说怎么样在命令行交互界面里跟其他程序 "交互". 为什么要讲命令行界面呢？因为命令行界面是最适合 **程序来执行程序** 的，而图形化界面是方便我们人类来执行程序。

打开 PowerShell, 我们首先随便输入一行命令

## 环境变量

# VSC

## 界面与概览

首先，我要告诉你 VSC 里各种东西的名字，至少是各种 "可以按的东西" 的名字，否则进一步的交流就是无根之木。

要想快速显示/隐藏某个组件，可以在菜单栏的 View(查看) > Appearance(外观) 关掉它 (或者打开它)。

### 活动栏/侧边栏 (Activity Bar/Side Bar)

首先是左边的一小条柱子，活动栏。活动栏上的图标就是一个个活动，或者说，一些功能集合。点击任一一个活动按钮，弹出来的那块就叫侧边栏。一般来讲侧边栏跟活动栏因为经常一起出现，所以一般叫法也是乱叫。活动栏跟侧边栏可以放在右边，你只需要在侧边栏的顶部右键出一个菜单就可以调。活动栏里的活动并不一定是这么多，有些扩展可以往活动栏里添加新的活动 (比如 Remote SSH).

因为我已经记熟了展开每一个活动的快捷键了，所以我直接关掉了活动栏以节省空间 -- 很多时候如果你一屏幕切开两三份，就会觉得活动栏非常地碍事了。

#### 资源管理器 (Explorer)

什么是"资源"? 实际上资源就是你写出来，打开来的东西。资源管理器里会有几栏，比如会有你打开的编辑器，你当前打开的文件夹的文件树，跟你代码的大纲。这些"栏"是可以被关掉的，同样是在"侧边栏的顶部右键出来的菜单"里调。文件树默认情况下会把连续的单个目录折叠，你可以在设置里选择不要折叠; 对于不想要看到的文件夹 (比如一些缓存文件夹，构建文件夹), 也可以在设置里选择不要显示。如果你打开的文件夹是一个 Git 仓库，那么文件树的每一项右边还会显示出或绿或黄的点/U/M. 绿色的 U 表示这个文件还未加入到 git 中，是 untrack 的; 黄色的 M 表示这个文件目前跟 git 里的版本不一致，是 modified 的; 点出现在文件夹里，表示这个文件夹里有 U/M 的文件。有时候，除了 UM，还会出现一些红黄的数字，这些代表 VSC 检测到的文件里的问题，你可以通过面板里的问题来查看，数字代表问题的数量，黄色是 warning，红色是 error. 如果你需要快速在文件夹里找到一个特定名字的文件，你可以点一下文件树，然后直接打字，就能定位到文件了。

我的话主要是关掉了资源管理器里的 "Opened editor" 栏，因为我平时根本不用它 -- 打开了哪些文件我还不知道吗？

#### 搜索 (Search)

在侧边栏的搜索范围是当前打开的整个文件夹。点一下搜索框右下的三个点，可以弹出几个框出来让你选择 "包括" 和 "排除" 的文件。我用这个搜索用得倒是比较少，一般用不到。

#### SCM (版本管理)

这里是 VSC 跟 Git 集成的地方，可以很轻松地用 "点点点" 的功能操作 Git 仓库。主界面会展示目前未加入暂存区 (未 add) 的文件 (Changes 栏) 跟已经在暂存区 (已 add 但未 commit) 的文件 (Staged 栏)。点一下右上角三个点有更多实用操作，一般我常用的就是 pull 跟 undo last commit. 其他命令我还是习惯命令行操作。点击一下文件会进入 diff 模式，会打开两个 editor，一边是 git 里的文件，一边是你目前的文件，你可以方便地看到你到底做了什么修改。当执行 pull 操作需要合并冲突的时候，也会打开 diff 模式，这时候你可以手动删除/增加当前文件来合并冲突。

#### 运行 (Run)

这里是 VSC 跟 GDB/LLDB 等调试器集成的地方，本质上就是配不同的 task，当你点击绿色三角的时候会运行一些特定的 task，通过不同的配置便可以做成泛用的。鉴于我一般习惯直接开 `gdb -tui`，所以我也不知道这里有啥小技巧。

#### 扩展 (Extension)

这里是装扩展跟卸扩展的地方。如果你没看到扩展的图标，那你可以试着把侧边栏拉大一点。点击扩展可以看到扩展的详情，特别是可以看到扩展新加入的配置项跟 Command -- 在配快捷键的时候特别有用。所谓 Command，相当于这个插件给你封装好的一个操作，类似于一个 "函数". VSC 里绝大部分操作都只是执行一个个 Command -- 比如你按上下箭头，光标会上下动，实际上是因为 VSC 默认把上箭头绑定到 editor 组件的 "向上移动光标" 这个 Command 上。Command 相当于在操作层面 VSC 给你暴露的 "基本语句"("原语")。

在侧边栏的扩展里的右上可以看到扩展加载所用时间 -- 大部分情况下，不必要的扩展并不会在启动时即被加载，所以大部分扩展根本没有加载时间。VSC 扩展的按需加载做得挺不错，一般可以放心装上五六十个不是问题。

`@` 开头的扩展过滤器可以获取特定类别的扩展，或者是排序 (`@sort:install`, 按安装数量排序)。详见文档：[Extensions View Filters](https://code.visualstudio.com/docs/editor/extension-marketplace#_extensions-view-filters)

也可以通过命令行或是 `Ctrl+Shift+P Install from VISX` 装下载好的扩展：

```
code --install-extension myextension.vsix
```

#### 账户 (Account)

这个主要是登录来同步配置文件的。VSC 可以把你的配置文件 (settings.json) 跟你装了啥扩展，配了啥快捷键等信息保存到特定服务器去。一般我们会选择用 Github 账号保存。

#### 设置 (Setting)

这里主要是一些设置面板的快速启动选单，自己探索一下就好。

### 面板 (Panel)

面板一般拿来放一些可以打字交互/文本输出的东西，比如终端，代码里检测出的问题，vsc 的 log，跟 debug 时的交互。面板可以放在下面也可以放在左右。一般我习惯 `Ctrl+\`` 打开面板，虽然这个快捷键的行为是打开终端。

#### 终端 (Terminal)

所谓终端，其实就是调用特定的 Shell，然后把输出输出处理一下给你看，用起来跟正常的 Terminal Emulator 差不多。有时候一些 Shell 的快捷键会跟 VSC 的冲突，可以在设置里选择哪些快捷键需要让 VSC 处理，哪些要直接原样发送到终端。

#### 问题 (Problem)

这里是放某些程序检测出的你的代码的问题的地方。VSC 并没有检测绝大部分语言的问题的能力，它只是运行特定的 task，这些 task 运行代码检查工具 (比如 linter 或是编译器), 然后获得它们的输出，随后使用 [problem matcher](https://code.visualstudio.com/Docs/editor/tasks#_defining-a-problem-matcher) 匹配出特定的输出信息，然后将其展示为 error/warning. 如果你想为你的项目添加一些检查工具，那么只需要在工作区的 `tasks.json` 里定义一些新的后台任务就可以了。

#### 输出 (Output)

这里存放了 VSC 里 **几乎所有东西的日志 (log)** . 不管是 VSC 自己的，还是 Remote 的，还是特定扩展的，只要是调试信息就会往这里丢。这个地方非常非常非常重要，如果你用 VSC 的过程中出现了什么奇奇怪怪的问题，特别是发现某些扩展没有按预期工作的时，记得检查一下这里的输出 (然后拿日志去求助/google)。

#### 调试终端 (Debug Console)

我没用过，所以略。

### 菜单栏 (Menu Bar)

你多点点就知道各个选项是干嘛的了。一般都会选择隐藏菜单栏，因为在隐藏状态下点一下 alt 它就又出来了，所以平时还是藏着好看。

### 状态栏

状态栏顾名思义就是拿来放当前 VSC 的一些状态的栏子。如果有不想要的部分 (说的就是你，feedback!), 可以右键状态栏然后关掉。

## 编辑与编辑器

> 基本使用方法：[VSC 文档](https://code.visualstudio.com/docs/editor/codebasics)

### 界面解释

VSC 的编辑器有很多花花绿绿的功能，我关掉了略缩图跟标签页显示。主要是略缩图太占地方了，而且对小文件效果基本上没有。之前我还把面包屑导航也关掉了，但是后来看一些项目的代码的时候又发现特殊时候它其实很有用 (比如一个函数超过了一页长，没有面包屑你看的时候就很难抓住上下文), 而且按 `Ctrl+Shift+.` 的话可以快速通过面包屑跳转代码里的部分，所以又开回来了。对于标签页显示，我觉得我每次切标签页都是在乱按，根本就没看过标签页的标题，那不如关掉。关掉以后切标签页可以使用 `Ctrl+Tab` 快速对切，用 `Ctrl+P` 打名字精准切换，分组可以使用 `Ctrl+\` 来分。
     
一个 VSC 的实例就叫一个窗口 (Window), 然后里面可以分组 (Group), 每个组里里面可以有多个标签页 (Tab), 不同的标签页就是不同的编辑器 (Editor). 鼠标拉住标签页可以拖出来，拖到不同的组去，或者拖出一个新组来。

### 滚动条，行号与色块

> Gutter indicators (修改状态指示条？)

滚动条上会显示当前视图长度，当前光标位置，选中的单词的位置

### SCM 相关操作

#### 版本管理是啥

#### Diff

#### Timeline

#### GitHub 集成

文档：[Working with GitHub](https://code.visualstudio.com/docs/editor/github)

### 多光标

光标，就是字符的插入点; 多光标，就是在多个地方同时插入相同的字符。跟多光标平行的还有多选，也就是选择多块文本。它的用处有很多，比如说，批量修改某个变量的名字，或者是给一整列东西加上一个什么字符。按住 `alt` 再用鼠标点就可以点出多个光标出来。使用 `Ctrl+Alt+↑/↓` 可以直接往上/往下 "扩展" 光标。选中一块文字之后按 `Ctrl+Shift+L` 可以选中文件中所有相同的文本，而按 `Ctrl+D` 可以一个一个选定相同的。

更详细的多光标指南可以看：<https://github.com/tjx666/blog/blob/main/src/VSCode%E5%8F%88%E9%85%B7%E5%8F%88%E5%AE%9E%E7%94%A8%E7%9A%84%E5%A4%9A%E5%85%89%E6%A0%87%E7%BC%96%E8%BE%91.md>

### 行操作

按 `Alt+↑/↓` 可以直接交换上下行，按 `Shift+Alt+↑/↓` 可以把行往上/往下复制一份。我经常干的事就是按住 `Shift` 然后上下移动箭头选中一系列行，然后按 `Shift+Alt+↑/↓` 把选中的复制一份，这时 VSC 会保持你的选中，你就可以把原来的那块用 `Alt+↑/↓` 移动到想要的地方去了，相当于 "用鼠标选上一块，Ctrl+C，滚动光标，Ctrl+V" 操作。

在行内任何位置可以按 `Ctrl+Enter` 直接往下新起一行。当你光标没有选中任何文本，直接按 `Ctrl+X/C/V` 时，就会直接 剪切/复制/粘贴 当前整个行。行内直接按 `Ctrl+/` 就可以直接注释掉当前行，如果你用 `Shift+↑/↓` 选了多行的话那么多行就会一起被注释掉。

> btw，都讲了行操作了，那就顺带讲讲块操作：如果你选择了一块文本，那么直接按 `"'\`([{` 之类的可以成组的按键的话，就可以直接用这个把选中的文本包起来。

### CAS + 箭头

> 可以直接打开 Shortcut 界面查

| 功能键           | 左右                  | 上下                      |
|------------------|---------------------|-------------------------|
| `ctrl`           | 按词移动              | 屏幕移动                  |
| `alt`            | 上一个/下一个光标位置 | 行互换                    |
| `shift`          | 左右选择              | 上下选择                  |
| `ctrl+alt`       | 编辑器移动到上下组    | 多光标延伸                |
| `ctrl+shift`     | 词移动 + 选中         | 无特殊 (上下选择/`shift`) |
| `alt+shift`      | 块选中 (按折叠的范围) | 重复行                    |
| `ctrl+alt+shift` | 无特殊 (`shift`)      | 无特殊 (`ctrl+alt`)       |

### 查找与搜索

#### 正则表达式

#### 新的搜索编辑器

文档：[Search Editor](https://code.visualstudio.com/docs/editor/codebasics#_search-editor)

### 自定义折叠

文档：[Folding](https://code.visualstudio.com/docs/editor/codebasics#_folding)

| Language | Start                            | End                                   |
|----------|----------------------------------|---------------------------------------|
| C/C++    | `#pragma region`                 | `#pragma endregion`                   |
| Java     | `//#region` or `//<editor-fold>` | `// #endregion` or `//</editor-fold>` |
| Python   | `#region` or `# region`          | `#endregion` or `# endregion`         |
| Markdown | `<!-- #region -->`               | `<!-- #endregion -->`                 |

### 神奇的 LSP ~~老涩批~~

LSP，全称 Language Server Protocol，语言服务器协议，是一个巨硬创立的，用于提供编程语言 "理解" 相关服务的，采用 [JSON-RPC](https://zh.wikipedia.org/wiki/JSON-RPC) 的，协议。在 LSP 里，一般有两个程序 (进程) 在运行，一个是语言服务器，另一个是 "语言客户端", 一般就是啥开发工具什么的。为了方便说明，下面以 clangd 跟 VSC 举例。

当你在用 VSC 写 C++ 的时候，VSC 把你目前编辑着的文件跟光标位置打包，发给在后台运行的 clangd. clangd 拿到了之后，分析整个文件，比如说根据 `#include` 去找到头文件再分析新文件，然后根据得到的光标位置提供一定的建议，再发回给 VSC. VSC 得到了建议，于是就显示在你看到的自动补全 (autocomplete) 的框里。随着你编辑进行，这些过程时刻发生，你就得到了基于语义的补全。

当你按着 `Ctrl` 去点某个函数名时，VSC 也同样把函数名发给 clangd, clangd 分析文件找到对应的函数的实现的位置，然后把那个位置发给 VSC, VSC 再跳转到那个位置，这就实现了 "跳转到定义".

LSP 将 "理解语言" 与 "提供编辑界面" 这两者切开了，这不得不说是一个全新的进步。传统的 IDE，比如 JetBrain 家的，做 IDE，理解代码提供补全之类的操作，都是跟 IDE 高度绑定的。而有了 LSP，我们就可以把代码理解的部分单独抽开来，这样不管你的编辑器是 vim，是 emacs，是 atom 还是 VSC，只要你实现了 LSP，你就可以借助对应语言的 LS 实现基于语义的补全，这大大解除了开发工具和语言之间的绑定。

目前，绝大部分语言都有了对应的 LS，比如 C/C++ 有 clangd, Rust 有 rls, Racket 有 racket-langserver 等等; 新兴的语言更是重视对 LS 的开发 -- 只要实现了一个良好的 LS，语言就马上有了一个非常强大的 "IDE", 相当于白嫖到一大堆辅助开发工具，何乐而不为呢。

当然，LSP 也有其缺点。根据 ice1000 佬的回答，"LSP 不能很好实现跨语言理解", 比如说在 Kotlin 代码里提供对 Java 库的支持。不过瑕不掩瑜，作为非常成功的一门技术，LSP 对 VSC 的发展起到了很大的支持作用。

> LSP 主页：https://microsoft.github.io/language-server-protocol/

---

> 在没有 LSP 的年代，通用的提供补全的工具是 ctags. 这个工具读取 C/C++ 文件，然后分析每一个标识符的语法属性，然后记录到一个文件里。Vim 等编辑器通过定时运行 ctags 来获取新的跨文件的标识符补全，然后文件内的标识符补全就靠词法补。不得不感叹时代变了啊。

### 代码理解

> 词义分别：IntelliSense (Suggestion), Auto Complete, Rich Language Editing

> **IntelliSense** is a general term for various code editing features including: *code completion, parameter info, quick info, and member lists*. IntelliSense features are sometimes called by other names such as "code completion", "content assist", and "code hinting."

> CamelCase filtering: "cra" -> "createApplication"

> inlay hints: 参数提示，Java 里常见的那种

> Code Action = Quick Fix + Refactoring

对开发工具来讲，有三种情况是需要其 "理解" 代码的：自动补全，定义跳转，重命名。在 VSC 里，跳转定义一般是 Ctrl + 点击，重命名是 F2. 如果按了但是没有反应，说明 VSC 不能理解这个语言。大部分 VSC 里的对某某语言的扩展，都只是给 VSC 加入了语法高亮规则，然后去运行特定 LS 以提供补全/跳转/重命名，有可能还有一些 linter 之类的一套玩意而已，本身并没有去 "理解" 代码。在没有 LS 的情况下，VSC 只能提供一些简单的基于词法的补全 (也就是说只有你前面打过的词才会变成后面的补全结果). 

有些 LS 需要特殊的配置文件/启动参数以启用特殊的功能，比如 clangd 应该就可以接受几个参数指定头文件目录跟代码目录。对这些参数的指定往往是该语言的扩展的设置项的很大一部分。

有些 LS 还会提供更进一步的功能，比如 inlay hints, 函数上面的小栏子 (一般写点 reference 次数之类的，C# 有), 以及重构和快速修复 (Quick Fix, 就是那个小灯泡)。

### Snippet 简介

Snippet，又称用户自定义片段，简单来说就是你填一个模板，设定好它的触发词。当你在代码里输入这个触发词时，这个模板就会出现在自动补全的提示结果里，然后按 Tab 就能展开。有些语言的扩展会自带一些 (基本上不顺手的) Snippet. (比如 C/C++ 扩展的 `namespace/struct/class`). 在后面会介绍如何配置自己的 Snippet.

## 工作区与设置

VSC 是基于 Electron 的，四舍五入一个套壳浏览器。所以 VSC 几乎所有的配置文件都采用了 [JSON](https://zh.wikipedia.org/wiki/JSON) 格式。打开 VSC 的设置，你就能看到 JSON...

并不会。

VSC 很贴心地给你套了一层 GUI，虽然底层文件保存的还是 JSON，但是至少这个 GUI 非常便于你去看一下到底有啥选项。(建议你全部看完过一趟，装了中文语言包的话是有中文说明的). 点一下右上的图标就可以进入 JSON 界面。VSC 的配置文件里都有非常好使的补全，写设置的时候写 JSON 很方便 (当然搜设置或者看设置意义的时候还是那个 GUI 方便一点.)

### JSON

JSON 的格式也非常简单。举个例子：

```jsonc
{
    "a-null-value": null,
    "a-number-value": 123,
    "another-number-value": 45.234,
    "a-string-value": "hello world",
    "a-boolean-value": true,
    "a-list-value": ["a", 233, true],
    "a-object-value": {
        "a-string-value": "hello world",
        "a-number-value": 123,
        "a-boolean-value": true
    },

    "a-nested-obj": {
        "obj-in-obj": {
            "arr-in-obj": ["a", 233, true]
        },
        "obj-id": 2334
    }

    // VSC 的 JSONc 格式允许的注释
}
```

JSON 里的"值"有六种：空值 (`null`), 数字 (直接写，可以整数可以浮点), 字符串 (双引号串起来), 布尔值 (`true`/`false`), 数组 (用`[]`框起来的逗号分隔的一堆值), 对象 (`{}`框起来的逗号分隔的一堆键值对). 键值对是形如：`"键": 值` 的一串玩意。数组跟对象里的值都可以还是数组跟字典。原生 JSON 不允许数组/字典最后一个元素的逗号，也不允许注释; 但是 VSC 使用一种改进的 JSON(称为 JSONc, JSON with comments), 允许了前二者。

### 我的 (部分) 通用设置

下面是我的设置的一部分，聚焦 VSC 本身的设置，基本上都有注释，你可以看着抄一点。语言相关的设置等到语言相关的章节再讲。

```jsonp
{   
    // ===================== VSCode 自身的设置 =====================

    // 外观设置 (主要是关掉一些看不顺眼的 UI)
    // 设置窗口标题上的字，参考：https://code.visualstudio.com/docs/getstarted/settings 里搜 window.title
    "window.title": "${dirty}${activeEditorShort}${separator}${rootName}",
    // 显示状态栏
    "workbench.statusBar.visible": true,
    // 关闭略缩图
    "editor.minimap.enabled": false,
    // 只有按了 alt 才会暂时显示菜单栏
    "window.menuBarVisibility": "toggle",
    // 关掉活动栏
    "workbench.activityBar.visible": false,
    // 关闭资源管理器的文件夹折叠，也就是说对文件夹 a 里的单个文件夹 b 不会折成 a/b 这样一起
    "explorer.compactFolders": false,
    // 我也忘记是啥了
    "workbench.editor.enablePreview": false,

    // 关掉烦死人的 "信任文件夹" 提示！
    "security.workspace.trust.enabled": false,
    "security.workspace.trust.emptyWindow": false,

    // 关掉对易混淆的字符 (比如全角/半角空格) 的高亮显示
    // 一般我肉眼就能扫出来
    "editor.unicodeHighlight.ambiguousCharacters": false,

    // 编辑器的外观设定
    // 首先是一个好看的主题，直接扩展里搜就有了
    "workbench.colorTheme": "Nord",
    // 然后是字体设置
    // 可以设置多个作为 fallback，带空格的要用单引号括起来
    "editor.fontFamily": "'Cascadia Code', 'Sarasa Mono CL'",
    // 开启连字字体
    "editor.fontLigatures": true,
    // 字体大小设置
    "editor.fontSize": 16,
    // 最后是光标设置
    // 光标闪烁的动画，设成 smooth 是真的很丝滑
    "editor.cursorBlinking": "smooth",
    // 光标移动动画，当你用鼠标点击其他地方的时候就可以看到一个光标飘过去的动画
    "editor.cursorSmoothCaretAnimation": true,

    // 杂项设定
    // 渲染控制字体 (也就是 ASCII 前面的那些字符)
    "editor.renderControlCharacters": true,
    // 打开新窗口时默认开启一个新文件
    "workbench.startupEditor": "newUntitledFile",
    // 忽略扩展推荐：我自然知道该装什么扩展
    "extensions.ignoreRecommendations": true,
    // 关掉大文件优化，我想一下子看到整个文件，并且不在意打开时卡的那一下
    "editor.largeFileOptimizations": false,
    // 忘记是干啥的了，貌似是跟 snippet 有关
    "editor.suggest.snippetsPreventQuickSuggestions": false,

    // 强制 Git 在 Commit 的时候使用 GPG 签名
    // 关于和 WSL 的搭配使用可以参考文章: https://lengthmin.me/posts/wsl2-gpg/
    "git.enableCommitSigning": true,

    // 代理设置
    // "http.proxy": "http://127.0.0.1:8889",

    // ===================== 扩展设置 =====================

    // 更改 UI 的字体，我装了一个扩展 CustomizeUI，可以暴力对 VSC 注入 CSS
    // 下面的一块都是对那个扩展的设置
    "customizeUI.activityBar": "regular",
    "customizeUI.font.monospace": "Sarasa Mono CL",
    "customizeUI.font.regular": "Sarasa Mono CL",
    "customizeUI.moveStatusbar": true,

    // 我装了一个小扩展 Jump to right bracket 
    // 可以绑定 Shift+Enter 跳出当前括号
    // 下面的设置指定了什么算右括号
    "jumpRightBrackets.rightBrackets": "}])>\"';$",

    // 一个小扩展: Clock
    // 可以在状态栏显示一个时钟，有助于促进早睡
    "clock.alignment": "Right",
    "clock.format": "HH:MM:ss",

    // Code-runner 的设置
    // 先保存文件再运行
    "code-runner.saveFileBeforeRun": true,
    // 在终端内运行，而不是在 Output 里运行
    "code-runner.runInTerminal": true,
}
```

### VSC 的设置层次

也许你已经注意到了，VSC 的设置页面里有个选项卡：

是的，VSC 的设置是 "分层" 的。作为用户，首先有一个用户设置。如果你使用远程开发插件，还会有一个 remote 设置。随后便是工作区的设置。这些设置会往上覆盖，也就是说如果某项设置在工作区跟用户设置里都有，那么工作区的会覆盖用户设置。

什么是工作区呢？根据[官方说法](https://code.visualstudio.com/docs/editor/workspaces), 你用 VSC 打开的文件夹就算一个工作区。工作区相关的设置跟其他文件会放在 `.vscode` 文件夹里。除了这种文件夹工作区之外，还有一种多文件夹工作区 -- 这种工作区使用一个 JSON 格式的 `xxx.code-workspace` 文件来保存各个文件夹的路径，使用 "File > Open Workspace"("文件 > 打开工作区") 选项打开这个文件之后就能打开这种工作区。

> A Visual Studio Code "workspace" is the collection of one or more folders that are opened in a VS Code window (instance). In most cases, you will have a single folder opened as the workspace but, depending on your development workflow, you can include more than one folder, using an advanced configuration called [Multi-root workspaces](https://code.visualstudio.com/docs/editor/workspaces#_multiroot-workspaces).

每一个工作区都可以有自己的设置，有自己的任务，有自己的 Snippet，有自己的一套选择启用的扩展，还可以保存已经打开的编辑器状态。但是快捷键跟一些特定设置只能在用户级设置。顺带一提，在某次更新后，已经有用户级的任务了。(之前任务必须跟某个工作区绑定)

VSC 还能够按语言分层设置，比如：

```json
// Markdown
"[markdown]": {
    // enable suggestion in markdown
    "editor.quickSuggestions": {
        "other": true,
        "comments": false,
        "strings": false
    },
    // using table formatter
    "editor.defaultFormatter": "darkriszty.markdown-table-prettify"
},
```

## 命令面板

按 Ctrl+P, VSC 就会在上面给你弹出一个打字的栏子，你可以在里面打文件名，然后快速打开某个文件。默认文件的排列顺序是按最近打开时间由近到远排序的，直接按 Ctrl+Tab 也可以一个一个选。

如果它只能跳转文件，那未免有点鸡肋 -- 事实上，通过打点特殊的前缀，它几乎可以跳转到任何 VSC 认识的地方。比如打个 `@`, 它就可以跳转到当前文件里的所有符号 (比如 Markdown 文件里的各个标题，代码里的各个函数); 打个 `:` 接数字就可以按行号跳到任意行。打一个问号 `?` ，你就能知道都有啥特殊的开头单词了。有了它，基本上鼠标就不用动了。

其中最重要的开头字符，就是 `>`. 它甚至重要到有一个单独的快捷键 `Ctrl+Shift+P`. 打开之后，你就可以执行 VSC 里的 **几乎任何指令** 。它有点类似于 vim 的 `:`, 但是去掉了一些朴素的指令 (比如说光标上下移动). 每当我 (或者你想) 打开什么东西，但是又不知道 VSC 把它放在哪里的时候，你就可以按 Ctrl+Shift+P，然后按名字搜一搜，基本上马上就有。而且 VSC 还会很贴心地把跟那个指令绑定的快捷键显示在旁边，如果你想提高效率，大可以记一下你常打操作的快捷键。一般来讲，我用得最多的指令，是换主题跟换编码 (打 `theme` 跟 `encoding` 就可以出来了). 有时候一些扩展提供的要手动开启的功能，也可以在这里开启，比如 Monkey Patch (一个用来往 VSC 里注入任意 JS 的扩展) 就需要你手动执行一个 enable 指令。你还可以在这里打 `task` 打开任务界面，然后执行任意任务。总之这里就是 VSC 的 "控制台"/"终端", 可以通过文字控制 VSC 的方方面面。

## 快捷键与 Snippet

### Snippet

### 快捷键绑定

#### Git: 部分暂存

```json
{
    "key": "ctrl+shift+=",
    "command": "git.stageSelectedRanges"
},
```

#### 重构相关

要想将 Code Action 绑定到快捷键，可以这样写：

```json
{
  "key": "ctrl+shift+r ctrl+e",
  "command": "editor.action.codeAction",
  "args": {
    "kind": "refactor.extract.function"
  }
}
```

正如参数名字 `kind` 暗示的，这个是一个 "筛选" 用而不是 "选择" 用的参数。当有多个重构适用时，它会弹出一个菜单让你选。

详见文档：[Keybindings for Code Actions](https://code.visualstudio.com/docs/editor/refactoring#_keybindings-for-code-actions)

## 任务与指令

### 调试

调试可以视作特别的任务，配置写在 `launch.json` 里。它一般由对应语言的扩展提供支持。

TODO: Debug C/C++

高级断点：条件断点与控制流计数器。[文档](https://code.visualstudio.com/docs/editor/debugging#_advanced-breakpoint-topics)

### 自定义任务

```json
// keybinding.json
[
    {
        "key": "f5",
        "command": "workbench.action.tasks.runTask",
        "args": "run-program"
    },
    {
        "key": "f6",
        "command": "workbench.action.tasks.runTask",
        "args": "test"
    }
]
```

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "compile-c-gcc",
            "type": "shell",
            "command": "gcc",
            "args": [
                "-Wall",
                "-Wextra",
                "${file}",
                "-o",
                "${fileBasename}.exe"
            ],

            // https://code.visualstudio.com/docs/editor/tasks#_output-behavior
            "presentation": {
                "echo": true,
                "reveal": "silent",
                "focus": false,
                "panel": "shared",
                "showReuseMessage": false,
                "clear": true
            }
        },
        {
            "label": "run-program",
            "type": "shell",
            "command": [
                "./${fileBasename}.exe"
            ],
            "dependsOn": [
                "compile-c-gcc"
            ],
            "problemMatcher": [
                "$gcc"
            ],

            "presentation": {
                "echo": true,
                "reveal": "always",
                "focus": true,
                "panel": "shared",
                "showReuseMessage": false,
                "clear": true
            }
        }
    ]
}
```

文档：[可以在 Task 里使用的变量](https://code.visualstudio.com/docs/editor/variables-reference)

> Note: **Not** all properties will accept variable substitution. Specifically, **only** `command`, `args`, and `options` support variable substitution.

### 自定义 Problem Matcher

文档：[自定义 Problem Matcher](https://code.visualstudio.com/docs/editor/tasks#_defining-a-problem-matcher)


# 当 VSC 遇到 xxx

## VSC + C/C++

## VSC + Python/Java

## VSC + Markdown + HyperSnips
