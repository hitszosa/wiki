# PyTorch on SLURM

本文是在`public_cluster`上通过`SLURM`运行基于 PyTorch 编写的深度学习任务的教程

> 参考来自 [Princeton Research Computing](https://researchcomputing.princeton.edu/support/knowledge-base/pytorch) 的文章

## Why

可能有同学已经发现了，使用开虚拟机的方式使用集群有诸多不便之处，虽然卡资源是独享的随时可以使用，但是有的时候无法分配到资源，这是因为集群资源不多，有的同学申请了虚拟机仅做测试用，并不实际运行任务，但是忘记了退出或者说想过段时间再用，就没有删除，导致资源利用率低。所以我们推荐大家使用`SLURM`在`public_cluster`中运行任务。

这样做有几个好处

- 无需隔一段时间到控制台打开机器
- 提高了资源使用效率
- 提交任务后即使当前没有计算资源，任务也会进入队列`FIFO`执行，能够无需人工操作在当前占用资源的任务结束后自动开始

!!! note "注意"
    这篇教程默认你已经登录到集群<br>
    请确定已在集群环境中已安装 Anaconda3 环境<br>
    public_cluster 通过 slurm 调度

## 安装 PyTorch

在 conda 环境下，安装 PyTorch

```shell
conda install pytorch torchvision torchaudio cudatoolkit=11.3 -c pytorch
```

## Example GPU Job

### 下载数据

示例将展示如何在一个 Cluster 上跑一个简单的 PyTorch 脚本。我们将会用 MNIST 数据集训练一个简单的 CNN 网络。

```shell
git clone https://github.com/PrincetonUniversity/install_pytorch.git
cd install_pytorch
```

这会创建一个叫做 install_pytorch 的文件夹在你的用户目录，它包含我们跑这个示例需要的文件。

~~计算集群不会联网，所以我们在控制节点必须准备好所有需要的资源。~~

我们的计算节点和登陆节点网络环境一样，因此你原本在运行时处理的下载（例如缓存各种预训练模型）可以照常使用

### 编写 slurm 脚本

示例脚本如下，`SBATCH`开头的注释行表示对任务的描述，slurm 会根据这个描述进行资源分配、超时 kill、排队等管理。
*星号表示重要，最好填

```shell
#!/bin/bash
#SBATCH --job-name=torch-test    # 提交的作业的名字
#SBATCH --nodes=1                #* 使用的节点数 
#SBATCH --ntasks=1               #* 给予所有节点的总任务数
#SBATCH --cpus-per-task=1        #* 每个任务的 cpu 核心数 (>1 if multi-threaded tasks)，不填写默认 1，最好使用合适的值，例如 8
#SBATCH --gres=gpu:8             #* 每个节点的 gpu 数，按需求使用，集群每节点 8 卡，也即最多可以分配 8 卡，不建议全部占满
#SBATCH --time=00:05:00          # 时间限制 (HH:MM:SS)，可以不需要这一行，这样脚本退出以后任务自动结束

. ~/.bashrc                      # 重要：配置环境，例如 Anaconda，sbatch 不会自动配置 bash 环境
python ~/install_pytorch/mnist_classify.py --epochs=3
```

!!! warning "重要"

    time 必须设定足够，否则 time 截止时，任务会强制被结束，但如果不使用 time，注意保证任务在合理时间内结束

你可以使用以下命令监视你的任务的状态

```shell
squeue -u $USER
```

一旦任务开始，你将会得到一个 `slurm-xxxxx.out` 的文件在 `install_pytorch` 下。

这份文件包含了 PyTorch 和 slurm 的输出。

!!! note "注意"
    `slurm-xxxxx.out` 文件名中 `xxxxx` 为你当前提交的作业的 id

### 取消作业

可以通过 `scancel` 的命令停止作业，这个命令向进程发送`SIGTERM`

```shell 
scancel <jobid>
```

如果你的进程无视`SIGTERM`，你可以使用`skill`向进程发送`SIGKILL`强制结束

```shell
skill -9 <jobid>
```

### 查看所有节点状态

```shell
sinfo -Nl
```

输入上述指令可以通过 slurm 查看所有节点状态，得到以下结果。

```shell
NODELIST   NODES PARTITION       STATE CPUS    S:C:T MEMORY TMP_DISK WEIGHT AVAIL_FE REASON              
gpu1           1      gpu*        idle   40   40:1:1      1        0      1   (null) none                
gpu2           1      gpu*        idle   40   40:1:1      1        0      1   (null) none     
```

### 查看运行节点的状态

一旦提交了一个作业，作业正在运行，你便可以通过查看所有节点状态知道你的作业运行在的一个或多个节点。这些节点可以被登录。

例如你的作业运行在 gpu1 节点，即可使用命令登录节点。

```shell
ssh gpu1
```

查看该节点的运行情况

例如输入

```shell
nvidia-smi
```

可以查看 NVIDIA GPU 的利用率
