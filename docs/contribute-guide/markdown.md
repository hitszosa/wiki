# 让你提交的 Markdown Pro 起来

> Markdown 也需要规范？

是的。虽然我们往往将 Markdown 作为 HTML 生成语言使用，但 Markdown 设计之初就希望保证其代码在纯文本界面下的可读性。你的控制台、编辑器不会优化文字的排版（一般都会用等宽字体），但是浏览器会，因此就算你的 Markdown 写的非常的混乱，在浏览器（或者 ~~通过浏览器套壳实现的~~ 所见即所得编辑器，比如 Typora）里看起来依旧非常正常。为了照顾 ~~强迫症患者~~ 这种需求，我们需要给 Markdown 一个规范。

## 中文文案的通用规范

详情请参考[《中文文案排版指北》](https://github.com/sparanoid/chinese-copywriting-guidelines/blob/master/README.zh-Hans.md)，下面仅指出两个重点。

### 盘古之白

中文和西文、数字之间加空格。按照 [BV13M4y157Xc](https://www.bilibili.com/video/BV13M4y157Xc) 中的说法，中西文之间的间隔应该是全角的四分之一宽，但是四分之一空格既不方便打也不方便用等宽字体显示，所以统一使用普通空格代替。（按照某位不知名人士的说法，聪明的浏览器会忽略这个空格，愚蠢的终端需要这个空格）

```txt
哦，我的天呐，你居然连 Markdown 是什么都不知道，真是见鬼！
我真想用隔壁苏珊大妈 20" 的靴子狠狠的踢你的屁股，你这个坏东西！
```

需要注意的是，如果西文后面直接接符号，则不需要空格。

```txt
我们的代码托管在 GitHub。
```

### 全角标点

除非是西文整句，否则全部使用中文标点。

```txt
俗话说得好，“Too young, too simple.”
```

### 规范英文拼写

GitHub 是一个英文社区，所以我们需要规范英文拼写。（这句话是 Copilot 补出来的）

错误示范（[出处](https://github.com/sparanoid/chinese-copywriting-guidelines/blob/master/README.zh-Hans.md#%E4%B8%93%E6%9C%89%E5%90%8D%E8%AF%8D%E4%BD%BF%E7%94%A8%E6%AD%A3%E7%A1%AE%E7%9A%84%E5%A4%A7%E5%B0%8F%E5%86%99)）：

> 我们的客户有 github、foursquare、microsoft corporation、google、facebook, inc.。
>
> 我们的客户有 GITHUB、FOURSQUARE、MICROSOFT CORPORATION、GOOGLE、FACEBOOK, INC.。
>
> 我们的客户有 Github、FourSquare、MicroSoft Corporation、Google、FaceBook, Inc.。
>
> 我们的客户有 gitHub、fourSquare、microSoft Corporation、google、faceBook, Inc.。

LUG 和下属项目的规范拼写如下：

- HITsz LUG
- Wiki
- Weekly

有一些特殊情况可以不规范拼写，如大标题。

## Markdown 的规范

### 禁止滥用 `**加粗**`

不要把加粗当标题用。如果需要强调且单独成段，建议用下面提到的 [Admotions](#admonitions) 代替。

### 合理放置 [超链接](https://example.com)

一般来讲，一个段落里不应该给同一个名词超链接加上超链接两次。

PS：如果需要直接放网址，有专门的 Markdown 语法。以下两种写法是等价的：

```md
[https://example.com](https://example.com)
<https://example.com>
```

GitHub issue 里的网址甚至可以不用加 `<>`。

### 空行

标题和正文之间应该空一行。注意，Markdown 段落之间也需要空一行。

## 用 linter 偷懒

推荐以下两个 linter：

- [markdownlint-cli2](https://github.com/DavidAnson/markdownlint-cli2)：顾名思义，它是一个 Markdown 文件的 linter，可以检查诸如错误使用 `**` 的问题。但是默认的规则有点苛刻了，建议关闭这些规则：MD024（禁止多处相同的标题）、MD026（禁止标题以西文标点结束）。此外，也有 [VS Code 插件版](https://marketplace.visualstudio.com/items?itemName=DavidAnson.markdownlint-vs-code)。
- [autocorrect](https://github.com/huacnlee/autocorrect)：专门用于规范中英文混写的文案，可惜不检测规范拼写。同样有 [VS Code 插件版](https://marketplace.visualstudio.com/items?itemName=huacnlee.auto-correct)。

插件版本有一个比较烦人的缺点，就是修复提示很可能会打断你写文的思路。如果不希望被这种问题所困，建议改成仅保存时自动修复。

另外，还有一个专门用来检测规范拼写的程序 [typos-cli](https://github.com/crate-ci/typos)。

## 项目限定内容

### Wiki

Wiki 使用了 Material for MkDocs，加了一些 Markdown 扩展来让它有更丰富的功能，比如：

#### 代码高亮

````md
<!-- ```py 也行 -->
```python
for i in range(10):
    print(i)
```
````

效果：

```python
for i in range(10):
    print(i)
```

MkDocs 使用 Pygments 进行语法分割，支持的语言见 [Pygments 文档](https://pygments.org/docs/lexers/)。

#### 按键（代替<kbd\>）

- `++ctrl++` -> ++ctrl++
- `++ctrl+shift+esc++` -> ++ctrl+shift+esc++

参考：<https://facelessuser.github.io/pymdown-extensions/extensions/keys/>

#### Admonitions

```md
!!! info "这个可以代替 Markdown 的引用"
    指的是 `> 引用内容` 的语法
```

效果：

!!! info "这个可以代替 Markdown 的引用"
    指的是 `> 引用内容` 的语法

参考：<https://squidfunk.github.io/mkdocs-material/reference/admonitions/>

### Weekly

Weekly 的标题采用的字体接近繁体字形，建议在标题使用直角引号「」。
