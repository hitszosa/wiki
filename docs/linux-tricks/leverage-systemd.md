---
title: 充分利用 systemd
date: 2022-09-18
---

## 前言

> The good, the bad and the `PID 1`.

systemd 不是一个不可拆解的庞然巨物，不是一个充斥着 bugs 的粗糙软件，而恰恰相反——是一个高度模块化、稳健而实用的基础软件。

请参见 Lennart Poettering 本人澄清几乎所有关于 systemd 误解的一片文章：<https://0pointer.net/blog/projects/the-biggest-myths.html> 。

~~是的，Peottering 是果蛆。~~

## 鸟瞰

systemd 仿自 [launchd](https://www.launchd.info)，但是经过了长期的发展，已经根据 Linux 内核的特性衍生出了一些 launchd 所不具有的功能，也早已不再以仿造 launchd 为目标了。

尽管如此，对于我们用户而言，很多设计与交互逻辑依然是十分相似的。因此，我们不妨以对比作为切入点，鸟瞰一下「庞大的」systemd 里面究竟有些什么。

| 模块 | systemd | launchd |
| -- | -- | -- |
| Daemon 生命周期管理 | systemd、**systemctl** | launchd、launchctl |
| 定时任务 | systemd、**systemctl** | launchd、launchctl |
| 日志管理 | journald、**journalctl** | (not available) |
| 网络操作 | networkd | (not available) |
| 登录管理 | logind、**loginctl** | (not available) |

需要注意的是，[Darwin](https://en.wikipedia.org/wiki/Darwin_(operating_system)) 不再将一些功能归属于 launchd 这个概念中，而 systemd 中的这些模块也是「可拆卸」的（甚至 systemd 核心里面的功能都是模块化的、可剥离的）。

systemd 的 Daemon 生命周期管理，包含以下几个核心功能（对于 launchd 也适用）：

- 初始启动（kickstart，或者 init）守护进程（daemons，可为任意可执行文件）
- 按需（on demand）启动、终止、重启守护进程
- 处理守护进程的异常
- 有计划地操作守护进程
- 维护守护进程的依赖关系
- 调试守护进程，以及提供日志（logging）功能

事实上，即使是 [Windows NT](https://en.wikipedia.org/wiki/Architecture_of_Windows_NT) 上的「服务管理」，核心功能也与之十分相似 ~~（甚至有人认为 launchd 是受 NT 启发的）~~。
