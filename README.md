# actionMinecraftBetter

本项目由 [Briiqn/Actions-Server](https://github.com/Briiqn/Actions-Server) Fork 而来，在其基础上进行了重构和扩展。

## 主要功能

- **解压部署（blank-deploy.yml）**：将服务器文件打包为 `server.zip` 或 `halfserver*.zip` 上传到仓库任意目录，通过 Deploy 工作流手动输入包名解压到指定目录并删除。`server.zip` 和 `halfserver*.zip` 已被加入 `.gitignore`，不会进入 git 历史，仅作为一次性传输容器使用。支持以下方式：
  - `server.zip`：将插件、世界数据、配置文件等打包为 `server.zip` 上传到目标目录，Deploy 时输入包名即可解压。
  - `halfserver*.zip`：当单个目录内容超过 GitHub 网页单文件 25MB 限制时，可将其拆分为多个 `halfserver1.zip`、`halfserver2.zip`、…、`halfserverN.zip` 分批上传到目标目录。Deploy 时输入多个包名（空格分隔）即可依次解压。
- **服务端自动下载**：默认使用 [Leaf](https://github.com/Winds-Studio/Leaf) 服务端核心（26.2 实验性构建），运行时通过官方 v2 API 动态获取最新构建并下载 `leaf.jar`，不进入仓库。
- **隧道自动配置**：集成 playit.gg 隧道，首次运行交互认领，后续自动连接。
- **优雅关停**：被 SIGTERM 关闭时通过 RCON 踢出所有在线玩家并附带说明消息，然后执行紧急保存和 git push。
- **定时存档**：每 10 分钟自动 commit + push 世界数据，最多丢失 10 分钟进度。
- **到期提醒**：运行接近 6 小时限制时自动创建 Issue 提醒手动重启（调试模式下跳过）。
- **调试模式**：通过 `workflow_dispatch` 的 `debug` 参数开启，关闭时跳过所有玩家通知和 Issue 提醒，方便开发测试。

## 使用方式

Fork 后进入仓库 Settings -> Secrets and variables -> Actions，配置以下两个 Secret：

| Secret | 说明 |
|--------|------|
| FINE_GRAINED_PAT | Fine-grained Personal Access Token，需 Contents 和 Issues 的 Read/Write 权限 |
| PLAYIT_SECRET | 首次运行留空。第一次启动会输出 playit.toml 内容，填入后再次运行即自动连接 |

初始文件部署（任选其一或组合）：

- 将服务器文件打包为 `server.zip` 上传到仓库任意目录
- 将大目录拆分为多个 `halfserver*.zip` 分批上传到目标目录
- 直接通过网页逐个创建配置文件和插件 jar

首次运行流程：

1. Actions -> Deploy (unpack) -> 在 `packages` 输入要解压的包名（如 `server.zip` 或 `halfserver1.zip halfserver2.zip`），留空则自动扫描所有 zip -> Run workflow
2. Actions -> Minecraft Server -> Run workflow（直接点击，不勾选 debug）
3. 查看运行日志，找到 CLAIM URL 并在浏览器中打开，完成 playit 隧道认领
4. 认领成功后日志中会输出 playit.toml 的完整内容，复制
5. 回到 Secrets 页面，新建 PLAYIT_SECRET 并粘贴内容
6. 再次 Run workflow，服务器启动，玩家通过 playit 分配的地址加入

## 服务端

默认使用 [Leaf](https://github.com/Winds-Studio/Leaf) 服务端核心。工作流运行时通过官方 v2 API 动态获取最新构建并下载 `leaf.jar`，不进入仓库。如需稳定版，将工作流中的版本线从 `26.2` 改为 `1.21.11`（最新稳定构建 #174）即可。如需更换其他服务端（如 Paper、Purpur），修改工作流中的下载链接和 Java 启动参数即可。

## 自定义

工作流文件 `.github/workflows/blank.yml`（服务端运行）和 `.github/workflows/blank-deploy.yml`（解压部署）包含所有运行逻辑，可根据需求自行修改：

- **JVM 参数**：调整内存分配（-Xms/-Xmx）、GC 策略、实验性 VM 选项等
- **同步间隔**：默认每 10 分钟执行一次 git commit + push，修改 `sleep 600` 的值即可
- **到期提醒阈值**：默认在运行 5 小时 40 分钟后创建 Issue 提醒，可调整 `20400` 秒的判定值
- **RCON 踢人消息**：修改 trap 中 `mcrcon` 命令后的字符串内容
- **服务端核心**：更换下载 URL 和启动参数

修改后提交到仓库，下次 Run workflow 即生效。

## 调试模式

工作流支持通过 `workflow_dispatch` 的 `debug` 布尔参数开启调试模式。勾选后运行：

- 被 SIGTERM 关闭时不会踢出在线玩家，也不会创建"服务器即将关闭"的提醒 Issue
- 运行接近 6 小时限制时不会创建提醒 Issue，仅在日志中输出一行提示
- 紧急保存和 git push 仍正常执行，世界数据安全不受影响

适合在修改工作流或测试新功能时使用，避免频繁触发提醒 Issue 干扰开发。

## 关停行为

工作流接近 6 小时限制时，GitHub 发送 SIGTERM。trap 处理依次执行：

1. 从 server.properties 读取 rcon.password，通过 mcrcon 向所有在线玩家发送踢出消息
2. 等待 5 秒确保消息送达
3. 执行 git commit + push 保存世界数据
4. 打包 logs 目录为 artifact 上传

主循环中每 10 分钟也会自动 commit 并 push 一次，正常情况最多丢失 10 分钟进度。

## 架构说明

- leaf.jar 和 playit-linux-amd64 在运行时从官方下载，不进入仓库
- server.zip 和 halfserver*.zip 通过 blank-deploy.yml 解压后删除，不进入仓库历史
- 工作流文件 `.github/workflows/blank.yml` 和 `.github/workflows/blank-deploy.yml` 本身进入仓库，受版本管理
- 所有敏感凭证（PAT、playit 认证信息）仅存在于 GitHub Secrets 中，仓库内不可见
- RCON 密码存储在 server.properties 中，只监听 127.0.0.1，外部无法访问
- 世界数据通过 git 同步，非实时数据库

###### © 2026 TouriCN|MIT License
