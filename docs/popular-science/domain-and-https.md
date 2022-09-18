---
title: "购入域名之后..."
tags:
- 域名
- HTTPS
- 服务器管理
- Docker

date: 2022-02-03 10:00:00 +08:00
---

## 域名是什么？

**域名**, Domain Name，是一个可以写在 url 里的名字。

我们知道，当下互联网上 (同一个网络内) 的实体是靠 IP 地址来唯一标识的。完成域名到 IP 地址转换的系统称为 DNS. DNS 的主要构成部分是大大小小的 DNS 服务器。DNS 服务器接受客户的访问，返回域名对应的 IP 地址; 它是多级缓存的分布式数据库，通过将请求“转发”给其他服务器实现了具有层次结构的名称空间。

### 如何“拥有”一个域名：理论

那么，在这个庞大的系统里，我们要如何插入一条属于我们的记录呢？

答：**给钱。**

尽管 DNS 是可以是分布式的系统，但显然域名的管理不能是分布式的 (真的吗？). **ICANN**, 互联网名称与数字地址分配机构 (Internet Corporation for Assigned Names and Numbers), 一个位于美国加州的非营利社团，负责域名与 IP 地址的分配。ICANN 直接管理顶级域名 (TLD, top-level domain), 并且适当地把一些顶级域名分配给其他组织，譬如：

- 基础建设 TLD (`.arpa`)
- 国家及地区 TLD (ccTLD): 如 `.cn`, `.uk` 等
- 国际 TLD (IDN)
- 通用 TLD (gTLD): 如 `.com`, `.org`, `.net` 等

这里有一个 [TLD 列表](https://en.wikipedia.org/wiki/List_of_Internet_top-level_domains).

ICANN 是非营利组织，它把一些顶级域名的注册服务承包出去，那些承包商可就不是非营利的了。他们负责维护域名的 whois 信息 (就是一大堆你的个人信息，为了在法律上证明你对该域名的所有权), 保证域名的唯一性，也许还为倒卖域名的其他公司提供支持 :(. 所以，为了获得一个域名，我们需要前往某个域名注册提供商那里支付一笔费用，随后他们便会负责帮你去 ICANN 或者哪里为你注册一个域名。

### 如何“拥有”一个域名：实践

常见的外国域名注册提供商有：

- GoDaddy，狗爹，最大的一家，小贵，而且有点黑料
- Gandi
- Namecheap，顾名思义，主打便宜，还送邮箱服务，但是貌似不能支付宝
- ResellerClub，印度公司，Web 页面很拉，但是便宜，而且是亚洲龙头

国内的提供商有：

- 万网
- DNSPod
- 新网
- [趣域网](https://www.quyu.net/), 免实名，但是页面看起来很拉
- 阿里云腾讯云这些貌似也可以买，而且说不定有备案服务

国内的提供商大多需要实名/备案，但国内的基本上都比国外便宜 (国外赚美刀花美刀怎么能一样呢). 况且很多国外的提供商都提供 **whois 信息保护**, 信息泄露可以少一点。

上面的信息都是聊天聊来的。出于贫穷 + 不想实名，我选择在 ResellerClub 购入了一个 `.top` 域名 (就是现在这个啦). 根据群友说法，`.top` 域名注册商是国内某家公司，所以人力成本很低，所以便宜 :(. 我的 id 字母混数字，就更加便宜，**10 年只需要 277 CNY**.

在 RC 的网站上选好域名，注册账号。注册账号要填很多信息，这些信息可能在打官司的时候有用，但鉴于我又不会拿这个域名来干什么，我就直接填了我们学校的信息 (笑)

地址倒是卡了我一下 -- 虽然帮李华写了很多信，但是没写过英文地址啊。咨询群友意见之后，我选择了全拼音 (包括“Lu，路”之类的), 首字母大写，逆序 (地址由小到大，逗号分隔) 的形式。

最后记得把 **域名隐私保护** 那个按钮打开就可以了。

注册了域名，接着就是要往 DNS 里插信息了。DNS 中维护了几类记录：

- 主机记录 (A 记录): 记录域名对应到哪个 IPv4 地址。
- 别名记录 (CNAME 记录): 记录域名对应到哪个域名。(“不要来烦我，去找这个域名！”)
- IPv6 主机记录 (AAAA 记录): 记录域名对应到哪个 IPv6 地址。
- 域名服务器记录 (NS 记录): 用来指定该域名由哪个 DNS 服务器来进行解析。每个注册的域名都可以由多个 DNS 域名服务器来进行解析，并且在注册的时候就会默认被塞几条域名注册商提供的 NS 记录。一般这个记录长这样子：`ns1.domain.com`、`ns2.domain.com`…. 这个记录只有域名注册商那里可以改，所以如果你能改这个记录，那你必然是域名的所有者。
- 纯文本记录 (TXT 记录): 就是简单地把 DNS 当“字典”用; 这种记录一般用来验证你是域名的管理者：只有能访问 NS 记录中的 DNS 服务提供商的人才能添加 DNS 记录。

因为 RC 默认的 DNS 记录管理很拉，所以我们选择使用 Cloudflare 来管理我们的 DNS 记录。首先注册一个 Cloudflare 账号，然后把刚刚买的域名填进去，接着按照指示去 RC 里修改 NS 记录把管理权交给 Cloudflare 就可以了。接着我们就可以在 Cloudflare 里加记录了。(或者，让程序通过 API 来往里面加记录)

有了域名还不够，现在都 2022 年了，怎么还能不上 HTTPS 呢。等等，HTTPS 是啥？

## 为什么要申请 SSL 证书 / 上 **HTTPS**?

-- 一张 SSL 证书能干什么呢？

-- 能证明你访问这个域名访问到的，就是申请证书的主机。

> 当然一张高级证书里可以塞进别的信息，证明该域名的一些其他信息。
>
> 像上面那样的，只是证明访问域名访问到的是特定主机的证书，叫 DV 证书，Domain Validated certificates
>
> 往证书里塞入其他信息，比如域名持有者的公司信息之类的，就可以得到高级证书，比如 OV 证书或者是 EV 证书。 (organization validated (OV) and [extended validation (EV) certificates](https://www.keyfactor.com/blog/what-are-extended-validation-certificates-and-are-they-dead/))
>
> 偶尔你可以在浏览器的小锁上看到某某公司的名字，这就是高级证书了 (笑)

### SSL 证书是如何起作用的？

SSL 证书的验证流程大概是这样的：

1. 浏览器解析域名，做 DNS，拿到 ip，访问 ip
2. 首先从那个 ip 里收一张证书回来
3. 然后拿证书开始加密通信

验证过程中可能出现的问题与解决方法：

- 我要是在 DNS 部分动手脚，访问到假 ip，给假证书，怎么办？
    - DNS 很难动手脚
    - 证书的颁发是需要权威机构 (CA) 的认证的，并且需要检验你是不是域名的管理者; 域名信息是写在证书里的，所以不能通过简单的让浏览器访问假 ip 来骗
- 证书在传输过程中被截了怎么办？
    - 证书也是被签名的，浏览器收到证书会先检验你这证书是不是真的。
    - 检验证书的证书，检验检验证书的证书，一直套娃，到最后的根证书，形成了一条相继确保安全的证书链。这条链的根节点，根证书，其私钥被全世界最权威的机构以最安全的形式保管，公钥被直接放在常见的操作系统里，安装就有。
    - 要想修改证书就得打破这条链，但是这要是被打破了，那天就塌了，也轮不到我们来担心了 :)

所以，有了 SSL 证书，至少中间人攻击就难搞了; 数据的传输也被证书加密，从而很难被嗅探到了。

### 那如何申请一张证书呢？

要申请证书，自然要先 **买服务器** (钱无预定). 为了省心我直接买了 Vultr 新的韩国机房的 5$/mo 的机子，1g1c 跑点小服务足以。IPv4 ping 100-150ms, IPv6 ping 250ms 左右，套层代理就很快了，所以没大问题。

然后就 **是找 ~~便宜~~ 靠谱的 CA** 来给你签一份证书。作为穷人自然是选择给免费发证书的 CA: **Let’s Encrypt**. Let’s Encrypt 注册的证书一张管 90 天，只能在倒数 30 天之前续签; 同时可以选择签通配符证书，也就是一张证书证掉所有域名 (比如*.origami404.top), 这样子就可以放肆使用子域名了。

申请证书的方式有很多种，比如你可以派一个人去 CA 的总部把证书抄回来然后打进去，也可以通过 **ACME 协议** (这协议就是 Let’s Encrypt 那班人弄出来的来着) 来自动申请。ACME 现在的版本是 ACME v2，不跟 v1 向后兼容，主要比 v1 好的地方就是支持签通配符证书。

### ACME 的原理

在 ACME 协议里，我们有两个程序，ACME 客户端跟服务端。服务端运行在 CA 的服务器上，客户端运行在需要申请证书的服务器上。当安装客户端时，客户端会要求你输入你想申请证书的域名与 CA. 随后其：

1. 生成一个密钥对 (也就是一个公钥跟对应的一个私钥), 然后与 CA py，通过 HTTPS 链接 (这个链接此时由 CA 的 SSL 证书保障) 来跟 CA share 这个对。这个密钥对又叫 authorization key pair (下文简称 AK)
2. CA 生成一个 HTTP/DNS 挑战 (issue challenges (DNS or HTTP)) 来验证客户端对其宣称的域名的控制权。
3. 为保安全，CA 还会发送一个随机数，叫 nonce，给客户端。客户端必须用 AK 的私钥给它签名，然后发回去，以证明自己手上确实有私钥。

HTTP 挑战认定了这样一个事实：如果在 DNS 里，你这个域名对应到的 ip 就是发出请求的服务器的话，那你就是“控制”着这个域名。因此，HTTP 挑战的主要流程如下：

1. CA 给客户端发一个 token
2. 客户端收到 token，创建一个文件，里面包含了这个 token 跟 AK 的指纹
3. 客户端监听 80 端口，准备把文件送出去，同时通知 CA 说我好了
4. CA 访问域名，解析到 ip 地址，访问那个 ip 地址的服务器的 80 端口，尝试收文件。一旦收到正确的文件了，就认为挑战成功。

DNS 挑战则认为，如果你能往 DNS 里插这个域名关联着的记录，那你就“控制”着这个域名。于是，DNS 挑战的主要流程如下：

1. CA 给客户端发一个 token
2. 客户端收到 token，然后生成一个包含了 token 和 AK 指纹的一个字符串 (内容同 HTTP 挑战里的文件内容)
3. 客户端往 DNS 里加入一条 TXT 记录，记录的值就是这个字符串，然后通知 CA 来收
4. CA 通过 DNS 查询得到 TXT 记录的值，检查是不是自己发的。如果值正确了，就认为挑战成功。

HTTP 挑战的主要坏处就是它要占用 80 端口一段时间 -- 在这段时间里我们的服务就得停一会，挺烦的。而 DNS 挑战的主要坏处就是它需要让客户端这个程序能操纵你的 DNS 管理商。对于正常的应用而言，一般都是选择 DNS 挑战，毕竟一般来讲我们都不想让网站每两个月就当掉 10-15 分钟，并且主流 DNS 管理商 (比如 Cloudflare) 都提供了对应的 API 供 ACME 客户端使用。但我选择了 HTTP 挑战 -- 因为 Caddy 可以自动帮我处理这件事。

这就不得不讲讲如何用一台服务器跑多个应用了 -- 毕竟 80/443 端口只有一个，不是吗？

## 用一台服务器跑多个应用

### 理论：子域名，反向代理，以及边缘网关

在网络里，IP 地址用于确定某一台主机，而端口号则可以用来确定主机上哪一个程序提供服务。系统把端口号分给不同的应用，在收到数据包之后根据端口号把数据包转给不同的应用处理，于是我们就可以在一台服务器上同时运行多个需要访问网络的应用了。需要访问不同应用，就在地址上加上端口号就可以了，比如这样：`127.0.0.1:3000`.

当我们在浏览器里输入域名的时候，浏览器就拿域名去找 DNS，讨回来一个 IP 地址，然后访问。但问题是，浏览器拿到了 IP 地址，会访问它哪个端口呢？

一个直觉的想法是，DNS 返回的 ip 地址应该是带着端口的，这样浏览器不就知道了嘛。然而 DNS 并没有这种功能 -- DNS 在设计时就只是负责 **把域名映射到 IP 地址**, 而 **不** 负责把域名映射到 IP 地址 + 端口号。

> DNS 在设计的时候为啥不把端口号加上呢？我猜测有二：
>
> - 端口号相比 IP 地址，变动得更加频繁，加进去会加重 DNS 的负担
> - 设计 DNS 的人忘记了 :(

所以浏览器拿到 IP 地址之后，它也不知道该访问啥; 首先它会看看你打进去的 URL 是不是包含端口 (有冒号), 有它就往那个 IP 发带着那个端口号的包; 没有的话，浏览器就 **直接假设你想访问 80 端口** (如果你的 URL 是 http 开头的，走 HTTP 协议的话), 然后往那个 IP 发带着 80 端口号的包。

现在假如我们想要这样子：

- 访问 A.origami404.top 就访问到 A 服务
- 访问 B.origami404.top 就访问到 B 服务
- 访问 C.origami404.top 就访问到 C 服务

要怎么办呢？一个简单的解决办法是，买三台服务器，把这三个域名绑定到三个 IP 上。然而我并没有钱。

俗话说，“计算机科学里的任何问题都可以通过加一个抽象层解决”. 既然操作系统可以看端口把收到的包分给不同的应用，我们也可以写一个应用，专门守住 80 端口，收到包之后解包，看看包里写着的“待访问域名”, 然后根据这个的值把包处理一下，然后跟对应的程序建立连接，把处理好的包发过去就好了。

像这种 **“守门口分包”** 的功能，叫 **反向代理**. 正向代理把服务端跟用户隔离，只让服务端看到自己; 反向代理也把用户端跟服务器隔离，但是只让用户看到自己。常见的可以用来当反代的软件有：

- Nginx: 很出名，很强大，很快。能均衡，反代，静态文件。但是配置文件看起来很复杂 (快进到图灵完备)
- Apache: 很老，不大快
- Haproxy: 只做均衡和反代，不能当静态文件服务器。配置上比 nginx 更像“网络配置”
- Caddy: 比较新，Go 写的; 自带一个 ACME 客户端，可以自动通过 HTTP 挑战续 SSL 证书; 配置文件写起来很友好
- Traefik: 很新，可以直接管理各种容器 (通过容器的 label 来动态地决定这包该发哪)

事实上大部分这些软件都能不止能做反代，还能搞负载均衡/静态文件服务器等，但无论如何做的都是“守门口分包”的工作：负载均衡就是把收到的包均分丢几个不同服务器去，静态文件服务器就是收包然后找文件发回去。

基于好奇跟懒，我选择了 **Caddy** 作为“守门员”. 理由就像上面提到的那样：

- 相对来讲很快的速度 (毕竟是 Go 写的), 对我这小户人家完全够用
- 可以 **自动申请/续签证书**, 不需要先把自己关掉，让出 80 端口，再用别的程序做 HTTP 挑战
- 配置文件 (Caddyfile) 很好写，Nginx 的配置文件直接劝退，抄都抄不懂 :(

偶尔，你也会看到上面的软件把自己叫做“边缘网关”. 这个边缘指的是提供服务的服务器集群，跟外部世界之间的边缘。这些软件接受所有来自外界的访问，将其转发，就好像一个坐在边界的网关一样，所以叫边缘网关。

### 实践：基于 Docker 的服务架构

如果一台服务器上跑多个服务的话，各个服务的安装/管理/配置将会是一场噩梦。幸好现在是 2022 年了，我们有了非常好用的容器化技术，以及还算可以的容器管理技术。这次我打算把所有服务都丢容器里边，同时充分利用 Docker 的 [network 功能](https://yeasy.gitbook.io/docker_practice/network), 尽可能少地暴露端口。

![服务器架构示意图](https://user-images.githubusercontent.com/29816865/158061389-54bc1234-5feb-45c7-9a7a-b1952ee237ce.png)

图中的每一个黑框框都代表一个容器，虚线框框代表一个 network，被框住的容器就代表其与该 network 相连。最外边代表整个 Docker 环境，只有 80/443 端口被暴露出来，连接到守门员 Caddy 的容器里。

我的管理方法是这样的，对每一项服务，比如 Outline，创建一个子文件夹，里面放一个 docker-compose.yml，描述了这个服务。如果有需要挂载出来的数据目录，或者挂载进去的配置文件，也都放在这个子文件夹里，方便备份和管理。大概目录就长这个样子：

```
.
└── services
    ├── caddy-main                  # Caddy 的目录
    │   ├── caddy-config               # 数据
    │   ├── caddy-data                 # 数据
    │   ├── Caddyfile                  # 配置，每次加新服务都要修改这个文件，增加反代规则，然后重启 Caddy
    │   └── docker-compose.yml         # Docker-compose
    ├── miniflux                    # Miniflux 的目录 (一个 RSS 服务)
    │   ├── data                       # Miniflux 的数据存放的目录
    │   └── docker-compose.yml         # Docker-compose
    ├── outline                     # Outline 的目录
    │   ├── docker-compose.yml         # Docker-compose
    │   ├── docker.env                 # Outline 的环境变量太多，单独拿一个文件出来设
    │   ├── init_minio.yml             # 用来初始化 MinIO 的命令，因为有关系，所以放在 Outline 这里
    │   ├── minio_main_data            # 数据
    │   ├── outline_db_data            # 数据
    │   └── redis.conf                 # 配置
    ............
```

然后 Caddy 如下配置：

```yml
# Caddy docker-compose.yml
version: "3"

services:
  caddy-main:
    image: caddy:2-alpine
    container_name: caddy-main
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      # 把各种配置文件挂进去，数据都挂出来
      - "./Caddyfile:/etc/caddy/Caddyfile:ro"
      - "./caddy-config:/config"
      - "./caddy-data:/data"

    networks:
      # 将 caddy-main 容器加入 service-net 之中
      - service-net
    
networks:
  # service-net 这个顶层网络我们在外部单独创建
  # 使其跟所有服务独立，便于管理 (不会说一不小心把哪个服务关了，所有服务都死了)
  service-net:
    external: true
```

```bash
# Caddyfile

# 对于任何访问到 80 端口的包，Caddy 都会把它们转发到对应域名的 443 端口去
# 也就是说相当于有一条 *:80 -> *:443 的转发规则
# 所以我们在下面只需要配置各个子域名的 443 端口的转发规则就可以了

# 对于访问 443 端口，目标域名为 outline.origami404.top 的包
outline.origami404.top:443 {
  # 这些包的日志将会被记录
  log {
    # 等级为 info
    level INFO
    # 记录在容器里的 /data/outline.log
    output file /data/outline.log {
      # 单个大小最大 10MB，保留最近的 10 个这样的文件，超过了就开始循环写入
      roll_size 10MB
      roll_keep 10
    }
  }

  # 设置给这个子域名进行 ACME HTTP 挑战的时候要交给 CA 的邮箱
  tls Origami404g@gmail.com

  # 启用基于 gzip 的压缩传输
  # 有可能会有问题，但我还没遇到 2333
  encode gzip

  # 设置反向代理 (reverse_proxy)
  # 把收到的包丢到地址 http://outline:3000 去
  # 因为我们提供服务的容器都在同一个 network, service-net 里
  # 所以我们可以通过容器的名字当 url 互相访问容器
  reverse_proxy outline:3000 
}

# 如果需要增加其他的服务，照猫画虎复制粘贴就可以了，只不过改改反代的目标地址而已
# 比如下面这堆，可以将对 rss.origami404.top 的访问，反代到对应的容器 miniflux 的 80 端口上。
rss.origami404.top:443 {
  log {
    level INFO
    output file /data/miniflux.log {
      roll_size 10MB
      roll_keep 10
    }
  }

  tls Origami404g@gmail.com
  encode gzip

  reverse_proxy miniflux:80
}
```

我看到貌似有可以直接通过给容器加 label 的形式定义 Caddy 的转发规则 (而不用写在 Caddyfile 里), 从而真正实现动态增加服务 -- 然而我现在还不会

然后我们稍微地配置一下 Miniflux 跟 Outline:

此处不会详细介绍如何配置它们，目的主要是介绍服务的架构

```yml
# Miniflux docker-compose.yml
version: "3"

# 基本配置照抄官网
services:
  miniflux:
    image: miniflux/miniflux:latest
    container_name: miniflux
    # 注意这里我们不需要写任何的 ports
    # 这个 ports 是用来把端口绑定到主机上的
    # 而我们根本不需要让外界能通过这个端口访问到该容器
    # 我们只需要让 Caddy 能访问到这个容器就可以了
    # 然而要做到上面这点，我们只需要把容器们放到同一个网络就可以了
    # 同一个网路的容器之间默认是都可以互相访问的
    #ports:
    #  - "80:80"
    depends_on:
      - db
    environment:
      - DATABASE_URL=postgres://miniflux:secret@db/miniflux?sslmode=disable
      - RUN_MIGRATIONS=1
      - CREATE_ADMIN=1
      - ADMIN_USERNAME=管理员用户名
      - ADMIN_PASSWORD=管理员用户密码
    healthcheck:
      test: ["CMD", "/usr/bin/miniflux", "-healthcheck", "auto"]
    # 在这里对于网络的配置要注意加入这个 docker-compose 创建的默认网络
    # 同一个 docker-compose.yml 里创建的容器可以互相访问的原因，就是它们在同一个网络里
    # 执行 docker-compose up 的时候，首先创建一个网络 xxx_default，然后再创建容器，把容器加入到网络里
    # 这个网络在 docker-compose.yml 里可以通过 default 来访问  
    networks:
      - service-net
      - default       # 就是这个
    
  db:
    image: postgres:latest
    container_name: miniflus_postgres
    environment:
      - POSTGRES_USER=miniflux
      - POSTGRES_PASSWORD=secret
    volumes:
      - ./data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "miniflux"]
      interval: 10s
      start_period: 30s

networks:
  # 同样地，指明这个网络是外部的，这样执行这个的 docker-compose up 的时候
  # 就不会尝试去创建一个网络 service-net，而是去找一个已经存在的网络 service-net 了
  service-net:
    external: true
```

```yml
# Outline docker-compose.yml

version: "3"
services:
  # 基本上就照抄官方的
  outline:
    image: outlinewiki/outline
    container_name: outline
    env_file: ./docker.env
    restart: always
    # 同样不需要暴露端口
    # ports:
    #   - "3000:3000"
    # 同样网络记得加入 default，要不然就不能访问数据库了
    networks:
      - service-net
      - default
    depends_on:
      - postgres
      - redis
      - storage

  redis:
    image: redis
    container_name: outline_redis
    env_file: ./docker.env
    restart: always
    # 把数据啊配置啊全部挂出来，不要用 docker volume
    volumes:
      - ./redis.conf:/redis.conf
    command: ["redis-server", "/redis.conf"]
    # 不需要对外提供服务的容器根本不需要加入到 service-net 网络中
    # 只需要加入到这个 docker-compose 定义的 default 网络就可以了

  postgres:
    image: postgres
    container_name: outline_postgres
    env_file: ./docker.env
    restart: always
    volumes:
      - ./outline_db_data:/var/lib/postgresql/data

networks:
  service-net:
    external: true
```

要运行的话，就只需要简单地创建网络，并且启动所有 docker-compose 就可以了。

```bash
$ docker network create service-net # 创建网络
$ cd caddy-main && docker-compose up -d    # 把容器都创建起来
$ cd ../outline && docker-compose up -d
$ cd ../miniflux && docker-compose up -d 
```

### 子域名注册

现在我们的服务器已经准备好接收访问各个子域名的包了，是时候来修改 DNS，使所有子域名都被解析到我们的服务器上了。在之前的文章里，我们已经把域名的 NS 记录改成了 Cloudflare. 登录 Cloudflare 检查一下，发现 CF 很贴心地直接帮我们加入了通配符形式的 DNS 记录，不需要我们再进一步配置了。

那么基本的配置到这里就结束了。

> 原地址：https://blog.origami404.top/post/2022-02-domain-and-server-management/