# Bash 语法补充包

> 目前版本：20221025

本文期望读者具有基本的 Bash 使用基础。如在阅读中遇到困难，可参阅[我社 Linux 101 第六讲]()内容。

## Bash 大致语法 Overview

使用 `A::B` 指示 "用 B 分隔的 A 列表"

```antlr
nls: newlines, newline list, 指一个或多个 newline
sop: separator_op, 分割不同 bash 指令用的操作符, 可以是 '&' 或 ';'

complete_cmd  :  list sop?
list          :  and_or::sop
pipeline      :  "!"? cmd::"|"

cmd: simple_cmd | compound_cmd redirect* | func_def

compound_cmd
    : '{' list '}'            # brace group
    | '(' list ')'            # subshell
    | for | case | if | while | until

redirect: 详见下面的章节

func_def: name '(' ')' compound_cmd redirect?
```

## 重定向相关 (Redirection)

参考：<https://www.gnu.org/software/bash/manual/html_node/Redirections.html>

**重定向带顺序的!!! 由后往前执行!!!!** 比如说 `2>&1 >f` 会先 `>f`, 然后才 `2>&1`, 导致无法产生想要的结果; 应该使用 `>f 2>&1`.

下面用 `n`, `nn` 代表数字 (文件描述符), `m` 代表数字或 `-`, `f` 代表文件名，`T` 代表任意文本，`[]` 中代表可选

- `[n]<f`: 重定向输入 `n(=0)` 为文件 `f`
- `[n]>[|]f`: 重定向输出 `n(=1)` 为文件 `f`, 当启用了 `noclobber` 且有 `|` 时不会覆盖已存在的文件
- `[n]>>f`: 使用附加模式重定向输出，
- `&>f`, `>&f`: 重定向 `1` 和 `2` 到文件 `f`, 优先选择第一种，等价于 `>f 2>&1` (注意顺序)
- `&>>f`: 使用附加模式重定向 `1`, `2`，等价于 `>>f 2>&1`
- `[n]<<[-]d ...TEXT d`: here-document, 向 `n(=0)` 里送 `...TEXT`, 带 `-` 则忽略 `...TEXT` 中的行首 Tab
- `[n]<<< ...TEXT`: here-string
- `[n]<&m`: 将 `n(=0)` 设置为 `m` 的复制; 若 `m=-`, 则关闭 `n`
- `[n]>&m`: 将 `m` 设置为 `n(=1)` 的复制; 若 `m=-`, 则关闭 `n`
- `[n]<&nn-`, 将 `nn` "移动" 到 `n(=0)` (也就是复制到，再关闭)
- `[n]>&nn-`, 将 `n(=1)` "移动" 到 `nn`

`noclobber` 可以通过如下方法启用：

```bash
set -o noclobber
echo Hello > file
echo Hello >| file # fail
```

一些特殊的文件：

- `/dev/null`: 空文件，可以 "吞噬" 任何输入，一般用于丢弃输出
- `/dev/random`: 输出随机值的文件
- `/dev/fd/n`: 文件描述符 `n` 代表的文件
- `/dev/stdin`, `/dev/stdout`, `/dev/stderr`
- `/dev/tcp/host/port`: 建立 TCP 链接
- `/dev/udp/host/port`: 建立 UDP 链接

## Bash 表达式展开过程

参考：<https://www.gnu.org/software/bash/manual/html_node/Shell-Expansions.html>

- 大括号展开 (brace expansion)
- 波浪号展开 (tilde expansion)
- 参数展开 (parameter expansion)
- 命令替换 (command substitution)
- 算术展开 (arithmetic expansion)
- 单词切分 (word splitting)
- 文件名展开 (filename expansion)
- (可选) 进程替换 (process substitution)
- 引号删除 (quote removal)

### 大括号展开

> 多选

类似文件名展开，但是文件名展开要求文件存在，而大括号展开不需要。

```bash
# 注意大括号展开的逗号后面不能加空格, 否则会失败
$ echo a{d,c,b}e
ade ace abe
```

大括号内还可以写数字/字符+增长, 比如 `{x..y[..incr]}`

大括号展开是严格 "词法" 的，它不会对任何特殊字符作出 "反应", 包括大括号自己：

```bash
chown root /usr/{ucb/{ex,edit},lib/{ex?.?*,how_ex}}
```

### 波浪号展开

> 快速访问[目录栈](https://www.gnu.org/software/bash/manual/html_node/The-Directory-Stack.html)与 `HOME`

- `~`: `$HOME`
- `~/foo`: `$HOME/foo`
- `~fred/foo`: fred 的 HOME 下的 foo 文件夹
- `~+/foo`: `$PWD/foo`
- `~-/foo`: `${OLDPWD-'~-'}/foo`, 但是展开结果与 `$OLDPWD/foo` 很类似，推测大括号是用于删除可能的末尾斜杠的
- `~N`, `~+N`: `dirs +N`, 其实 `~+` 也可以看作 `N=0` 的情况
- `~-N`: `dirs -N`, 其实 `~-` 也可以看作 `N=1` 的情况

### 参数展开

- `${!p*}`, `${!p@}`: 间接展开，先展开名字前缀为 `p` 的变量，把展开后的东西当变量名再展开一次
- `${p:-default}`: 默认值
- `${p:=default}`: 空的时候不但用默认值代替，还会赋值
- `${p:?prompt}`: 空的时候往 stderr 打 prompt
- `${p:+prompt}`: 不空的时候往 stderr 打 prompt
- `${p:offset}`, `${p:offset:length}`: 切片，从 `[offset, offset + lenght)`, 从 0 开始，负数反之。负数 offset 负号前要打空格，否则会跟 `:-` 冲突。如果 p 是 `@` 或是 `*`, 那么对位置参数数组做切片
- `${#p}`: 获得 p 展开后的字符长度
- `${p@op}`: 对 p 展开后做一些变换：
    - `U`: 转全大写
    - `u`: 转首字母大写
    - `L`: 转全小写
    - `Q`: 转成 "quoted" 的形式，即还可以被 input 读回去的形式
    - `E`: 展开后带一些反斜杠转义，使结果成为有效的变量名，可以被用于 `$'...'`
    - `P`: 展开后，使结果成为有效的 Prompt
    - `A`: 展开成 `p=$p` 的格式
    - `K`: 与 `A` 类似，但是对数组会展开成键值对赋值的形式
    - `a`: 展开成 p 的属性
    - `k`: 与 `K` 类似，但是展开后的东西会过一边单词切分

还有一堆跟[模式匹配](https://www.gnu.org/software/bash/manual/html_node/Pattern-Matching.html)有关的变量展开，标志是大括号里有 `#%/^,` 的

### 命令替换

`$(command)` 和 `` `command` ``. 特别地，`$(cat file)` 可以被替换为更快的 `$(< file)`. 命令替换可以叠加，如果使用两个反引号引住命令替换，替换后的内容将不会经过内层的单词切分和文件名展开。

### 算术展开

`$(( expr ))`. 里面的单词会经历参数展开，变量替换和引号消除，可以嵌套。合法的算式可以看这里 [Shell Arithmetic](https://www.gnu.org/software/bash/manual/html_node/Shell-Arithmetic.html)

### 进程替换
- `<(cmd-list)`: 执行 `cmd-list`, 其输出会被写入到某个文件里，然后这个文件的路径被当成展开结果
- `>(cmd-list)`: 会被展开成一个文件路径，然后往该路径里的任何写入会被当成 `cmd-list` 的标准输入

参考：[SO](https://unix.stackexchange.com/a/27346/546053)

常见使用：

```bash
# 这样不行, 因为 bash 的 stdin 是读给脚本当 stdin 的, 不是用来读脚本的
curl -o - http://example.com/script.sh | bash

# 这样可以, 下载下来的脚本被存到某个文件里, 然后 bash 可以接受一个文件名作为参数执行那个文件
bash <(curl -o - http://example.com/script.sh)
```

```bash
# 查找所有我不够权限看的文件
# 这里把标准错误写到了一个文件里, 这个文件里的内容会被当成 sed 的标准输入
(ls /proc/*/exe >/dev/null) 2> >(sed -n \
  '/Permission denied/ s/.*\(\/proc.*\):.*/\1/p' > denied.txt )
```

### 单词切分
将环境变量 `IFS` 当作单词切分的分隔符，然后对新扩展的任何内容进行切分。需要注意的是，如果没有进行过扩展，那么单词切分也不会被进行。

> When a quoted null argument appears as part of a word whose expansion is non-null, the null argument is removed.

这意味着 `-d''` 并不会是 `-d ''`, 而只是 `-d`. 

### 文件名扩展

参见 [glob](https://en.wikipedia.org/wiki/Glob_(programming)).

### 引号移除

移除所有引号并进行转义。

## 完整 Posix Shell 语法

<https://pubs.opengroup.org/onlinepubs/9699919799/utilities/V3_chap02.html#tag_18_10>