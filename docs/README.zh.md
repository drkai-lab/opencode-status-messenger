# OpenCode Telegram Notification Plugin

当您的OpenCode会话完成时，通过Telegram接收通知。

**Forked from** [Davasny/opencode-telegram-notification-plugin](https://github.com/Davasny/opencode-telegram-notification-plugin) | **Maintained by AppWizz by Kai**

## 架构

本项目由两个组件组成：

| 组件 | 位置 | 说明 |
|------|------|------|
| **Plugin** (客户端) | OpenCode运行的本地机器或VPS | 监控会话事件并向Worker发送通知的轻量级JS文件 |
| **Worker** (服务端) | Cloudflare Workers（推荐，免费额度）或您的VPS | 接收Plugin通知并通过Telegram API投递消息的机器人 |

### 运行位置选择

- **Plugin**: 必须与OpenCode在同一台机器上运行（本地开发环境或VPS）
- **Worker**: 最佳部署在 **Cloudflare Workers**（无服务器，每日10万请求免费）。或者您可以在带有Node.js的VPS上托管。

---

## 功能特性

- ✅ 通过Telegram实时通知
- ✅ 会话完成追踪与时长指标
- ✅ 安全的基于密钥的身份验证（聊天ID永不存储在插件中）
- ✅ 可撤销的安装密钥
- ✅ 兼容OpenCode 1.14+ API
- ✅ 轻量级 — 单个JS文件，无依赖

---

## 快速开始（使用公开机器人）

最简单的方式是使用预部署的机器人：

1. 与 [@opencodestatusmessenger](https://t.me/opencodestatusmessenger) 开始聊天
2. 发送 `/start`
3. 机器人会向您发送安装命令，例如：
   ```bash
   curl -fsSL https://raw.githubusercontent.com/drkai-lab/opencode-status-messenger/main/scripts/install.sh | bash -s -- <YOUR_INSTALL_KEY>
   ```
4. 在终端中运行该命令
5. 完成！您的OpenCode会话现在将通过Telegram发送通知

---

## 插件手动安装

如果您偏好手动安装或使用自定义Worker：

### 步骤1: 获取安装密钥

1. 与 [@opencodestatusmessenger](https://t.me/opencodestatusmessenger) 聊天
2. 发送 `/start` 以接收您的唯一安装密钥

### 步骤2: 下载插件文件

```bash
# 创建插件目录
mkdir -p ~/.config/opencode/plugin

# 下载编译后的插件
curl -fsSL "https://raw.githubusercontent.com/drkai-lab/opencode-status-messenger/main/plugin/dist/telegram-notify.js" \
  -o /tmp/telegram-notify.js
```

### 步骤3: 配置插件

编辑 `/tmp/telegram-notify.js` 并替换以下占位符：

```javascript
var INSTALL_KEY = "__INSTALL_KEY__";      // 替换为机器人提供的密钥
var WORKER_URL = "__WORKER_URL__";        // 替换为您的Worker URL
```

示例:
```javascript
var INSTALL_KEY = "a1b2c3d4-e5f6-7890-abcd-ef1234567890";
var WORKER_URL = "https://opencode-telegram-bot.your-subdomain.workers.dev";
```

### 步骤4: 安装到插件目录

```bash
# 将配置好的文件移动到插件目录
mv /tmp/telegram-notify.js ~/.config/opencode/plugin/telegram-notify.js

# 验证安装
ls -la ~/.config/opencode/plugin/telegram-notify.js
```

### 步骤5: 完成

重启OpenCode或开始下一个会话，插件将自动加载。

**通知时机:**  
通知不是在安装后立即发送，而是在 **OpenCode完成任务时** 通过Telegram发送。也就是说，当您执行某个任务且该会话结束时，您才会首次收到消息。

---

## 命令（Telegram机器人）

| 命令 | 说明 |
|------|------|
| `/start` | 获取带有唯一密钥的安装命令 |
| `/revoke` | 生成新密钥（使旧密钥失效） |
| `/status` | 检查插件是否已安装并处于活动状态 |
| `/help` | 显示帮助消息 |

---

## 工作原理

1. 机器人生成唯一的安装密钥
2. 插件文件放置在 `~/.config/opencode/plugin/`
3. 当OpenCode完成任务时，插件向Worker URL发送通知
4. Worker处理请求并通过Telegram API投递消息

您的Telegram聊天ID永远不会存储在插件中 — 只有可撤销的密钥。

---

## 卸载

```bash
rm ~/.config/opencode/plugin/telegram-notify.js
```

可选：向机器人发送 `/revoke` 以使您的密钥失效。

---

## 安全性

- 您的聊天ID永不离开服务器
- 您可以随时使用 `/revoke` 撤销密钥
- 插件只包含UUID密钥，不包含您的聊天ID
- 所有通信使用HTTPS

---

## 部署您自己的Worker（VPS / Cloudflare）

为了完全控制和自定义，部署您自己的实例。

### 选项A: Cloudflare Workers（推荐 — 免费）

**前提条件:**
- Cloudflare账户
- 从 [@BotFather](https://t.me/BotFather) 获取Telegram Bot Token

```bash
# 克隆仓库
git clone https://github.com/drkai-lab/opencode-status-messenger.git
cd opencode-status-messenger

# 安装依赖
pnpm install

# 登录Cloudflare（首次仅需）
cd worker && pnpm exec wrangler login

# 为用户数据创建KV命名空间
pnpm exec wrangler kv namespace create "USERS"
# 复制返回的ID并更新 worker/wrangler.jsonc

# 部署Worker
pnpm exec wrangler deploy

# 设置BOT_TOKEN密钥
pnpm exec wrangler secret put BOT_TOKEN

# 设置Telegram webhook
curl "https://api.telegram.org/bot<BOT_TOKEN>/setWebhook?url=https://<your-worker-url>/webhook"
```

### 选项B: VPS部署（完全控制）

**前提条件:**
- 安装了Node.js 22+和pnpm 10+的Linux VPS
- 从 [@BotFather](https://t.me/BotFather) 获取Telegram Bot Token

```bash
# 准备系统
sudo apt update && sudo apt upgrade -y
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
npm install -g pnpm@latest

# 克隆和设置
git clone https://github.com/drkai-lab/opencode-status-messenger.git
cd opencode-status-messenger
pnpm install

# 构建插件（用于分发）
cd plugin && pnpm build

# 配置Worker环境
cd ../worker
cp .env.example .env.local
# 编辑.env.local填入您的BOT_TOKEN等设置

# 使用PM2进行进程管理启动
npm install -g pm2
pm2 start src/index.ts --watch
pm2 save
```

### 故障排除

| 问题 | 解决方案 |
|------|----------|
| `wrangler` 未找到 | 运行 `pnpm add -g wrangler` |
| KV命名空间错误 | 验证 `worker/wrangler.jsonc` 有正确的 `kv_namespaces` 配置 |
| Webhook超时 | 确保您的Worker URL可公开访问 |
| 机器人无响应 | 检查 `/status` 命令并验证密钥有效 |
| 插件未加载 | 确认文件存在于 `~/.config/opencode/plugin/telegram-notify.js` |

---

## 开发

### 前提条件

- Node.js 22+
- pnpm 10+
- Cloudflare账户（用于Worker开发）
- Telegram Bot Token（从 [@BotFather](https://t.me/BotFather) 获取）

### 设置

```bash
# 安装依赖
pnpm install

# 创建KV命名空间
wrangler kv namespace create "USERS"

# 将KV ID添加到 worker/wrangler.jsonc

# 设置机器人令牌
wrangler secret put BOT_TOKEN

# 启动本地开发
pnpm dev
```

### 重新部署

更改后重新部署：
```bash
cd worker && pnpm exec wrangler deploy
```

---

## 项目结构

```
├── plugin/           # OpenCode插件（客户端）
│   ├── src/          # TypeScript源码
│   └── dist/         # 编译后的JS输出 (telegram-notify.js)
├── worker/           # Cloudflare Worker / VPS服务器（机器人后端）
│   ├── src/          # 使用Hono + Grammy的机器人逻辑
│   └── test/         # Vitest测试
├── scripts/          # 安装辅助脚本 (install.sh)
└── docs/             # 文档和策略文件
```

---

## 许可证

参见 [LICENSE.md](./LICENSE.md) — MIT License，从原作fork并带有归属声明。
