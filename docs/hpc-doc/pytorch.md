# PyTorch on HPC Clusters

> 参考来自 [Princeton Research Computing](https://researchcomputing.princeton.edu/support/knowledge-base/pytorch) 的文章

## 安装

!!! note "注意"

    这篇教程默认你已经登录到集群<br>
    请确定已在集群环境中已安装 Anaconda3 环境<br>
    public_cluster 通过 slurm 调度



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

这会创建一个叫做 install_pytorch 的文件夹在你的用户目录，它包含我们跑这个示例需要的文件。计算集群不会联网，所以我们在控制节点必须准备好所有需要的资源。

```shell
python download_mnist.py
```

### 编写 slurm 脚本

示例脚本如下

```shell
#!/bin/bash
#SBATCH --job-name=torch-test    # 提交的作业的名字
#SBATCH --nodes=1                # 使用的节点数
#SBATCH --ntasks=1               # 给予所有节点的总任务数
#SBATCH --cpus-per-task=1        # 每个任务的 cpu 核心数 (>1 if multi-threaded tasks)
#SBATCH --gres=gpu:8             # 每个节点的 gpu 数
#SBATCH --time=00:05:00          # 时间限制 (HH:MM:SS)

. ~/.bashrc                      # 重要：配置环境，sbatch 不会自动配置 bash 环境
python ~/install_pytorch/mnist_classify.py --epochs=3
```

!!! warning "重要"

    time 必须设定足够，否则 time 截止时，任务会强制被结束



你可以使用以下命令监视你的任务的状态

```shell
squeue -u $USER
```

一旦任务开始，你将会得到一个 `slurm-xxxxx.out` 的文件在 `install_pytorch` 下。

这份文件包含了 PyTorch 和 slurm 的输出。

!!! note "注意"
 
    `slurm-xxxxx.out` 文件名中 `xxxxx` 为你当前提交的作业的 id

### 取消作业
可以通过 `scancel` 的命令停止作业

```shell 
scancel jobid
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



