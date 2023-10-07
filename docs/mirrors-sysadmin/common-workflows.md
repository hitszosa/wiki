# 常见工作流程

## 排查同步问题

1. 发现有镜像同步状态为 failed
2. 进入镜像站，使用 `tail -30 tail -30 /var/tunasync/log/<mirror>/latest` 查看日志。其中 `<mirror>` 为欲查看的镜像名
3. 通过日志定位到问题，并尝试解决
4. 使用 `tunasynctl` 重新同步对应的镜像，命令形如 `tunasynctl restart --worker main_worker <mirror>`。其中 `<mirror>` 为欲重新同步的镜像名
5. 再次查看日志确认问题已解决

## 下架镜像

1. 在 `/etc/tunasync/worker.conf` 中删除对应的条目
2. 使用 `tunasynctl` 重载配置文件，命令形如 `tunasynctl reload --config /etc/tunasync/worker.conf --worker main_worker`
3. 使用 `tunasynctl` 关闭对应的镜像同步，命令形如 `tunasynctl disable --worker main_worker <mirror>`。其中 `<mirror>` 为欲下架的镜像名
4. 使用 `tunasynctl` 冲刷对应的镜像，命令形如 `tunasynctl flush`

## 添加新闻

1. 确保能科学上网
2. 进入 `hitszosa/mirrors-frontend` 仓库
3. 确保 `main` 分支为最新
4. 切换到 `prod` 分支
5. 执行 `git rebase main`，可能需要自己处理冲突
6. 在 `content/news` 目录下编写新闻，格式请参考该目录下的其他文件
7. 执行 `yarn install`
8. 执行 `yarn generate`，生成后的网站位于 `.output/public/`
9. 使用 `rsync` 将网站部署到服务器，命令形如 `rsync -r ./.output/public/* <user>@10.249.8.102:/var/www/public`。其中 `<user>` 为您在服务器中的用户名
10. 最后，确保网站正常后将修改推送到 GitHub。由于潜在的 rebase 操作，所有可能需要使用 `git push --force`。

## 更新镜像站帮助网站

1. 确保能科学上网
2. 进入 `hitszosa/mirrorz-help` 仓库
3. 打开 `reference/src/routes.json` 并找到想要添加到条目
4. 将上一步找到的条目复制并粘贴到 `override/mirrorz-help/src/routes.json` 中的相应位置，最好按字典序放置，注意 JSON 格式规范（如列表末尾不能有多余的空格）
5. 执行 `make prod`
6. 使用 `rsync` 搬迁生成后的网站，生成后的网站位于 `mirrorz-help/out/`，命令形如：`rsync -r mirrorz-help/out/* <user>@10.249.8.102:/var/www/help`。其中 `<user>` 为您在服务器中的用户名
7. 最后，确保网站正常后将修改推送到 GitHub