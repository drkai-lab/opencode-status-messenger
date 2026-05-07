# OpenCode Telegram Notification Plugin

Get notified on Telegram when your OpenCode sessions complete.

**Forked from** [Davasny/opencode-telegram-notification-plugin](https://github.com/Davasny/opencode-telegram-notification-plugin) | **Maintained by AppWizz by Kai**

## Features

- ✅ Real-time notifications via Telegram
- ✅ Session completion tracking with duration metrics
- ✅ Secure key-based authentication (chat ID never stored in plugin)
- ✅ Revocable installation keys
- ✅ Compatible with OpenCode 1.14+ API
- ✅ Lightweight — single JS file, no dependencies

## Quick Start (Bot Installation)

The easiest way to get started:

1. Start a chat with [@opencodetelegramnotificationbot](https://t.me/opencodetelegramnotificationbot)
2. Send `/start`
3. Run the command the bot sends you
4. Done! Your OpenCode sessions will now send Telegram notifications

## Commands

| Command | Description |
|---------|-------------|
| `/start` | Get installation command |
| `/revoke` | Generate new key (invalidates old one) |
| `/status` | Check installation status |
| `/help` | Show help message |

## How It Works

1. The bot generates a unique installation key for you
2. The installation script configures the plugin with your key
3. When OpenCode finishes a task, the plugin notifies the bot
4. The bot sends you a Telegram message

Your Telegram chat ID is never stored in the plugin — only a revocable key.

## Uninstall

```bash
rm ~/.config/opencode/plugin/telegram-notify.js
```

Optionally, send `/revoke` to the bot to invalidate your key.

## Security

- Your chat ID never leaves the server
- You can revoke your key anytime with `/revoke`
- The plugin only contains a UUID key, not your chat ID

---

## VPS Deployment Guide (Beginner Friendly)

Deploy your own instance on a Virtual Private Server (VPS) for full control. This guide assumes you have a basic Linux VPS (Ubuntu/Debian recommended).

### Prerequisites

- A VPS with **Node.js 22+** and **pnpm 10+** installed
- A Cloudflare account (for Worker hosting)
- A Telegram Bot Token from [@BotFather](https://t.me/BotFather)
- Basic terminal familiarity

### Step 1: Prepare Your VPS

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Node.js 22 (if not installed)
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs

# Install pnpm
npm install -g pnpm@latest

# Clone the repository
git clone https://github.com/drkai-lab/opencode-status-messenger.git
cd opencode-status-messenger

# Install dependencies
pnpm install
```

### Step 2: Set Up Cloudflare Worker

The notification bot runs on **Cloudflare Workers** (serverless, free tier available).

```bash
# Login to Cloudflare (first time only)
cd worker
pnpm exec wrangler login

# Create KV namespace for user data
pnpm exec wrangler kv namespace create "USERS"
# Copy the returned ID and update worker/wrangler.jsonc with it

# Deploy the worker
pnpm exec wrangler deploy
```

After deployment, note your worker URL (e.g., `https://opencode-telegram-bot.<your-subdomain>.workers.dev`).

### Step 3: Configure Telegram Bot Token

```bash
# Set the BOT_TOKEN secret
pnpm exec wrangler secret put BOT_TOKEN
# Paste your bot token when prompted
```

### Step 4: Set Telegram Webhook

Connect your worker to Telegram:

```bash
curl "https://api.telegram.org/bot<BOT_TOKEN>/setWebhook?url=https://<your-worker-url>/webhook"
```

Replace `<BOT_TOKEN>` with your actual bot token and `<your-worker-url>` with your deployed worker URL.

### Step 5: Verify Installation

1. Open Telegram and find your bot
2. Send `/start`
3. The bot should respond with an installation command
4. Run that command on any machine where you use OpenCode

### Optional: Auto-Restart with PM2 (for local Node services)

If you run additional services locally, install PM2 for process management:

```bash
npm install -g pm2
pm2 start worker/src/index.ts --watch
pm2 save
```

### Troubleshooting

| Issue | Solution |
|-------|----------|
| `wrangler` not found | Run `pnpm add -g wrangler` |
| KV namespace error | Verify `worker/wrangler.jsonc` has correct `kv_namespaces` config |
| Webhook timeout | Ensure your worker URL is publicly accessible |
| Bot not responding | Check `/status` command and verify key is valid |

---

## Development

### Prerequisites

- Node.js 22+
- pnpm 10+
- Cloudflare account
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

### Manual Deployment (from localhost)

```bash
# 1. Login to Cloudflare (first time only)
cd worker && pnpm exec wrangler login

# 2. Create KV namespace (first time only)
pnpm exec wrangler kv namespace create "USERS"
# Copy the ID and update worker/wrangler.jsonc

# 3. Deploy the worker
pnpm exec wrangler deploy

# 4. Set the BOT_TOKEN secret (first time or when rotating token)
pnpm exec wrangler secret put BOT_TOKEN
# Paste your bot token when prompted

# 5. Set Telegram webhook (first time only)
curl "https://api.telegram.org/bot<BOT_TOKEN>/setWebhook?url=https://opencode-telegram-bot.<your-subdomain>.workers.dev/webhook"
```

To redeploy after changes, just run:
```bash
cd worker && pnpm exec wrangler deploy
```

### Set Webhook

After deployment, set the Telegram webhook:

```bash
curl "https://api.telegram.org/bot<BOT_TOKEN>/setWebhook?url=https://opencode-telegram-bot.<subdomain>.workers.dev/webhook"
```

## Project Structure

```
├── plugin/           # OpenCode plugin (client-side)
│   ├── src/          # TypeScript source
│   └── dist/         # Compiled JS output
├── worker/           # Cloudflare Worker (server-side bot)
│   ├── src/          # Bot logic with Hono + Grammy
│   └── test/         # Vitest tests
├── scripts/          # Installation helpers
└── docs/plans/       # Strategy documents (gitignored)
```

## License

See [LICENSE.md](./LICENSE.md) — MIT License, forked from original with attribution.
