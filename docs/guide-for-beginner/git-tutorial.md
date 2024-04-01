# Git 教程

<!-- > 本教程有别于其他 Git 教程，它专注于常见的工程流程。若您需要更多的 Git 基础知识，请参考 [Pro Git](https://git-scm.com/book/zh/v2)。 -->

## 初识 Git

Git 是一个分布式版本控制系统，用于跟踪文件的变化并协作开发。它是由 Linus Torvalds 于 2005 年创建的。

在 Windows 上，可以通过 [Git for Windows](https://gitforwindows.org/) 安装 Git，在 Linux 上，可以通过包管理器安装 Git，例如：

```bash
# Debian/Ubuntu
sudo apt install git
```

在 MacOS 上，可以通过 [Homebrew](https://brew.sh/) 安装 Git，或者使用 Xcode 自带的 Git。

对于刚下载的 Git，需要进行一些配置：

```bash
git config --global user.name "Your Name"
git config --global user.email "user@example.com"
```

这里的 `--global` 选项表示这些配置将应用于所有的 Git 仓库。若要对特定仓库进行配置，可以去掉 `--global` 选项。

## Git 基础概念

### 仓库

Git 的管理是基于仓库（Repository）的，仓库可以看成一个储存了 Git 元信息的目录。可以在需要进行版本控制的目录下执行 `git init` 命令，将其初始化为一个 Git 仓库。

### 工作区与暂存区

一个文件在 Git 中有三种状态：已提交（committed）、已修改（modified）和已暂存（staged），这三种状态对应 Git 三个工作区域的概念：

- 工作区（Working Directory）：即我们的工作目录，我们对文件的修改都发生在这里。
- 暂存区（Staging Area）：即我们的缓存区，我们可以通过 `git add` 命令将工作区的文件添加到暂存区。暂存区存储了我们对文件的修改。
- 仓库（Repository）：即我们的版本库，我们可以通过 `git commit` 命令将暂存区的文件提交到仓库。

所以我们想要修改一个文件，实际上需要经历三个步骤：

1. 在工作目录中修改文件。
2. 使用 `git add` 将修改的文件添加到暂存区。
3. 使用 `git commit` 将暂存区的文件提交到仓库。

### Commit 与 HEAD 与 Reset

Commit 是 Git 中最小的版本单元，它包含了一个快照（snapshot）和一个指向该快照的指针。每次提交都会生成一个唯一的 SHA-1 值，这个值可以用来标识提交。

我们可以将目前的 Git 仓库想象为一个树状结构，每个 Commit 都是一个节点，每个节点都有一个指向父节点的指针。这样就形成了一个有向无环图。当前的 Commit 节点通过一个名为 `HEAD` 的指针指出。通过这样的树状结构，我们可以很方便地查看并回溯历史记录。

回溯历史记录的方法为使用 `git reset` 命令，它有三种模式：

- `--soft`：仅仅将 `HEAD` 指针移动到指定的 Commit，不改变暂存区和工作区。
- `--mixed`：将 `HEAD` 指针移动到指定的 Commit，同时重置暂存区，但不改变工作区。这是默认的模式。
- `--hard`：将 `HEAD` 指针移动到指定的 Commit，同时重置暂存区和工作区。

我们可以看见，只有 `--hard` 模式会改变工作区。如果我们希望回溯历史记录时同时改变工作区，记得加上 `--hard` 选项。但这同时也会删除工作区的修改，所以当你想要回溯历史记录时，记得先将工作区的修改 `commit` 到仓库，这也是我们说的保持工作区干净的原因。

### 分支与切换

分支实际上与 HEAD 指针类似，它也指向一个 Commit。可以说分支的存在是为了更简单地在不同的 Commit 之间切换，在 Git 为基础的协同开发中，分支一般代表不同的开发任务与进度。例如在开发一个新功能时，我们在 `master` 分支上新建一个 `feature-a` 分支，然后在 `feature-a` 分支上进行开发，这时开发这个 `feature-a` 的人可以在不影响 `master` 分支的情况下进行开发，并且可以随时切换回 `master` 分支。

在 Git 中，分支的切换是通过 `git checkout` 命令实现的：

```bash
# 在 master 分支上新建 feature-a 分支
git checkout -b feature-a
# 进行开发
some-development
# 切换回 master 分支
git checkout master
```

由于分支本身只是一个指针，所以也可以使用 `git checkout commit-id` 来切换到某个 Commit，这样我们就可以在不同的 Commit 之间切换。

事实上，`git checkout` 命令还可以作用于文件，所以在新版本的 Git 中，切换分支有一个新的命令 `git switch`，它只能作用于分支。可以通过 `git switch --help` 查看相关信息，这里先按下不表。

### Merge 与 Rebase

这里可以算是 Git 中最复杂的部分了。许多大型项目进行一次 Rebase 可能要花费数小时，甚至数天的时间。

延续上述场景，当 `feature-a` 开发完成后，我们需要将 `feature-a` 分支上完成的工作合并到 `master` 分支上。我们使用 `git merge` 命令来合并分支：

```bash
# 切换到 master 分支
git checkout master
# 合并 feature-a 分支
git merge feature-a
```

这样就能将 `feature-a` 分支上的工作合并到 `master` 分支上。但这样的合并会产生一个新的 Commit，这个 Commit 会有两个父节点，一个是 `master` 分支的 Commit，一个是 `feature-a` 分支的 Commit。这样的合并方式叫做合并提交（Merge Commit）。

> 例外：Fast-forward 合并
>
> 当 `feature-a` 分支的 Commit 是 `master` 分支的 Commit 的直接后继时，Git 会使用 Fast-forward 合并，这样的合并不会产生新的 Commit，只是将 `master` 分支的指针直接指向 `feature-a` 分支的 Commit。

另一种合并方式是 Rebase，Rebase 会将 `feature-a` 分支上的 Commit 逐个应用到 `master` 分支上，这样的合并方式会产生新的 Commit，但这些 Commit 不会有两个父节点，这样的合并方式叫做变基（Rebase）。

```bash
# 切换到 master 分支
git checkout master
# 变基 feature-a 分支
git rebase feature-a
```

Rebase 的好处是可以保持 Commit 的线性，这样的 Commit 更容易理解，但 Rebase 也有一个缺点，那就是 Rebase 会改变 Commit 的 SHA-1 值，如上述操作可以说 Rebase 是诸葛将修改应用到 master 分支上，这样会重写 master 分支的历史记录，于是当进行多人协作时，就会发现两个人的 master 分支不一致。所以在协作中多使用 Merge 合并。不过也有另一种操作，即先 Rebase 后 Merge，这样可以保持 Commit 的线性，又不会改变历史记录。

```bash
# 在 feature-a 分支上变基
git rebase master
# 切换到 master 分支
git checkout master
# 合并 feature-a 分支
git merge feature-a
```

由于 Rebase 后的 feature-a 是 master 的直接后继，所以这里会使用 Fast-forward 合并。

在合并之后，若不再需要 `feature-a` 分支，可以使用 `git branch -d feature-a` 删除分支。

### 远程仓库

在多人协作中，通常会有一个远程仓库，这个远程仓库是所有人共享的。这里以 Gitee 为例，GitHub 是一个提供 Git 仓库托管服务的网站。远程仓库有三种协议：HTTP、SSH 和 Git，为了方便起见，我们这里使用 SSH 协议。

首先需要正确配置 SSH 密钥。可以通过 `ssh-keygen` 命令生成 SSH 密钥：

```bash
ssh-keygen -t rsa -b 4096 -C ""comment"
```

这里的 "comment" 是注释，用来标识这个密钥，帮助你区分不同的密钥。这次生成的密钥会保存在 `$HOME/.ssh/id_rsa` 和 `$HOME/.ssh/id_rsa.pub` 文件中（`$HOME$` 代表家目录）。`id_rsa` 是私钥，`id_rsa.pub` 是公钥。复制 `id_rsa.pub` 中的内容，然后在 Gitee 的设置中添加 SSH 密钥。为了测试是否配置成功，使用

```bash
ssh -T git@gitee.com
```

验证是否配置成功。若配置成功，将出现 `Hi userxxx! You've successfully authenticated, but GITEE.COM does not provide shell access.` 的提示。

我们在 Gitee 上新建一个仓库，按照仓库创建后的提示即可将本地仓库与远程仓库关联：

```bash
# 关联远程仓库 使用 SSH 协议
git remote add origin git@gitee.com:userxxx/test-project.git
git push -u origin master
```

这样就将本地仓库与远程仓库关联起来了。事实上，目前只是将本地仓库的 `master` 分支推送到了远程仓库的 `master` 分支上，并且 `-u` 选项表示将本地的 `master` 分支与远程的 `master` 分支关联起来，作为这个分支的默认远程分支。当后续对 `master` 分支进行修改后，可以直接使用 `git push` 命令将修改推送到远程仓库。同理可以将其他分支推送到远程仓库。

```bash
# 推送 feature-a 分支到远程仓库
git checkout feature-a
git push origin feature-a
```

当远程仓库发生了修改，可以通过 `git pull` 命令将远程仓库的修改拉取到本地仓库。

```bash
# 切换到 master 分支
git checkout master
# 拉取远程仓库的修改
git pull
```

除了将本地仓库与远程仓库关联，还可以通过 `git clone` 命令将远程仓库克隆到本地：

```bash
git clone git@gitee.com:userxxx/test-project.git
```

## Git 协同开发工作流

在你了解了 Git 的基本概念后，你可以开始使用 Git 进行协同开发了。在学习 Git 基础知识时，你已经了解到 Git 稍显繁杂的操作，但你真正入门 Git 协同开发只需要掌握几个基本操作。这里推荐一种简单的基于 Feature 分支的工作流。

### 主分支只接受合并请求

在这种工作流中，主分支（`master`）只接受来自其他分支的合并，不允许直接向主分支提交代码。这样可以保证主分支代码的稳定性。

### 每个 Feature 都有一个独立的分支

在这种工作流中，每个 Feature 都有一个独立的分支，这样可以保证不同 Feature 之间的代码不会相互影响。在给项目添加新功能时，从主分支新建一个 Feature 分支，然后在这个分支上进行开发。

```bash
# 切换到 master 分支
git checkout master
# 更新 确保 master 分支是最新的
git pull
# 新建一个 Feature 分支
git checkout -b feature-a
# 进行开发
some-development
```

### 合并请求

当 Feature 开发完成后，需要将 Feature 分支合并到主分支上。在协作开发中，这种合并在某个人的本地仓库中进行并不方便审阅，我们一般在远程仓库中进行合并请求（Pull Request）。

在 Gitee 的仓库页面上点击 `Pull Request`，新建一个合并请求，源分支选择 Feature 分支，目标分支选择主分支，然后填写合并请求的标题与描述。这样就可以发起一个合并请求了。你们可以在这个合并请求中讨论代码的修改，审阅代码，最终在页面中点击合并按钮将 Feature 分支合并到主分支上。这种合并方式可以避免复杂的合并操作，并且当多个 feature 同时进行开发时也不容易产生合并冲突。在完成合并后，可以删除 Feature 分支。

## Git 技巧

这里列举几个不错的教程：

- [git - 简明指南 (中文翻译)](https://rogerdudler.github.io/git-guide/index.zh.html)
- [Git 的奇技淫巧 (中文翻译)](https://github.com/521xueweihan/git-tips)
- [Git - Book](https://git-scm.com/book/zh/v2)

同时推荐在熟悉 Git 的基本概念后使用 Git 的图形化工具，如 VsCode 的 Git 插件、JetBrains 的 Git 集成等。

更多的 Git 使用技巧等待你在使用中发现。

