# 安装与部署

OpsDeck 是 Rust + Vue 3 多账户操作控制台，提供论坛登录、签到、回帖、定时任务、插件、日志和通知。默认端口为 `13113`。公开下载仓库为 [xiarenyyds/OpsDeck-Rust](https://github.com/xiarenyyds/OpsDeck-Rust)，Docker 镜像为 `xrbzy/opsdeck:latest`。

## Docker 部署

推荐使用 Docker，运行镜像已提供 OpenCV 依赖。在一个固定目录保存以下 `compose.yaml`：

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
      ADMIN_SESSION_COOKIE_SECURE: "auto"
    restart: unless-stopped
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "7"
```

```bash
docker compose up -d
docker compose logs --tail 100 -f
```

浏览器打开 `http://localhost:13113`；远程主机将 `localhost` 替换为主机地址。
反向代理使用 HTTPS 时传递 `X-Forwarded-Proto` 或标准 `Forwarded`，让 `auto` 正确设置会话 Cookie。

必须保留 `data` 挂载，其中包含数据库、配置、日志和 `.opsdeck-update` 在线更新状态。无需挂载 `/app/runtime`；容器内应用路径需可写，以支持二进制原子替换。

## 二进制部署

从 [Releases](https://github.com/xiarenyyds/OpsDeck-Rust/releases) 下载与系统匹配的归档。当前构建脚本支持以下平台；具体可用附件以该版本发布页为准。

| 平台 | 归档名 |
| --- | --- |
| Linux x64 | `opsdeck-linux-amd64-vX.Y.Z.tar.gz` |
| Linux ARM64 | `opsdeck-linux-arm64-vX.Y.Z.tar.gz` |
| macOS Apple 芯片 | `opsdeck-macos-arm64-vX.Y.Z.tar.gz` |

Debian 12 安装运行库：

```bash
sudo apt-get update
sudo apt-get install -y libopencv-core406 libopencv-imgproc406 \
  libopencv-imgcodecs406 libopencv-dnn406 libopencv-objdetect406 \
  libopencv-features2d406 libopencv-flann406 libopencv-calib3d406
```

Ubuntu 24.04 使用对应的 `406t64` 包名；macOS 使用 `brew install opencv`。
解压后在固定工作目录执行对应二进制，避免切换目录导致程序创建另一份 `data`：

```bash
tar -xzf opsdeck-linux-amd64-vX.Y.Z.tar.gz
chmod +x opsdeck-linux-amd64-vX.Y.Z
./opsdeck-linux-amd64-vX.Y.Z
```

将示例版本替换为实际附件版本。普通启动会进入 Supervisor 模式；`serve` 子命令绕过 Supervisor，不支持应用内热更新。

## 首次初始化

1. 打开网页，选择创建新账户；已有授权账户按页面提示选择恢复。
2. 用户名使用 5–12 位字母、数字或下划线，以字母或数字开头；密码至少 8 位。
3. 按向导绑定 Authenticator，输入当前动态验证码并提交。
4. 后端自动申请基础许可证，再进入控制台。账号数量、AI 和代理等能力以实际授权为准。

保存好 Authenticator。二维码、恢复材料、许可证、Cookie、API Key 与 Telegram Token 都属于凭据，不应出现在公开截图或反馈中。界面操作见 [界面图解](ui-guide.md)。

## 升级与备份

升级前停止写入并备份整个 `data`，再恢复服务；仅复制正在写入的 SQLite 主文件可能丢失 WAL 中的数据。

- 普通版本：在界面的更新窗口检查并应用，客户端验证 GitHub 资产 SHA256 digest，Supervisor 负责健康检查和失败回滚。
- 运行环境 ABI 变化：按更新提示执行以下命令更新整个镜像。

```bash
docker compose pull
docker compose up -d
```

热更新备份会在健康检查成功后清理，不能替代自己的长期备份。容器重建会从持久化目录恢复已激活版本。

## 常见问题

| 现象 | 排查方向 |
| --- | --- |
| 无法打开网页 | 检查容器日志、端口映射、防火墙和实际监听地址 |
| 刷新后返回登录页 | 检查 Cookie、代理 HTTPS 头和服务器时间 |
| 二进制提示找不到 OpenCV | 安装对应系统运行库，或改用 Docker |
| 更新检查提示无版本 | 确认目标仓库已有正式 Release；草稿不出现在 latest API 中 |
| 无法启用 AI 或代理 | 检查许可证能力与配置，不把功能开关视为授权结果 |
| 重建后配置消失 | 检查 `data` 挂载是否仍指向原目录 |

运行后可从 `/api/docs` 或 `/api/docs.html` 阅读内置完整指南。这两个入口提供用户文档，不是 OpenAPI 规格。
