# OpsDeck 使用文档

新用户建议先阅读 [安装与部署](installation.md) 和 [界面图解](ui-guide.md)。
主程序发布仓库为 [xiarenyyds/OpsDeck-Rust](https://github.com/xiarenyyds/OpsDeck-Rust/releases)，Docker Hub 为 `xrbzy/opsdeck`。

## 目录

- [简介](#简介)
- [快速开始](#快速开始)
- [首次初始化与登录](#首次初始化与登录)
- [主要功能](#主要功能)
- [当前已知限制](#当前已知限制)
- [Telegram 交互机器人](#telegram-交互机器人)
- [代理与授权边界](#代理与授权边界)
- [在线更新](#在线更新)
- [环境变量](#环境变量)
- [内置使用文档](#内置使用文档)
- [常见问题](#常见问题)

## 简介

OpsDeck 是一个基于 Rust 与 Vue 3 的多账户操作控制台，当前主要用于管理：

- 『色花(98)堂』与『2048核基地』论坛账号
- 登录、签到、回帖和定时任务
- 回帖内容、版块、黑名单与 AI 回复
- 账号代理、全局系统代理与 CloudflareBypassForScraping
- 插件、文件、下载器、通知和运行日志
- Docker 与二进制部署的在线更新

后台使用 Cookie 会话登录，首次创建管理员账户时必须绑定并验证 Authenticator 2FA。

## 快速开始

### 方式一：Docker 部署

```bash
docker pull xrbzy/opsdeck:latest

docker run -d \
  --name opsdeck \
  -p 13113:13113 \
  -v $(pwd)/data:/app/data \
  --restart unless-stopped \
  xrbzy/opsdeck:latest
```

如需使用 `docker compose`：

```yaml
services:
  opsdeck:
    image: xrbzy/opsdeck:latest
    container_name: opsdeck
    init: true
    ports:
      - "13113:13113"
    volumes:
      - ./data:/app/data
    environment:
      - ADMIN_SESSION_COOKIE_SECURE=false
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "7"
    restart: unless-stopped
```

启动：

```bash
docker compose up -d
```

Docker 部署注意事项：

- 必须保留 `./data:/app/data`，业务数据、配置、日志和在线更新状态都在这里
- 不需要挂载 `/app/runtime`
- 普通业务版本可在容器内更新二进制，容器重建后会从
  `/app/data/.opsdeck-update` 恢复已激活版本
- Debian、OpenCV 等运行环境 ABI 变化时，更新窗口会要求拉取完整镜像并重建容器
- 不要把容器根文件系统设置为完全只读，否则无法原子替换 `/app/opsdeck`
- `.opsdeck-update` 是系统保护目录，文件管理器和 `/api/data` 均不能浏览或修改

### 方式二：二进制部署

直接运行二进制前，需要安装 OpenCV 运行时依赖。

**Debian 12（amd64 / arm64）**

```bash
sudo apt-get update
sudo apt-get install -y \
  libopencv-core406 \
  libopencv-imgproc406 \
  libopencv-imgcodecs406 \
  libopencv-dnn406 \
  libopencv-objdetect406 \
  libopencv-features2d406 \
  libopencv-flann406 \
  libopencv-calib3d406
```

**Ubuntu 24.04（amd64 / arm64）**

```bash
sudo apt-get update
sudo apt-get install -y \
  libopencv-core406t64 \
  libopencv-imgproc406t64 \
  libopencv-imgcodecs406t64 \
  libopencv-dnn406t64 \
  libopencv-objdetect406t64 \
  libopencv-features2d406t64 \
  libopencv-flann406t64 \
  libopencv-calib3d406t64
```

**macOS ARM64**

```bash
brew install opencv
```

运行示例：

```bash
# Linux AMD64
tar -xzf opsdeck-linux-amd64-v*.tar.gz
chmod +x opsdeck-linux-amd64-v*
./opsdeck-linux-amd64-v*

# macOS ARM64
tar -xzf opsdeck-macos-arm64-v*.tar.gz
chmod +x opsdeck-macos-arm64-v*
./opsdeck-macos-arm64-v*
```

程序默认以 Supervisor 模式启动，以便执行在线更新、健康检查和失败回滚。
如需明确绕过 Supervisor 直接运行服务，可使用 `opsdeck serve`；这种模式不支持应用内热更新。

默认访问地址：

- `http://localhost:13113`

## 首次初始化与登录

首次启动时，登录页会显示初始化向导。

### 创建账户

1. 输入管理员用户名和密码。
2. 用户名必须为 5–12 位字母、数字、下划线，并以字母或数字开头。
3. 密码至少 8 位。
4. 生成 Authenticator 二维码，用认证器扫描后输入 6 位动态验证码。
5. 后端自动申请基础许可证并创建本地管理员。

创建完成后：

- 管理员配置写入数据库
- 许可证密钥使用本地 2FA 密钥加密后写入数据库
- 后台会话通过 HttpOnly Cookie 保存，默认有效期为 7 天

### 恢复已有账户

如果当前设备还没有本地管理员，但已有 OpsDeck 授权账户，可在初始化页切换到“恢复已有账户”，使用已有用户名、密码和 2FA 完成本地授权恢复。

### 登录与 2FA 管理

- 登录时会根据当前账户状态要求输入 2FA 动态验证码
- “系统设置”可修改密码、启用或停用登录 2FA
- “系统设置”也支持重新生成并确认新的 2FA 密钥
- 修改密码或 2FA 等敏感配置时，需要再次验证当前凭据

## 主要功能

### 仪表盘

仪表盘集中展示：

- CPU、内存、磁盘与主机信息
- 论坛账号数量、签到和回帖统计
- 各论坛近一周、近一月趋势
- 账号与插件的定时计划
- 已启用插件及最近插件日志
- 手动刷新和可保存的自动刷新间隔

### 账号管理

当前支持『色花(98)堂』和『2048核基地』两个论坛。

每个论坛账号页支持：

- 添加账号并完成登录
- 查看账号资料、状态、最近登录、签到和回帖时间
- 卡片与列表两种展示方式
- 按账号分组筛选和排序
- 为账号选择已保存代理
- 手动签到和手动回帖
- 编辑登录、签到、回帖定时计划
- 删除账号

账号数量受许可证 `max_instances` 限制。限制按两个论坛合并后的全局账号顺序计算（账号
ID 从小到大），不是每个论坛分别计算。超出上限的账号仍会显示在列表中并标记为“已锁定”，
但不能登录、签到、回帖、读取或保存任务/回帖配置，也不能通过定时任务执行；即使账号之前
已经保存过定时计划，后端在实际执行时仍会再次拦截。删除前面的账号或同步更高额度后，
重新加载账号列表即可看到新的可用范围。

### 定时任务

每个账号可独立配置：

- 登录定时
- 签到定时
- 回帖定时
- 固定时间或随机时间范围
- 回帖执行间隔和每日上限

定时任务与前端、Telegram 手动操作使用同一套后端执行器、会话恢复、授权检查、代理解析、日志和通知流程。
账号超出许可证上限时，前端会禁用相关操作，后端执行器和定时任务也会拒绝执行。

当前生产调度链路是 `DailyTaskGenerator` + `TaskSchedulerService`：前者生成当天的具体时间点，后者加载时间表并在到点后调用论坛执行器。`src/services/smart_scheduler.rs` 是保留的旧实现，不是当前启动入口。

### 回帖设置与 AI

回帖设置采用“论坛共享层 + 账号差异层”两层存储，不是为每个账号完整复制一份公共配置。

论坛共享层保存：

- 配置回帖内容池，执行时随机选择内容
- UID 黑名单和关键词黑名单
- 论坛 URL
- 共享 AI 配置

账号差异层保存：

- 选择回帖版块
- 分页范围
- HJD 的“回帖仅楼主可见”
- 只有与共享 AI 配置不一致时才保存 AI override；一致时自动回退共享配置

登录后的资料同步可以为单个账号写入论坛 URL 覆盖值；没有覆盖时使用论坛共享 URL。

读取有效配置时，优先级为“账号覆盖 → 论坛共享配置 → 默认配置”。回帖内容、黑名单和导入导出操作对应论坛共享层，版块配置导出对应账号差异层。

论坛差异：

- 『色花(98)堂』支持自定义分页表达式，例如 `3-5,8,9-15`
- 『2048核基地』分页固定为 1–2 页，并支持“回帖仅楼主可见”

AI 智能回复支持：

- OpenAI 兼容聊天接口，用于生成回帖内容
- Gemini 原生 `generateContent` HTTP API，用于生成回帖内容
- OpenAI Vision，用于验证码等图片识别，不会单独启用 AI 文本回帖
- OpenAI、Gemini 与 OpenAI Vision 可分别启用全局系统代理

Gemini API Key 使用 Google AI Studio 生成的密钥。它默认使用 `https://generativelanguage.googleapis.com/v1beta`，默认模型为 `gemini-2.5-flash`；使用官方接口时 Base URL 可留空。如果同时启用 OpenAI 和 Gemini，回帖执行器优先使用 OpenAI，其次使用 Gemini。

启用前可分别使用“测试”按钮验证 OpenAI、Gemini 和 OpenAI Vision 的 API Key、模型与 Base URL 连通性。

启用 AI 配置前会检查当前许可证是否包含 `ai` 功能。

### 代理管理

代理页面分为两类能力：

1. 账号代理：保存 HTTP、HTTPS、SOCKS 等代理，供论坛账号使用。
2. 基础设施代理：配置 CloudflareBypassForScraping 和全局系统代理。

账号代理支持：

- 添加、修改、删除和列表查看
- TCP 与 HTTP 连通性测试
- 导入、导出
- 分配给一个或多个论坛账号
- 查看关联账号与最近测试状态

全局系统代理可供授权验证、在线更新、AI、通知和支持该能力的插件使用。CloudflareBypassForScraping 用于处理 Cloudflare 挑战。

### 插件中心

默认可见插件：

- Google Drive
- 色花磁力爬虫
- Emby 助手

默认隐藏插件：

- 色花小黑屋
- JavDB 爬虫

隐藏插件只用于开发或测试；需要时设置 `SHOW_ALL_PLUGINS=true` 后重启。生产环境建议不要设置该变量。

Google Drive 插件当前支持 OAuth 多账号授权；网页登录成功后会自动列出“我的云端硬盘”和当前账号可访问的全部共享云端硬盘，可以直接切换浏览，无需手动填写硬盘 ID。插件同时支持目录分页、上传下载、新建目录、重命名、全盘搜索与手动停止搜索。所有文件的单项操作菜单均可按需“显示路径”，完整路径以当前网盘的实际名称为根节点；搜索结果还可以在勾选单个或多个项目后批量显示完整路径，路径补全后可以筛选或导出。文件菜单还支持详情、星标、移动、复制普通文件、移入回收站、恢复和永久删除。共享权限弹窗支持用户、群组、域和链接共享的添加/删除、通知邮件、到期时间及共享链接复制；工具栏可以查看账号存储空间和回收站占用。非空文件使用 Google Drive 可恢复分片上传，网络中断后会校准已确认偏移并继续传输，也可以取消当前上传。当前不支持直接复制整个文件夹。

Emby 助手的媒体库浏览、搜索、收藏、播放、扫描、元数据刷新、用户管理、服务器管理和计划任务读取/执行已经接入；“查看剧集”和“管理用户设备”目前只显示开发中提示，“通知设置”的保存按钮目前只提示成功，不会写回 Emby 服务器。请不要把这三个入口当作已完成能力。

### 文件管理

文件管理器支持：

- 浏览允许访问的根目录和子目录
- 文件名搜索，可选择包含子目录
- 上传文件和新建文件夹
- 下载文件
- 图片、文本和其它支持格式的预览
- 文本文件编辑与保存
- 调用图片编辑能力
- 重命名、删除和批量删除
- 按名称、类型、大小和修改时间排序

更新目录 `.opsdeck-update` 被后端强制保护，不会通过文件管理接口暴露。

### 下载设置

当前支持配置：

- CloudDrive2
- qBittorrent
- Aria2
- 115 网盘

不同插件可复用这些全局下载器配置执行推送或下载任务。

CloudDrive2 使用官方 gRPC API 推送离线下载任务，需要配置 gRPC 地址、API 令牌和目标文件夹。API 令牌必须具有“添加离线下载”（`AddOfflineFiles`）权限。

### 通知管理

当前支持：

- Telegram 通知
- Telegram Bot、Webhook、测试菜单和菜单头图
- 企业微信群机器人
- 钉钉机器人

各通知渠道可分别配置登录、签到、回帖、错误和插件通知，并提供测试功能。Telegram、企业微信和钉钉均可按配置选择是否使用全局系统代理。

### Telegram 交互机器人

Telegram 通知和交互机器人共用 Telegram 配置中的 Token，但由独立的 `bot.enabled` 开关控制。交互机器人可以通过菜单或命令管理：

- 论坛账号、手动登录、签到和回帖
- 账号的登录、签到、回帖定时任务
- 共享回帖内容、黑名单、OpenAI / Gemini / OpenAI Vision 配置，以及账号版块与论坛网址
- 已保存账号代理（查看、添加和修改）
- 全局系统代理与 CloudflareBypassForScraping 设置
- 插件、下载器、通知、系统设置、授权状态和日志

机器人模式：

- `polling`：后端保存配置后立即启动 long polling，不需要公网回调地址。
- `webhook`：填写公网 HTTPS 的 `bot.webhook_url`，建议同时设置 `bot.webhook_secret`，保存后点击“注册 Webhook”。Telegram 回调地址是 `POST /api/telegram/bot/webhook`。

凭据可以复用通知配置（`bot.reuse_notification_credentials=true`），也可以为机器人填写独立的 `bot.token` 和 `bot.chat_id`。Chat ID 支持逗号分隔的白名单；环境变量 `TELEGRAM_BOT_ALLOWED_CHAT_IDS`、`TELEGRAM_BOT_TOKEN`、`TELEGRAM_BOT_WEBHOOK_URL` 和 `TELEGRAM_BOT_WEBHOOK_SECRET` 会覆盖数据库中的对应值。

安全边界：

- 只有白名单 Chat ID 能执行菜单和命令；未列入白名单的聊天会被拒绝。可以先发送 `/chatid` 获取当前标识，再把它加入白名单。
- 已保存账号代理的读取、创建、修改和账号分配必须具备 `proxies` 授权；未授权时账号创建向导只能跳过代理。全局系统代理和 CloudflareBypassForScraping 不受该授权限制。
- 保存启用 OpenAI、Gemini 或 OpenAI Vision 前会检查 `ai` 授权；Telegram 菜单可直接查看和修改三类提供商配置。
- 超出许可证账号上限的账号不能通过机器人执行手动操作、修改任务或回帖配置；定时任务和底层执行器仍会再次检查账号锁定状态。

### 系统设置

系统设置包括：

- 修改管理员密码
- 管理登录 2FA
- 重置 2FA 密钥

### 日志管理

日志页面按论坛展示：

- 登录日志
- 签到日志
- 回帖日志
- 组合日志与搜索

插件日志页面展示插件执行记录，并可查看色花磁力爬虫等插件的运行详情。

### 主题与文档入口

顶部工具栏支持：

- 亮色 / 暗色主题切换
- 多套主题色
- 打开内置使用文档
- 打开 Telegram 频道

侧边栏底部 GitHub 图标和版本号用于打开应用更新窗口。

## 当前已知限制

以下接口或按钮在当前版本中仍是兼容脚手架或未完成实现：

- 通用任务路由 `GET /api/tasks`（列表）和 `POST /api/tasks`（创建）仍分别返回空列表或“功能开发中”。实际账号任务配置使用 `GET /api/tasks/{uid}?forum_code=...` 和 `PUT /api/tasks/{uid}/schedules?forum_code=...`。
- 通用插件路由 `POST /api/plugins/{id}/execute` 只返回模拟成功，`GET /api/plugins/{id}/stream` 只发送初始日志；通用目录浏览处理器也不是实际插件入口。真实动作必须使用插件自己的 handler 和专用路由，例如 Google Drive、色花磁力爬虫、JavDB 等插件路由。
- Emby 助手的“查看剧集”“管理用户设备”“通知设置保存”仍处于未完成状态，详见上面的插件中心说明。

这些限制不影响当前已接入的论坛账号、签到、回帖、定时任务、授权、代理、Telegram 和已实现插件专用路由，但扩展开发时不要把兼容接口误认为通用执行框架。

## 代理与授权边界

账号代理受许可证 `proxies` 功能控制：

- 读取已保存代理列表需要授权
- 添加、修改、删除、测试已保存代理需要授权
- 把已保存代理分配给账号需要授权
- 手动登录、签到、回帖解析账号代理时会再次检查授权
- 定时登录、签到、回帖使用相同检查
- 未授权时会跳过账号代理并记录告警，不会绕过授权读取已保存代理；论坛任务仍会继续进入后续流程并以无账号代理方式执行，后续网络或业务步骤仍可能因自身原因失败

临时代理测试用于填写代理时验证连通性，不读取数据库中的已保存代理，因此允许在保存前执行。

以下属于基础设施能力，不受账号代理 `proxies` 授权限制：

- 全局系统代理
- CloudflareBypassForScraping

这两项需要保证授权验证、插件、AI、通知和在线更新等基础链路在没有账号代理授权时仍可正常工作。

## 在线更新

点击侧边栏底部 GitHub 图标或版本号可打开更新窗口。

二进制部署和 Docker 部署共用同一套更新流程：

1. 通过 GitHub Release API 获取最新正式版本。
2. 精确匹配当前平台二进制资产。
3. 下载后计算 SHA256，并与 GitHub `assets[].digest` 比对。
4. 备份关键数据库、配置和 `.env`。
5. 停止定时任务并由 Supervisor 原子替换二进制。
6. 启动新版本并执行健康检查。
7. 健康检查失败时自动回滚到上一版本。

自动检查结果默认缓存 10 分钟；更新窗口中的“重新检查”会强制刷新。

更新窗口会同时显示两个不同标识：

- `macos-arm64`、`linux-amd64` 等是平台标识，用于选择对应的 Release 二进制资产
- `native-macos-arm64`、`debian12-opencv406-objdetect` 等是运行环境 ABI 标识，用于判断当前运行环境能否安全热更新

因此 macOS 二进制部署同时看到 `macos-arm64` 和 `native-macos-arm64` 属于正常情况；前者描述操作系统与 CPU，后者描述原生运行环境兼容性。

Docker 部署额外遵循：

- 普通业务版本使用二进制更新
- 运行环境 ABI 不兼容时显示“必须更新镜像”
- 此时执行 `docker pull xrbzy/opsdeck:latest` 并重建容器
- Docker Hub 业务镜像只保留 `latest`，版本历史保留在 GitHub Release

## 环境变量

常用环境变量：

```env
DATABASE_URL=sqlite:./data/database.db?mode=rwc
HOST=0.0.0.0
PORT=13113

LICENSE_SERVER=https://auth.xrbzy.xyz
ADMIN_SESSION_COOKIE_SECURE=auto

RUST_LOG=opsdeck=info,sqlx=warn,tower_http=warn
```

说明：

- `LICENSE_SERVER`：授权验证服务器地址
- `ADMIN_SESSION_COOKIE_SECURE`：支持 `auto`、`true`、`false`
- `auto` 会读取 `X-Forwarded-Proto` 或 `Forwarded` 判断当前请求是否为 HTTPS
- Docker 运行时镜像默认设置为 `false`，便于直接通过 HTTP 访问
- HTTPS 反向代理应正确转发协议头；只有明确需要强制安全 Cookie 时才设置为 `true`
- `SHOW_ALL_PLUGINS=true`：加载默认隐藏插件，仅用于开发或测试
- `OPSDECK_UPDATE_HEALTH_TIMEOUT_SECONDS=90`：更新后健康检查超时，最小 10 秒，默认 90 秒
- `OPSDECK_UPDATE_RELEASE_API_URL`：覆盖最新 Release API，仅建议自建发布仓库或测试环境使用
- `OPSDECK_UPDATE_CACHE_SECONDS=600`：更新检查缓存秒数，`0` 表示不缓存，最大 86400 秒
- `OPSDECK_UPDATE_ALLOW_HTTP=true`：仅允许本地测试使用 HTTP Release/资产地址，生产环境不要开启
- `OPSDECK_DATA_DIR=./data`：更新状态、更新备份与 `/api/data` 文件服务使用的数据根目录；除非所有持久化路径都已同步规划，否则不要单独修改
- 应用日志固定按日写入 `data/logs/app.log.YYYY-MM-DD`；Docker 可通过 `docker logs` 查看，前端系统日志同时提供实时流和当天历史
- `OPSDECK_GITHUB_TOKEN`：可选，用于提高 GitHub API 限额

许可证凭据只能通过初始化或恢复流程取得，不能通过环境变量注入。

## 内置使用文档

启动后可访问：

- `http://localhost:13113/api/docs`：Markdown 使用指南
- `http://localhost:13113/api/docs.html`：HTML 使用指南

这两个地址是使用文档，不是 OpenAPI 接口规范。

文档会在编译期嵌入二进制，因此 Docker 和二进制部署不需要额外挂载仓库的 `docs/` 目录。

## 常见问题

### 1. Docker 登录成功后又跳回登录页

当前默认模式是 `ADMIN_SESSION_COOKIE_SECURE=auto`，直接 HTTP 通常不需要额外配置。

如果使用 HTTPS 反向代理，请确认代理转发：

```text
X-Forwarded-Proto: https
```

如果部署环境明确要求固定行为，再设置：

```env
ADMIN_SESSION_COOKIE_SECURE=true
```

Docker 镜像默认已设置为 `false`。

### 2. 二进制启动失败，提示找不到 OpenCV 相关库

说明宿主机缺少 OpenCV 运行时依赖。请按当前系统安装上文列出的完整依赖，或改用 Docker 部署。

### 3. 授权校验失败

优先检查：

- `LICENSE_SERVER` 是否指向正确的验证服务器
- 主项目内置公钥是否与验证服务器当前签名公钥一致
- 系统时间是否正确
- 验证服务器是否可访问
- 如依赖全局系统代理，代理是否测试成功

### 4. 代理页面提示未授权

已保存账号代理属于 `proxies` 授权功能。临时测试可以执行，但列表、保存、修改、删除、已保存代理测试和账号分配都会被后端校验。

全局系统代理与 CloudflareBypassForScraping 是基础设施配置，不受账号代理授权限制。

### 5. 从其它页面切回来后数据没有立刻刷新

部分非实时页面使用缓存减少重复请求。可点击页面刷新按钮立即重新拉取；仪表盘等实时资源会按自身刷新间隔更新。

### 6. Telegram 机器人没有响应或提示未授权

确认机器人已启用、Token 和允许访问的 Chat ID 已配置，并检查当前聊天 ID 是否位于白名单。Polling 模式应确认后端日志中已启动 long polling；Webhook 模式应确认公网 HTTPS 地址、Webhook Secret 和“注册 Webhook”操作均成功。

### 7. Docker 在线更新后重建容器又回到旧版本

确认仍挂载原来的 `./data:/app/data`，并检查 `data/.opsdeck-update/active.json` 是否存在。不要删除该保护目录，也不要更换为新的空数据卷。
