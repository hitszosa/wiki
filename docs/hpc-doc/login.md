# Linux 下登录到集群

> 这里主要介绍 Linux，如何登陆到集群及如何使用 Docker 运行 EasyConnect 
>
> ~~防止 EasyConnect 污染自己的环境~~

### 1. Docker 安装

使用浏览器打开 [Get Docker](https://docs.docker.com/get-docker/)

```
https://docs.docker.com/get-docker/
```

选择 Docker for Linux

![install_docker](https://gitee.com/villard/wiki-images/raw/master/login/install_docker.webp)

下面可以看到 Server 一栏，选择你正在使用的发行版，按照指示安装

安装完毕后尝试运行 `hello-world`

```shell
sudo docker run hello-world
```

如果成功出现 `Hello from Docker!` 则正常

若出现如下<b>错误</b>

```shell
docker: Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?.
See 'docker run --help'.
```

则运行以下命令开启 docker

```shell
sudo systemctl start docker
```

然后再尝试运行 `hello-world`



### 2. EasyConnect 

我们使用的  EasyConnect docker  版来自这里

```
https://github.com/Hagb/docker-easyconnect
```

安装纯命令行版 EasyConnect 

```shell
sudo docker pull hagb/docker-easyconnect:cli
```

完成后输入以下命令运行  EasyConnect 

```shell
sudo docker run --device /dev/net/tun --cap-add NET_ADMIN -ti -p 127.0.0.1:1080:1080 -p 127.0.0.1:8888:8888 -e EC_VER=7.6.3 -e CLI_OPTS="-d vpn.hitsz.edu.cn -u username -p password" hagb/docker-easyconnect:cli
```

>- 其中 `-e EC_VER=7.6.3` 表示使用 7.6.3 版本的 EasyConnect，请根据实际情况修改版本号；
>- vpn 服务器已指定好，其中 `username` 为学号 ， `passwoord`  为密码（若密码中有特殊字符，可能需要使用转义符号）
>- 浏览器（或其他支持的应用）可配置 socks5 代理（可以通过插件配置），地址 127.0.0.1，端口 1080；也可以使用 http 代理，地址 127.0.0.1，端口 8888。
>- 请注意密码会保存在 bash history 中，若有隐私要求请关闭 bash history



### 3. 浏览器代理

主要有以下两种方法

#### a. Firefox 和 Chrome 均可使用 SwitchyOmega 进行管理

配置大致如下

![SwitchyOmega](https://gitee.com/villard/wiki-images/raw/master/login/SwitchyOmega.webp)

#### b. 使用系统代理

这里以 Gnome 环境举例

![gnome](https://gitee.com/villard/wiki-images/raw/master/login/gnome.webp)

![gnome_set](https://gitee.com/villard/wiki-images/raw/master/login/gnome_settings.webp)

### 4. 登录到集群

浏览器代理配置好后，可输入网址登录集群

```shell
http://hpc.hitsz.edu.cn
```



在最上方导航栏选择项目里的 `public_cluster`

![public](https://gitee.com/villard/wiki-images/raw/master/login/hpchome.webp)



在划红线处，选择最右边按钮，**ssh 按钮**，可以看到服务器的**内网 ip**

在**左边导航栏**的 **用户** 一栏，可以看到自己的集群用户名

![user](https://gitee.com/villard/wiki-images/raw/master/login/user.webp)

其中 u 开头的即是用户名

### 5. ssh

前面已经拿到了内网 ip，端口和 ssh 账号

下面介绍如何通过 sock5 代理进行 ssh

先安装一个叫 `netcat` 的包，以 ubuntu 为例

```shell
sudo apt install netcat
```

因为貌似有几个命令都叫 netcat ，所以尝试 netcat 命令，看看是不是安装了正确的 netcat

```shell
$netcat
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

> - `127.0.0.1:1080` 为使用之前 easyconnect docker 的 socks5 端口
> - username 为前面在 hpc 集群网站获得的 u 开头的用户名
> - hostname 为网站获得的集群 ip
> - peer 为端口
> - ssh 登录密码为 hpc 平台登录密码



### 6. ssh 密钥快捷认证

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

按下 `i` 按键，进入编辑模式，并右键粘贴到上面节点终端中编辑界面的一个新行里。按下 `Esc` ，按下 `:wq` 可保存并退出。

