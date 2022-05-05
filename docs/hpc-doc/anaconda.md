# Anaconda3 及 Jupyter 环境搭建

> - 因为集群自带的环境非常老旧，所以各种环境的安装必须考虑这点，这点在之后的教程会更详细地谈到

## 1. 使用 vscode-remote

参考教程 [使用 vscode-remote 开发](../vscode-remote/)

在配置好 vscode-remote 后，我们连接到集群

![vscode](https://gitee.com/villard/wiki-images/raw/master/anaconda3/vscode.webp)

按键盘 ++ctrl+grave++  呼出终端，这样就可以使用 remote bash 了

++grave++ 键在 ++1++ 的左边，++tab++ 的上面

## 2. Anaconda3 安装

集群上已有安装的 Anaconda，推荐直接使用。集群上的 Anaconda 位于`/opt/app/anaconda3`，使用以下的命令将 Anaconda 加入自己的环境中

```shell
export PATH=$PATH:/opt/app/anaconda3/bin
conda init
```

这样 Anaconda 会在你的`.bashrc` 中加入类似以下这样的内容

```bash
# >>> conda initialize >>>
# !! Contents within this block are managed by 'conda init' !!
__conda_setup="$('/opt/app/anaconda3/bin/conda' 'shell.bash' 'hook' 2> /dev/null)"
if [ $? -eq 0 ]; then
    eval "$__conda_setup"
else
    if [ -f "/opt/app/anaconda3/etc/profile.d/conda.sh" ]; then
        . "/opt/app/anaconda3/etc/profile.d/conda.sh"
    else
        export PATH="/opt/app/anaconda3/bin:$PATH"
    fi
fi
unset __conda_setup
# <<< conda initialize <<<
```

此时重新登录或`source ~/.bashrc`就可以看到在行首的`(base)`了

接下来需要创建一个新的虚拟环境，因为`base`位于公共位置，因此你无法在这个环境中新安装软件，因此使用以下命令创建并激活新环境

```shell
conda create -n <name>
conda activate <name>
```

### Anaconda3 自行安装

复制链接，打开 Anaconda 官网 <https://www.anaconda.com/products/individual#Downloads> 获得 Linux 版下载链接

在 VS Code 下方的 remote bash 中输入下载链接

例如：

> 可以直接复制此链接，但不保证版本最新

```shell
wget https://repo.anaconda.com/archive/Anaconda3-2021.11-Linux-x86_64.sh
```

下载完后，输入以下命令给予 Anaconda3 安装包运行权限

```shell
chmod +x ./Anaconda3-2021.11-Linux-x86_64.sh
```

运行

```shell
./Anaconda3-2021.11-Linux-x86_64.sh
```

## 3. Jupyter Notebook 环境

在 VS Code 中，安装 Python 插件

![python](https://gitee.com/villard/wiki-images/raw/master/anaconda3/python.webp)

在 remote bash 中输入，安装 Jupyter 环境和 Jupyter Notebook

```shell
conda install jupyter notebook
```

创建一个新文件夹

```shell
mkdir dev
cd dev
```

在 VS Code 中打开 dev 文件夹

创建 `.ipynb` 文件

!!! warning "注意"
    **选择解释器时请务必选择 Anaconda3 中的 Python**

如图中第二个，如果创建了虚拟环境需要选择创建时使用的名字

![interpreter](https://gitee.com/villard/wiki-images/raw/master/anaconda3/chooseinterpreter.webp)
