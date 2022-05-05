# 使用 vscode-remote 开发

> 在介绍了如何 ssh 进入集群后
>
> 接下来我们介绍如何使用 vscode-remote 进行开发

## 1. 安装 Remote - SSH 插件

如图所示

![plugin](https://gitee.com/villard/wiki-images/raw/master/vscode-remote/vscodeplugin.webp)

## 2. 添加 Remote Server

左侧会多出一个 **远程资源管理器** 的图标

在远程资源管理器中

选择添加

![addserver](https://gitee.com/villard/wiki-images/raw/master/vscode-remote/ssh_add_host.webp)

在其中输入之前成功登录的 ssh 命令

![](https://gitee.com/villard/wiki-images/raw/master/vscode-remote/entersshcommand.webp)

例如

```shell
ssh username@hostname -p peer
```

成功后会要求输入密码，密码为 **hpc 平台账号登录密码**

!!! question "不在学校？"
    若需使用 EasyConnect 代理，请参考 [**公共集群的基本使用方法**](../login/) 教程中的 ssh 代理方式

## 3. 添加文件夹

点击添加文件夹

可先在用户目录创建新文件夹，避免直接使用用户根目录开发

![success](https://gitee.com/villard/wiki-images/raw/master/vscode-remote/success.webp)

![addfolder](https://gitee.com/villard/wiki-images/raw/master/vscode-remote/addfolder.webp)

## 4. 终端

按键盘 ++ctrl+grave++ 呼出终端，这样就可以使用 remote bash 了

++grave++ 键在 ++1++ 的左边，++tab++ 的上面
