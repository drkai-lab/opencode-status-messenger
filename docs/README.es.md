# OpenCode Telegram Notification Plugin

Recibe notificaciones en Telegram cuando tus sesiones de OpenCode se completen.

**Forked from** [Davasny/opencode-telegram-notification-plugin](https://github.com/Davasny/opencode-telegram-notification-plugin) | **Maintained by AppWizz by Kai**

## Arquitectura

Este proyecto consiste en dos componentes:

| Componente | Ubicación | Descripción |
|------------|-----------|-------------|
| **Plugin** (Cliente) | Tu máquina local o VPS donde corre OpenCode | Archivo JS ligero que monitorea eventos de sesión y envía notificaciones al Worker |
| **Worker** (Servidor) | Cloudflare Workers (recomendado, plan gratuito) o tu propia VPS | Bot de Telegram que recibe notificaciones del Plugin y entrega mensajes vía API de Telegram |

### ¿Dónde Deberías Ejecutar Esto?

- **Plugin**: Debe correr en la misma máquina que OpenCode (entorno de desarrollo local o VPS)
- **Worker**: Mejor desplegado en **Cloudflare Workers** (serverless, gratis hasta 100k peticiones/día). Alternativamente, puedes hostearlo en una VPS con Node.js.

---

## Características

- ✅ Notificaciones en tiempo real vía Telegram
- ✅ Seguimiento de finalización de sesión con métricas de duración
- ✅ Autenticación segura basada en clave (el chat ID nunca se almacena en el plugin)
- ✅ Claves de instalación revocables
- ✅ Compatible con API OpenCode 1.14+
- ✅ Ligero — un solo archivo JS, sin dependencias

---

## Inicio Rápido (Usando Bot Público)

La forma más fácil de comenzar usa el bot pre-desplegado:

1. Inicia un chat con [@opencodetelegramnotificationbot](https://t.me/opencodetelegramnotificationbot)
2. Envía `/start`
3. El bot te envía un comando de instalación como:
   ```bash
   curl -fsSL https://raw.githubusercontent.com/drkai-lab/opencode-status-messenger/main/scripts/install.sh | bash -s -- <YOUR_INSTALL_KEY>
   ```
4. Ejecuta ese comando en tu terminal
5. ¡Hecho! Tus sesiones de OpenCode ahora enviarán notificaciones a Telegram

---

## Instalación Manual del Plugin

Si prefieres instalación manual o estás usando un Worker personalizado:

### Paso 1: Obtén Tu Clave de Instalación

1. Chatea con [@opencodetelegramnotificationbot](https://t.me/opencodetelegramnotificationbot)
2. Envía `/start` para recibir tu clave única de instalación

### Paso 2: Descarga el Archivo del Plugin

```bash
# Crea el directorio del plugin
mkdir -p ~/.config/opencode/plugin

# Descarga el plugin compilado
curl -fsSL "https://raw.githubusercontent.com/drkai-lab/opencode-status-messenger/main/plugin/dist/telegram-notify.js" \
  -o /tmp/telegram-notify.js
```

### Paso 3: Configura el Plugin

Edita `/tmp/telegram-notify.js` y reemplaza estos marcadores de posición:

```javascript
var INSTALL_KEY = "__INSTALL_KEY__";      // Reemplaza con tu clave del bot
var WORKER_URL = "__WORKER_URL__";        // Reemplaza con la URL de tu Worker
```

Ejemplo:
```javascript
var INSTALL_KEY = "a1b2c3d4-e5f6-7890-abcd-ef1234567890";
var WORKER_URL = "https://opencode-telegram-bot.your-subdomain.workers.dev";
```

### Paso 4: Instala en el Directorio del Plugin

```bash
# Mueve el archivo configurado al directorio del plugin
mv /tmp/telegram-notify.js ~/.config/opencode/plugin/telegram-notify.js

# Verifica la instalación
ls -la ~/.config/opencode/plugin/telegram-notify.js
```

### Paso 5: Completado

Reinicia tu instancia de OpenCode o inicia la siguiente sesión, el plugin se cargará automáticamente.

**Momento de la notificación:**  
La notificación no se envía inmediatamente después de la instalación, sino **cuando OpenCode completa una tarea** vía Telegram. Es decir, recibirás el primer mensaje cuando ejecutes alguna tarea y esa sesión termine。

---

## Comandos (Bot de Telegram)

| Comando | Descripción |
|---------|-------------|
| `/start` | Obtener comando de instalación con tu clave única |
| `/revoke` | Generar nueva clave (invalida la anterior) |
| `/status` | Verificar si tu plugin está instalado y activo |
| `/help` | Mostrar mensaje de ayuda |

---

## Cómo Funciona

1. El bot genera una clave única de instalación para ti
2. El archivo del plugin se coloca en `~/.config/opencode/plugin/`
3. Cuando OpenCode termina una tarea, el plugin envía una notificación a la URL del Worker
4. El Worker procesa la petición y entrega un mensaje de Telegram vía la API

Tu ID de chat de Telegram nunca se almacena en el plugin — solo una clave revocable.

---

## Desinstalación

```bash
rm ~/.config/opencode/plugin/telegram-notify.js
```

Opcionalmente, envía `/revoke` al bot para invalidar tu clave.

---

## Seguridad

- Tu ID de chat nunca sale del servidor
- Puedes revocar tu clave en cualquier momento con `/revoke`
- El plugin solo contiene una clave UUID, no tu ID de chat
- Toda comunicación usa HTTPS

---

## Despliega Tu Propio Worker (VPS / Cloudflare)

Para control total y personalización, despliega tu propia instancia.

### Opción A: Cloudflare Workers (Recomendado — Gratis)

**Prerrequisitos:**
- Cuenta de Cloudflare
- Token de Bot de Telegram de [@BotFather](https://t.me/BotFather)

```bash
# Clona el repositorio
git clone https://github.com/drkai-lab/opencode-status-messenger.git
cd opencode-status-messenger

# Instala dependencias
pnpm install

# Inicia sesión en Cloudflare (solo primera vez)
cd worker && pnpm exec wrangler login

# Crea namespace KV para datos de usuario
pnpm exec wrangler kv namespace create "USERS"
# Copia el ID retornado y actualiza worker/wrangler.jsonc con él

# Despliega el Worker
pnpm exec wrangler deploy

# Establece secreto BOT_TOKEN
pnpm exec wrangler secret put BOT_TOKEN

# Configura webhook de Telegram
curl "https://api.telegram.org/bot<BOT_TOKEN>/setWebhook?url=https://<your-worker-url>/webhook"
```

### Opción B: Despliegue en VPS (Control Total)

**Prerrequisitos:**
- VPS Linux con Node.js 22+ y pnpm 10+ instalados
- Token de Bot de Telegram de [@BotFather](https://t.me/BotFather)

```bash
# Prepara el sistema
sudo apt update && sudo apt upgrade -y
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
npm install -g pnpm@latest

# Clona y configura
git clone https://github.com/drkai-lab/opencode-status-messenger.git
cd opencode-status-messenger
pnpm install

# Construye el plugin (para distribución)
cd plugin && pnpm build

# Configura entorno del worker
cd ../worker
cp .env.example .env.local
# Edita .env.local con tu BOT_TOKEN y otros ajustes

# Inicia con PM2 para gestión de procesos
npm install -g pm2
pm2 start src/index.ts --watch
pm2 save
```

### Solución de Problemas

| Problema | Solución |
|----------|----------|
| `wrangler` no encontrado | Ejecuta `pnpm add -g wrangler` |
| Error de namespace KV | Verifica que `worker/wrangler.jsonc` tenga configuración correcta de `kv_namespaces` |
| Timeout del webhook | Asegúrate de que tu URL del Worker sea accesible públicamente |
| Bot sin respuesta | Revisa el comando `/status` y verifica que la clave sea válida |
| Plugin no se carga | Confirma que el archivo existe en `~/.config/opencode/plugin/telegram-notify.js` |

---

## Desarrollo

### Prerrequisitos

- Node.js 22+
- pnpm 10+
- Cuenta de Cloudflare (para desarrollo del Worker)
- Token de Bot de Telegram (de [@BotFather](https://t.me/BotFather))

### Configuración

```bash
# Instala dependencias
pnpm install

# Crea namespace KV
wrangler kv namespace create "USERS"

# Agrega el ID KV a worker/wrangler.jsonc

# Establece token del bot
wrangler secret put BOT_TOKEN

# Inicia desarrollo local
pnpm dev
```

### Re-despliegue

Para re-desplegar después de cambios:
```bash
cd worker && pnpm exec wrangler deploy
```

---

## Estructura del Proyecto

```
├── plugin/           # Plugin de OpenCode (lado cliente)
│   ├── src/          # Fuente TypeScript
│   └── dist/         # Salida JS compilada (telegram-notify.js)
├── worker/           # Cloudflare Worker / servidor VPS (backend del bot)
│   ├── src/          # Lógica del bot con Hono + Grammy
│   └── test/         # Pruebas Vitest
├── scripts/          # Ayudas de instalación (install.sh)
└── docs/             # Documentos de documentación y estrategia
```

---

## Licencia

Ver [LICENSE.md](./LICENSE.md) — MIT License, forked from original con atribución.
