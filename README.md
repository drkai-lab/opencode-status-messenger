# OpenCode Telegram Notification Plugin

Get notified on Telegram when your OpenCode sessions complete.

**Forked from** [Davasny/opencode-telegram-notification-plugin](https://github.com/Davasny/opencode-telegram-notification-plugin) | **Maintained by AppWizz by Kai**

## Architecture

This project consists of two components:

| Component | Location | Description |
|-----------|----------|-------------|
| **Plugin** (Client) | Your local machine or VPS where OpenCode runs | Lightweight JS file that monitors session events and sends notifications to the Worker |
| **Worker** (Server) | Cloudflare Workers (recommended, free tier) or your own VPS | Telegram bot that receives notifications from the Plugin and delivers messages via Telegram API |

### Where Should You Run This?

- **Plugin**: Must run on the same machine as OpenCode (local development environment or VPS)
- **Worker**: Best deployed on **Cloudflare Workers** (serverless, free for up to 100k requests/day). Alternatively, you can host it on a VPS with Node.js.

---

## Features

- ✅ Real-time notifications via Telegram
- ✅ Session completion tracking with duration metrics
- ✅ Secure key-based authentication (chat ID never stored in plugin)
- ✅ Revocable installation keys
- ✅ Compatible with OpenCode 1.14+ API
- ✅ Lightweight — single JS file, no dependencies

---

## Quick Start (Using Public Bot)

The easiest way to get started uses the pre-deployed bot:

1. Start a chat with [@opencodetelegramnotificationbot](https://t.me/opencodetelegramnotificationbot)
2. Send `/start`
3. The bot sends you an installation command like:
   ```bash
   curl -fsSL https://raw.githubusercontent.com/drkai-lab/opencode-status-messenger/main/scripts/install.sh | bash -s -- <YOUR_INSTALL_KEY>
   ```
4. Run that command on your terminal
5. Done! Your OpenCode sessions will now send Telegram notifications

---

## Plugin Installation (Manual)

If you prefer manual installation or are using a custom Worker:

### Step 1: Get Your Install Key

1. Chat with [@opencodetelegramnotificationbot](https://t.me/opencodetelegramnotificationbot)
2. Send `/start` to receive your unique install key

### Step 2: Download the Plugin File

```bash
# Create plugin directory
mkdir -p ~/.config/opencode/plugin

# Download the compiled plugin
curl -fsSL "https://raw.githubusercontent.com/drkai-lab/opencode-status-messenger/main/plugin/dist/telegram-notify.js" \
  -o /tmp/telegram-notify.js
```

### Step 3: Configure the Plugin

Edit `/tmp/telegram-notify.js` and replace these placeholders:

```javascript
var INSTALL_KEY = "__INSTALL_KEY__";      // Replace with your key from bot
var WORKER_URL = "__WORKER_URL__";        // Replace with your Worker URL
```

Example:
```javascript
var INSTALL_KEY = "a1b2c3d4-e5f6-7890-abcd-ef1234567890";
var WORKER_URL = "https://opencode-telegram-bot.your-subdomain.workers.dev";
```

### Step 4: Install to Plugin Directory

```bash
# Move configured file to plugin directory
mv /tmp/telegram-notify.js ~/.config/opencode/plugin/telegram-notify.js

# Verify installation
ls -la ~/.config/opencode/plugin/telegram-notify.js
```

### Step 5: Restart OpenCode

Restart your OpenCode instance for the plugin to load. You'll receive a confirmation message on Telegram.

---

## Commands (Telegram Bot)

| Command | Description |
|---------|-------------|
| `/start` | Get installation command with your unique key |
| `/revoke` | Generate new key (invalidates old one) |
| `/status` | Check if your plugin is installed and active |
| `/help` | Show this help message |

---

## How It Works

1. The bot generates a unique installation key for you
2. The plugin file is placed in `~/.config/opencode/plugin/`
3. When OpenCode finishes a task, the plugin sends a notification to the Worker URL
4. The Worker processes the request and delivers a Telegram message via the API

Your Telegram chat ID is never stored in the plugin — only a revocable key.

---

## Uninstall

```bash
rm ~/.config/opencode/plugin/telegram-notify.js
```

Optionally, send `/revoke` to the bot to invalidate your key.

---

## Security

- Your chat ID never leaves the server
- You can revoke your key anytime with `/revoke`
- The plugin only contains a UUID key, not your chat ID
- All communication uses HTTPS

---

## Deploy Your Own Worker (VPS / Cloudflare)

For full control and customization, deploy your own instance.

### Option A: Cloudflare Workers (Recommended — Free)

**Prerequisites:**
- Cloudflare account
- Telegram Bot Token from [@BotFather](https://t.me/BotFather)

```bash
# Clone repository
git clone https://github.com/drkai-lab/opencode-status-messenger.git
cd opencode-status-messenger

# Install dependencies
pnpm install

# Login to Cloudflare (first time only)
cd worker && pnpm exec wrangler login

# Create KV namespace for user data
pnpm exec wrangler kv namespace create "USERS"
# Copy the returned ID and update worker/wrangler.jsonc with it

# Deploy the worker
pnpm exec wrangler deploy

# Set BOT_TOKEN secret
pnpm exec wrangler secret put BOT_TOKEN

# Set Telegram webhook
curl "https://api.telegram.org/bot<BOT_TOKEN>/setWebhook?url=https://<your-worker-url>/webhook"
```

### Option B: VPS Deployment (Full Control)

**Prerequisites:**
- Linux VPS with Node.js 22+ and pnpm 10+
- Telegram Bot Token from [@BotFather](https://t.me/BotFather)

```bash
# Prepare system
sudo apt update && sudo apt upgrade -y
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
npm install -g pnpm@latest

# Clone and setup
git clone https://github.com/drkai-lab/opencode-status-messenger.git
cd opencode-status-messenger
pnpm install

# Build plugin (for distribution)
cd plugin && pnpm build

# Configure worker environment
cd ../worker
cp .env.example .env.local
# Edit .env.local with your BOT_TOKEN and other settings

# Start with PM2 for process management
npm install -g pm2
pm2 start src/index.ts --watch
pm2 save
```

### Troubleshooting

| Issue | Solution |
|-------|----------|
| `wrangler` not found | Run `pnpm add -g wrangler` |
| KV namespace error | Verify `worker/wrangler.jsonc` has correct `kv_namespaces` config |
| Webhook timeout | Ensure your worker URL is publicly accessible |
| Bot not responding | Check `/status` command and verify key is valid |
| Plugin not loading | Confirm file exists at `~/.config/opencode/plugin/telegram-notify.js` |

---

## Development

### Prerequisites

- Node.js 22+
- pnpm 10+
- Cloudflare account (for Worker development)
- Telegram Bot Token (from [@BotFather](https://t.me/BotFather))

### Setup

```bash
# Install dependencies
pnpm install

# Create KV namespace
wrangler kv namespace create "USERS"

# Add KV ID to worker/wrangler.jsonc

# Set bot token
wrangler secret put BOT_TOKEN

# Start local development
pnpm dev
```

### Redeploy

To redeploy after changes:
```bash
cd worker && pnpm exec wrangler deploy
```

---

## Project Structure

```
├── plugin/           # OpenCode plugin (client-side)
│   ├── src/          # TypeScript source
│   └── dist/         # Compiled JS output (telegram-notify.js)
├── worker/           # Cloudflare Worker / VPS server (bot backend)
│   ├── src/          # Bot logic with Hono + Grammy
│   └── test/         # Vitest tests
├── scripts/          # Installation helpers (install.sh)
└── docs/             # Documentation and strategy documents
```

---

## License

See [LICENSE.md](./LICENSE.md) — MIT License, forked from original with attribution.
