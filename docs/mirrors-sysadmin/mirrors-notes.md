# 镜像站记载

## 目录清单

- `/usr/bin/tunasync*`：tunasync 本尊和 tunasynctl
- `/etc/tunasync/*`：工人和经理配置
- `/var/tunasync/log/`：日志，failed 时可 debug
- `/var/tunasync/util/`：杂项工具
  - `gen_res_link`：按照 `gen_res_link.ini` 生成资源下载清单到 `/var/www/public/static/`
  - `backup`：备份镜像站相关内容到 `/backups/backup-*.tar.gz`
- `/var/tunasync/script/`：自定义的同步脚本
- `/etc/systemd/system/tunasync-*.service`：tunasync 需要的 systemd unit
- `/var/www/public/`：镜像站前端静态网站放置处
- `/var/www/help/`：私有化 Mirrorz Help 静态网站放置处
- `/etc/nginx/conf.d/mirrors.conf`：镜像站 Nginx 配置
- `/etc/nginx/conf.d/mirrors-help.conf`：私有化 Mirrorz Help Nginx 配置

## 其他清单

- 工人名字是 main_worker
- tunasync 通信位于 localhost:14242

## imbuhui，怎么办

请 Google 搜索 tunasync 等获取特定的文档和教程。
