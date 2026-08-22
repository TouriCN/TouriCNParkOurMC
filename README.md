# actionsMinecraftBetter

本项目由 [Briiqn/Actions-Server](https://github.com/Briiqn/Actions-Server) Fork 而来，在其基础上进行了重构和扩展。

## 使用方式

Fork 后配置以下两个 GitHub Secrets，即可通过 Actions 页面手动触发运行：

| Secret | 说明 |
|--------|------|
| FINE_GRAINED_PAT | Fine-grained Personal Access Token，需 Contents 和 Issues 的 Read/Write 权限 |
| PLAYIT_SECRET | 首次运行留空。第一次启动会输出 playit.toml 内容，填入后再次运行即自动连接 |

首次运行流程：
1. Actions -> Minecraft Server -> Run workflow
2. 日志中出现 playit 认领链接，浏览器打开并完成认领
3. 认领成功后日志输出 playit.toml 内容，复制并填入 PLAYIT_SECRET
4. 再次 Run workflow，服务器启动，玩家通过 playit 分配的地址加入

## 关停行为

工作流接近 6 小时限制时，GitHub 发送 SIGTERM。trap 处理依次执行：

1. 从 server.properties 读取 rcon.password，通过 mcrcon 向所有在线玩家发送踢出消息
2. 等待 5 秒确保消息送达
3. 执行 git commit + push 保存世界数据
4. 打包 logs 目录为 artifact 上传

主循环中每 10 分钟也会自动 commit 并 push 一次，正常情况最多丢失 10 分钟进度。

## 架构说明

- canvas.jar 和 playit-linux-amd64 在运行时从官方下载，不进入仓库（符合 Mojang EULA 和 playit 分发条款）
- 所有敏感凭证（PAT、playit 认证信息）仅存在于 GitHub Secrets 中，仓库内不可见
- RCON 密码存储在 server.properties 中，只监听 127.0.0.1，外部无法访问
- 插件全部为开源协议（MIT / GPL-3.0 / MPL-2.0），已包含在 plugins/ 目录中
- 世界数据通过 git 同步，非实时数据库

## 许可证

CC0 1.0 Universal。详见 LICENSE。
