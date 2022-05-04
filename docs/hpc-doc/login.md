# Linux 如何使用代理登录到集群

> 这里主要介绍 Linux，如何登陆到集群及如何使用 Docker 运行 EasyConnect
>
> ~~防止 EasyConnect 污染自己的环境~~

## 1. Docker 安装

### 什么是 Docker

Docker 是一种容器，容器是一个标准的软件单元，它打包代码及其所有依赖项，以便应用程序从一个计算环境快速可靠地运行到另一个计算环境。

Docker 容器镜像是一个轻量级的、独立的、可执行的软件包，包括运行应用程序所需的一切。容器将软件与其环境隔离开来，并确保尽管开发和暂存之间存在差异，但它仍能统一工作。

传统 vm 虚拟化环境运行应用，每个虚拟机存在一个 Linux 系统，会有很大一部分程序重复运行，浪费资源。

<center>
<img src="https://gitee.com/villard/wiki-images/raw/master/login/vm.webp" width ="350px">
<img src="https://gitee.com/villard/wiki-images/raw/master/login/container.webp" width="350px">
</center>

### 下载 Docker

打开终端，输入以下命令安装 Docker

```shell
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

安装完毕后运行以下命令开启 Docker

```shell
sudo systemctl start docker
```

尝试运行 `hello-world`

```shell
sudo docker run hello-world
```

如果成功出现 `Hello from Docker!` 则正常

## 2. EasyConnect

我们使用的 EasyConnect Docker 版来自这里

```url
https://github.com/Hagb/docker-easyconnect
```

安装纯命令行版 EasyConnect

```shell
sudo docker pull hagb/docker-easyconnect:cli
```

完成后输入以下命令运行 EasyConnect

在使用 VPN 时请务必保持终端不要关闭

```shell
sudo docker run --device /dev/net/tun --cap-add NET_ADMIN -ti -p 127.0.0.1:1080:1080 -p 127.0.0.1:8888:8888 -e EC_VER=7.6.3 -e CLI_OPTS="-d vpn.hitsz.edu.cn" hagb/docker-easyconnect:cli
```

> - 其中 `-e EC_VER=7.6.3` 表示使用 7.6.3 版本的 EasyConnect，请根据实际情况修改版本号。
> - VPN 服务器已指定
> - 浏览器（或其他支持的应用）可配置 SOCKS5 代理（可以通过插件配置），地址 127.0.0.1，端口 1080；也可以使用 http 代理，地址 127.0.0.1，端口 8888。

上述命令输入后会要求输入账号密码，`username` 为学号，`passwoord` 为密码

## 3. 设置浏览器代理

主要有以下两种方法（只需选择一种）

### a. Firefox 和 Chrome 均可使用 SwitchyOmega 进行管理

Firefox 打开下面链接安装 SwitchyOmega 插件

```url
https://addons.mozilla.org/zh-CN/firefox/addon/switchyomega/
```

安装完后，在右上角打开 SwitchyOmega 的选项后出现下面的页面

![option](https://gitee.com/villard/wiki-images/raw/master/login/install.webp)

在左边栏选择新建情景模式

在名称中填入一个名字，然后点击创建

![option](https://gitee.com/villard/wiki-images/raw/master/login/option.webp)

创建完成后在情景模式中填入

> - 代理协议 `HTTP` 
> - 代理服务器 `127.0.0.1`
> - 端口 `8888`

选择应用更改

![option](https://gitee.com/villard/wiki-images/raw/master/login/complete.webp)

然后在右上角的 SwitchyOmega 插件里选择你刚刚创建的情景模式，就可以访问学校内网了

![enable](https://gitee.com/villard/wiki-images/raw/master/login/enable.webp)

### b. 使用系统代理

这里以 Gnome 环境举例

![gnome](https://gitee.com/villard/wiki-images/raw/master/login/gnome.webp)

![gnome_set](https://gitee.com/villard/wiki-images/raw/master/login/gnome_settings.webp)

## 4. 登录到集群

浏览器代理配置好后，可输入网址登录集群

```shell
http://hpc.hitsz.edu.cn
```

在最上方导航栏选择项目里的 `public_cluster`

![public](https://gitee.com/villard/wiki-images/raw/master/login/hpchome.webp)

在划红线处，选择最右边按钮，**ssh 按钮**，可以看到服务器的**内网 ip**

在 **左边导航栏** 的 **用户** 一栏，可以看到自己的集群用户名

![user](https://gitee.com/villard/wiki-images/raw/master/login/user.webp)

其中 u 开头的即是用户名

## 5. ssh

前面已经拿到了内网 IP，端口和 ssh 账号，下面介绍如何通过 SOCKS5 代理进行 ssh

先安装一个叫 `netcat` 的包，以 ubuntu 为例

```shell
sudo apt install netcat
```

因为貌似有几个命令都叫 netcat ，所以尝试 netcat 命令，看看是不是安装了正确的 netcat

```console
$ netcat
usage: nc [-46cDdFhklNnrStUuvz] [-C certfile] [-e name] [-H hash] [-I length]
	  [-i interval] [-K keyfile] [-M ttl] [-m minttl] [-O length]
	  [-o staplefile] [-P proxy_username] [-p source_port] [-R CAfile]
	  [-s sourceaddr] [-T keyword] [-V rtable] [-W recvlimit] [-w timeout]
	  [-X proxy_protocol] [-x proxy_address[:port]] [-Z peercertfile]
	  [destination] [port]

```

若与上述一致则正确。

然后输入 ssh 命令

```shell
ssh -o "ProxyCommand=netcat -X5 -x 127.0.0.1:1080 %h %p" username@hostname -p peer
```

> - `127.0.0.1:1080` 为使用之前 EasyConnect Docker 的 SOCKS5 端口
> - username 为前面在 hpc 集群网站获得的 u 开头的用户名
> - hostname 为网站获得的集群 ip
> - peer 为端口
> - ssh 登录密码为 hpc 平台登录密码

## 6. ssh 密钥快捷认证

在本机中输入

```shell
ssh-keygen
```

跟随指引，一路回车，便可完成 ssh 密钥的生成。**请务必保存好自己的私钥，不要泄露！**

接下来，我们将私钥加入到自己的身份认证中。在终端继续输入：

```shell
ssh-add ~/.ssh/id_rsa
```

然后将公钥打开

```shell
cat ~/.ssh/id_rsa.pub
```

复制其中的内容

在集群服务器上新建 authorized_keys

```shell
mkdir -p ~/.ssh
vim ~/.ssh/authorized_keys
```

这时会打开 vim 文本编辑器

按下 ++i++ 按键，进入编辑模式，并右键粘贴到上面节点终端中编辑界面的一个新行里。按下 ++esc++ ，输入 `:wq` 并敲 ++enter++ 键可保存并退出。
