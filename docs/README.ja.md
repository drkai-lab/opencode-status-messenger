# OpenCode Telegram Notification Plugin

OpenCodeのセッション完了時にTelegramで通知を受け取るプラグインです。

**フォーク元:** [Davasny/opencode-telegram-notification-plugin](https://github.com/Davasny/opencode-telegram-notification-plugin) | **保守: AppWizz by Kai**

## アーキテクチャ

このプロジェクトは2つのコンポーネントで構成されています：

| コンポーネント | 配置場所 | 説明 |
|---------------|----------|------|
| **Plugin** (クライアント) | OpenCodeが動作するローカルマシンまたはVPS | セッションイベントを監視し、Workerに通知を送信する軽量JSファイル |
| **Worker** (サーバー) | Cloudflare Workers（推奨・無料枠あり）または自前のVPS | Pluginからの通知を受け取り、Telegram API経由でメッセージ配信するボット |

### 実行場所の選択

- **Plugin**: OpenCodeと同じマシンに配置する必要があります（ローカル開発環境またはVPS）
- **Worker**: **Cloudflare Workers** が推奨されます（サーバーレス・1日10万リクエストまで無料）。 alternativ にNode.js環境のあるVPSでも動作します。

---

## 機能

- ✅ Telegram経由のリアルタイム通知
- ✅ セッション完了時の所要時間計測
- ✅ キーベース認証（チャットIDはPluginに保存されない）
- ✅ 失効可能なインストールキー
- ✅ OpenCode 1.14+ API対応
- ✅ 軽量 — シングルJSファイル、依存関係なし

---

## クイックスタート（公開ボット利用）

最も簡単な方法は事前にデプロイされたボットを利用することです：

1. [@opencodetelegramnotificationbot](https://t.me/opencodetelegramnotificationbot) とチャットを開始
2. `/start` を送信
3. ボットからインストールコマンドが届きます（例）:
   ```bash
   curl -fsSL https://raw.githubusercontent.com/drkai-lab/opencode-status-messenger/main/scripts/install.sh | bash -s -- <YOUR_INSTALL_KEY>
   ```
4. ターミナルでそのコマンドを実行
5. 完了！OpenCodeセッションがTelegramに通知を送るようになります

---

## Pluginの手動インストール

カスタムWorkerを使用している場合や手動インストールを好む場合：

### ステップ1: インストールキーの取得

1. [@opencodetelegramnotificationbot](https://t.me/opencodetelegramnotificationbot) とチャット
2. `/start` を送信して固有のインストールキーを取得

### ステップ2: Pluginファイルのダウンロード

```bash
# プラグインディレクトリを作成
mkdir -p ~/.config/opencode/plugin

# コンパイル済みプラグインをダウンロード
curl -fsSL "https://raw.githubusercontent.com/drkai-lab/opencode-status-messenger/main/plugin/dist/telegram-notify.js" \
  -o /tmp/telegram-notify.js
```

### ステップ3: Pluginの設定

`/tmp/telegram-notify.js` を編集し、以下のプレースホルダーを置換します：

```javascript
var INSTALL_KEY = "__INSTALL_KEY__";      // ボットから取得したキーに置換
var WORKER_URL = "__WORKER_URL__";        // WorkerのURLに置換
```

例:
```javascript
var INSTALL_KEY = "a1b2c3d4-e5f6-7890-abcd-ef1234567890";
var WORKER_URL = "https://opencode-telegram-bot.your-subdomain.workers.dev";
```

### ステップ4: プラグインディレクトリに配置

```bash
# 設定済みファイルをプラグインディレクトリへ移動
mv /tmp/telegram-notify.js ~/.config/opencode/plugin/telegram-notify.js

# インストール確認
ls -la ~/.config/opencode/plugin/telegram-notify.js
```

### ステップ5: 完了

OpenCodeを再起動するか、または次のセッションを開始してください。プラグインは自動的に読み込まれます。

**通知のタイミング:**  
インストール直後ではなく、**OpenCodeがタスクを完了した際**にTelegramへ通知が届きます。つまり、何かタスクを実行してそのセッションが終了すると、初めてメッセージを受信します。

---

## コマンド（Telegramボット）

| コマンド | 説明 |
|----------|------|
| `/start` | 固有キー付きインストールコマンドを取得 |
| `/revoke` | 新しいキーを生成（旧キーを失効） |
| `/status` | プラグインのインストール状態を確認 |
| `/help` | ヘルプメッセージを表示 |

---

## 動作原理

1. ボットが固有のインストールキーを生成
2. プラグインファイルを `~/.config/opencode/plugin/` に配置
3. OpenCodeがタスク完了時に、プラグインがWorker URLへ通知を送信
4. Workerがリクエストを処理し、Telegram API経由でメッセージ配信

あなたのTelegramチャットIDはPluginに保存されません — 失効可能なキーのみです。

---

## アンインストール

```bash
rm ~/.config/opencode/plugin/telegram-notify.js
```

オプション: ボットに `/revoke` を送信してキーを失効できます。

---

## セキュリティ

- チャットIDはサーバー外に出ない
- `/revoke` でいつでもキーを失効可能
- PluginにはUUIDキーのみが含まれ、チャットIDは含まれない
- 全通信はHTTPSを使用

---

## 自作Workerのデプロイ（VPS / Cloudflare）

完全な制御とカスタマイズのために、独自のインスタンスをデプロイします。

### オプションA: Cloudflare Workers（推奨 — 無料）

**前提条件:**
- Cloudflareアカウント
- [@BotFather](https://t.me/BotFather) からTelegram Bot Tokenを取得

```bash
# リポジトリをクローン
git clone https://github.com/drkai-lab/opencode-status-messenger.git
cd opencode-status-messenger

# 依存関係をインストール
pnpm install

# Cloudflareにログイン（初回のみ）
cd worker && pnpm exec wrangler login

# ユーザーデータ用のKVネームスペースを作成
pnpm exec wrangler kv namespace create "USERS"
# 返されたIDをコピーし、worker/wrangler.jsoncを更新

# Workerをデプロイ
pnpm exec wrangler deploy

# BOT_TOKENシークレットを設定
pnpm exec wrangler secret put BOT_TOKEN

# Telegram webhookを設定
curl "https://api.telegram.org/bot<BOT_TOKEN>/setWebhook?url=https://<your-worker-url>/webhook"
```

### オプションB: VPSデプロイ（完全制御）

**前提条件:**
- Node.js 22+ と pnpm 10+ がインストールされたLinux VPS
- [@BotFather](https://t.me/BotFather) からTelegram Bot Tokenを取得

```bash
# システムを準備
sudo apt update && sudo apt upgrade -y
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
npm install -g pnpm@latest

# クローンとセットアップ
git clone https://github.com/drkai-lab/opencode-status-messenger.git
cd opencode-status-messenger
pnpm install

# Pluginをビルド（配布用）
cd plugin && pnpm build

# Worker環境を設定
cd ../worker
cp .env.example .env.local
# .env.localにBOT_TOKENなどを記入

# PM2でプロセス管理しながら起動
npm install -g pm2
pm2 start src/index.ts --watch
pm2 save
```

### トラブルシューティング

| 問題 | 解決策 |
|------|--------|
| `wrangler` が見つからない | `pnpm add -g wrangler` を実行 |
| KVネームスペースエラー | `worker/wrangler.jsonc` の `kv_namespaces` 設定を確認 |
| Webhookタイムアウト | Worker URLが公開アクセス可能か確認 |
| ボットが応答しない | `/status` コマンドでキーの有効性を確認 |
| プラグインがロードされない | `~/.config/opencode/plugin/telegram-notify.js` の存在を確認 |

---

## 開発

### 前提条件

- Node.js 22+
- pnpm 10+
- Cloudflareアカウント（Worker開発用）
- Telegram Bot Token（[@BotFather](https://t.me/BotFather) より取得）

### セットアップ

```bash
# 依存関係をインストール
pnpm install

# KVネームスペースを作成
wrangler kv namespace create "USERS"

# worker/wrangler.jsoncにKV IDを追加

# ボットトークンを設定
wrangler secret put BOT_TOKEN

# ローカル開発を開始
pnpm dev
```

### 再デプロイ

変更後の再デプロイ:
```bash
cd worker && pnpm exec wrangler deploy
```

---

## プロジェクト構造

```
├── plugin/           # OpenCodeプラグイン（クライアント側）
│   ├── src/          # TypeScriptソース
│   └── dist/         # コンパイル済みJS出力 (telegram-notify.js)
├── worker/           # Cloudflare Worker / VPSサーバー（ボットバックエンド）
│   ├── src/          # Hono + Grammyを使用したボットロジック
│   └── test/         # Vitestテスト
├── scripts/          # インストールヘルパー (install.sh)
└── docs/             # ドキュメントと戦略文書
```

---

## ライセンス

[LICENSE.md](./LICENSE.md) を参照 — MIT License、元の作品からのフォークでAttribution付き。
