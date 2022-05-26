---
title: 检查网络的小技巧
date: 2022-05-26
---

我！网！断！了！

作为程序员网络即是生命，如果连自己断网了都修不好未免有些丢脸。本文总结了一些与网络配置做斗争的小技巧。

## 基本思路

调试网络就好像 Debug，首先要知道哪里可能出问题，然后把可能出问题的地方一个个排除掉。首先基本原则是分层。比如在宿舍用路由器，用着用着突然断网，首先应该检查路由器内部网通不通，即检查本机到路由器、本机到其他内网机器的通讯。然后检查到校园网内网的机器通不通，然后再检查到国内机器通不通，最后再检查到境外机器通不通，像剥洋葱一样一层层剥开。对每一层，可以先检查 ping，然后再检查特定端口或者 curl, curl 不通则先检查 DNS 再检查其他过程。

最后是忠告：晚上八点以后不要折腾网络。

## 常用工具

```bash
ping <ip地址/域名>
# 用来快速检查是否能到达目的 ip
# 或者快速确定 (DNS正常 && 能到达目的 ip)
# 若目的地的防火墙 ban 了 ICMP 报文就没用了
```

```bash
traceroute <ip地址/域名>
# 用来查看到达目标地址的路径
# 可以检查到网络包是在哪里被丢掉的
# windows 下是 tracert
```

> 高级版 ping+traceroute: `mtr`, 详见: <https://vitux.com/how-to-use-the-linux-mtr-command/>
> 
> 不过一般发行版都没有自带

```bash
curl -v <ip地址/域名>
# -v 表示 verbose, 此命令让 curl 详细输出 HTTP 通讯过程
# 在 ping 通目的地的情况下拿来检查目的网站的网页有没有问题
```

```bash
telnet <目的IP> [<端口号>]
netcat <目的IP> <端口号>
# 可以快速判断端口有没有开
# 它们会直接往目标套接字倒原始数据 (或者收原始数据)
# 偶尔可以拿来学习 TCP 层上的应用层协议
# netcat 可以直接打缩写 nc
```

```bash
ss -tulp
netstat -tulp
# 查看本机开了哪些端口, 并且显示绑定/监听这些端口的程序
# Windows 下只有 netstat, 而且不支持上面那些参数, 只能拿来看端口有没有开
```

```bash
nslookup <域名>
# 用来检查 DNS 有没有坏掉
```

```bash
host <域名>
host -v <域名>
# 用来获得更详细的 DNS 应答信息
# 第二个命令可以看到 DNS 请求是往哪发, 谁回复的
# 四舍五入是 DNS 版 traceroute
# Windows 上的 host 是打印计算机信息...
```

```bash
ip addr
ip addr show <接口名(eth0)>
sudo ifconfig
# 查看当前系统的网络接口
# 一个网络接口四舍五入就是一个虚拟的网络设备, 比如说虚拟的网卡, 虚拟的交换机 blabla
# 当你的机子上有一大堆 docker 创建的垃圾接口时, 可以通过第二条命令指定要输出哪个接口的信息
# ifconfig 比较好看, 但是可能不是所有机子上都有
# 在 Windows 上可以用 ipconfig 得到差不多的效果
```

```bash
sudo route
ip route show
# 两个命令都可以查看路由表, 前者要 root, 后者不用
# 路由表就是一个规定了特定 ip 地址(网段)要往哪个接口/网关走的表
```

```bash
sudo arp -a
# 查看 arp 表, 就是一个写着哪个 IP 对应哪个 Mac 的表
# Windows 上也可以用
```

> 高级版监听工具: `tcpdump`, 可以把网卡收到的所有包都收集起来，然后结果可以丢进 WireShark 分析
> 
> 用法: `sudo tcpdump -i <interface 名字> -w <保存的文件名>`, 然后把生成的文件拖进 WireShark 就可以随心所欲地分析了。
