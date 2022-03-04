# Anaconda3 及 Jupyter 环境搭建

> - 因为集群自带的环境非常老旧，所以各种环境的安装必须考虑这点，这点在之后的教程会更详细地谈到

### 1. 使用 vscode-remote

参考教程  **使用 vscode-remote 开发**

在配置好 vscode-remote 后，我们连接到集群

![vscode](https://gitee.com/villard/wiki-images/raw/master/anaconda3/vscode.webp)



按键盘 <kbd>Ctrl</kbd> + <kbd>`</kbd>  呼出终端，这样就可以使用 remote bash 了

<kbd>`</kbd> 键在 <kbd>1</kbd> 的左边 ，<kbd>tab</kbd> 的上面

### 2. Anaconda3 安装

复制链接，打开 [anaconda 官网](https://www.anaconda.com/products/individual#Download)

```
https://www.anaconda.com/products/individual#Downloads
```

获得 Linux 版下载链接



在 vscode 下方的 remote bash 中输入下载链接

例如：

> 可以直接复制此链接，但不保证版本最新

```sh
wget https://repo.anaconda.com/archive/Anaconda3-2021.11-Linux-x86_64.sh
```



下载完后，输入以下命令给予 anaconda3 安装包运行权限

```shell
chmod +x ./Anaconda3-2021.11-Linux-x86_64.sh
```

运行

```shell
./Anaconda3-2021.11-Linux-x86_64.sh
```

### 3. Jupyter Notebook 环境

在 vscode 中，安装 python 插件

![python](https://gitee.com/villard/wiki-images/raw/master/anaconda3/python.webp)



在 remote bash 中输入，安装 jupyter 环境和 jupyter notebook

```shell
conda install -n base jupyter notebook
```



创建一个新文件夹

```shell
mkdir dev
cd dev
```

在 vscode 中打开 dev 文件夹

创建 `.ipynb` 文件

<b>选择解释器时请务必选择 anaconda3 中的 python </b>

如图中第二个

![interpreter](https://gitee.com/villard/wiki-images/raw/master/anaconda3/chooseinterpreter.webp)

