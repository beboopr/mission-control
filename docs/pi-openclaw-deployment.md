# Raspberry Pi + OpenClaw Gateway Deployment

This guide documents a hardened production deployment of Mission Control on a Raspberry Pi (aarch64, 8 GB RAM) wired to a local OpenClaw gateway for multi-agent orchestration.

It is the companion to [SECURITY-HARDENING.md](./SECURITY-HARDENING.md) and shows the **concrete configuration** behind that checklist.

---

## Topology

```
┌────────────────────┐    HTTPS+HSTS     ┌─────────────────┐
│ Internet (phone)   │──────────────────▶│ Cloudflare Edge │
└────────────────────┘                   └────────┬────────┘
                                                  │ trycloudflare tunnel
                                                  ▼
┌────────────────────┐    WireGuard      ┌─────────────────┐
│ Tailnet            │──────────────────▶│ Raspberry Pi    │
└────────────────────┘                   │  ┌────────────┐ │
                                         │  │ MC :3030   │ │
┌────────────────────┐    plain HTTP     │  └─────┬──────┘ │
│ Home LAN           │──────────────────▶│        │loopback│
└────────────────────┘                   │        ▼        │
                                         │  ┌────────────┐ │
                                         │  │ OpenClaw   │ │
                                         │  │ Gateway    │ │
                                         │  │ 127.0.0.1: │ │
                                         │  │   18800    │ │
                                         │  └────────────┘ │
                                         └─────────────────┘
```

| Network path | URL example | Transport security | Application auth |
|---|---|---|---|
| Public internet | `https://<random>.trycloudflare.com` | TLS @ Cloudflare + HSTS | password + rate-limit |
| Tailnet | `http://100.x.x.x:3030` | WireGuard | password + rate-limit |
| Home LAN | `http://192.168.50.100:3030` | none (LAN-isolated) | password + rate-limit |

---

## Prerequisites

- Raspberry Pi OS Bookworm (aarch64), 8 GB RAM
- Node 22 LTS (Mission Control bundles its own node under `~/.openclaw/tools/node/bin/`)
- PM2 for the MC process
- `cloudflared` for the public tunnel
- UFW firewall enabled
- (Optional) Tailscale for remote access without exposing the public tunnel

---

## 1. Build Mission Control (standalone)

```bash
cd ~/services
git clone https://github.com/beboopr/mission-control.git mission-control-v2
cd mission-control-v2
pnpm install
pnpm build

# Copy static + public into standalone (Next.js 16 quirk)
cp -r .next/static  .next/standalone/.next/static
cp -r public        .next/standalone/public
```

Run with PM2:

```bash
pm2 start node --name mission-control \
  --cwd ~/services/mission-control-v2/.next/standalone \
  -- server.js
pm2 save
```

> **After every rebuild**, repeat the two `cp -r` commands. The standalone output also generates a fresh `.data/` directory, which means `AUTH_SECRET` and `API_KEY` regenerate and you must re-login.

---

## 2. Mission Control `.env` (production)

```bash
# Network
PORT=3030

# Allowed hostnames — block everything else
MC_ALLOWED_HOSTS=localhost,127.0.0.1,::1,192.168.50.100,100.112.100.3,*.trycloudflare.com

# Cookies
MC_COOKIE_SAMESITE=strict
# Leave MC_COOKIE_SECURE *unset* so MC auto-detects per request
# (secure cookies on HTTPS via Cloudflare, plain cookies on Tailnet/LAN HTTP).

# HSTS — adds Strict-Transport-Security to every response
MC_ENABLE_HSTS=1

# OpenClaw gateway target (loopback only)
OPENCLAW_HOME=/home/ghost/.openclaw
OPENCLAW_GATEWAY_HOST=127.0.0.1
OPENCLAW_GATEWAY_PORT=18800

# Optional gateway-related front-end hints
NEXT_PUBLIC_GATEWAY_OPTIONAL=true
NEXT_PUBLIC_GATEWAY_HOST=
NEXT_PUBLIC_GATEWAY_PORT=
```

Lock down permissions:

```bash
chmod 600 ~/services/mission-control-v2/.env
chmod 600 ~/services/mission-control-v2/.next/standalone/.env
chmod 600 ~/services/mission-control-v2/.next/standalone/.data/mission-control.db*
chmod 600 ~/services/mission-control-v2/.next/standalone/.data/.auto-generated
```

---

## 3. OpenClaw gateway

### Install / upgrade

```bash
cd ~/.openclaw
~/.openclaw/tools/node/bin/npm install --prefix . openclaw@latest

# OpenClaw expects its package under lib/node_modules/openclaw
mv ~/.openclaw/lib/node_modules ~/.openclaw/lib/node_modules.old-$(date +%s) 2>/dev/null || true
mv ~/.openclaw/node_modules     ~/.openclaw/lib/node_modules

# Wrapper
sudo ln -sf ~/.openclaw/bin/openclaw /usr/local/bin/openclaw
```

`~/.openclaw/bin/openclaw` (chmod 0755):

```bash
#!/usr/bin/env bash
set -euo pipefail
export NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
export OPENCLAW_NO_RESPAWN=1
exec "/home/ghost/.openclaw/tools/node/bin/node" \
     "/home/ghost/.openclaw/lib/node_modules/openclaw/dist/entry.js" "$@"
```

```bash
mkdir -p /var/tmp/openclaw-compile-cache
```

### Hardened `~/.openclaw/openclaw.json` (chmod 0600)

```json
{
  "agents": {
    "defaults": {
      "workspace": "~/.openclaw/workspace",
      "maxConcurrent": 4,
      "subagents": { "maxConcurrent": 8 },
      "sandbox": { "mode": "all" }
    }
  },
  "tools": {
    "profile": "minimal",
    "exec": { "security": "allowlist" },
    "fs":   { "workspaceOnly": true },
    "deny": ["group:automation", "group:runtime", "group:fs", "group:network"]
  },
  "logging": {
    "redactSensitive": "tools"
  },
  "channels": {
    "telegram": { "enabled": false }
  },
  "gateway": {
    "port": 18800,
    "mode": "local",
    "bind": "loopback",
    "auth": {
      "mode": "token",
      "token": "<256-bit hex — generate with `openssl rand -hex 32`>"
    },
    "controlUi": {
      "allowedOrigins": [
        "http://127.0.0.1:3030",
        "http://localhost:3030",
        "http://100.112.100.3:3030"
      ]
    }
  }
}
```

> Notes:
> - `tools.profile` accepts only `minimal | coding | messaging | full`. Older docs may show `restricted` — that fails validation in 2026.4.x.
> - `logging.redactSensitive` accepts only `off | tools`.
> - The gateway origin whitelist applies to the **HTTP control UI only**. Primary defense for the WS endpoint is the token challenge/response.

### Hardened `~/.openclaw/.env` (chmod 0600)

```bash
OPENCLAW_GATEWAY_TOKEN=<same 256-bit hex as openclaw.json gateway.auth.token>
# add other channel creds here as needed
# OPENAI_API_KEY=...
# OPENROUTER_API_KEY=...
```

Keep `TELEGRAM_BOT_TOKEN` **out of** `~/.openclaw/.env` if you have another bot polling the same token (e.g. a separate Telegram bot project) — Telegram returns `409 Conflict` to the second poller and OpenClaw will hot-loop until you remove it.

### systemd user unit

`~/.config/systemd/user/openclaw-gateway.service`:

```ini
[Unit]
Description=OpenClaw Gateway (hardened)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/home/ghost/.openclaw/tools/node/bin/node /home/ghost/.openclaw/lib/node_modules/openclaw/dist/entry.js gateway --port 18800
Restart=on-failure
RestartSec=5
KillMode=process
TimeoutStopSec=10

Environment=HOME=/home/ghost
Environment=TMPDIR=/tmp
Environment=NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
Environment=OPENCLAW_NO_RESPAWN=1
Environment=OPENCLAW_GATEWAY_PORT=18800
Environment=OPENCLAW_SERVICE_VERSION=2026.4.21
EnvironmentFile=%h/.openclaw/.env

# Sandboxing (subset that works in user-mode systemd on the Pi kernel)
NoNewPrivileges=true
PrivateTmp=true

# Resource limits
LimitNOFILE=8192
LimitNPROC=2048
MemoryMax=1G
TasksMax=512

[Install]
WantedBy=default.target
```

> Stricter flags (`ProtectSystem=strict`, `ProtectHome=read-only`, `RestrictNamespaces=true`, `RestrictAddressFamilies=...`) trigger `status=226/NAMESPACE` on Pi OS Bookworm in user-mode. They work fine in a system-level unit (with `User=ghost`) — promote the unit if you need the extra hardening.

```bash
loginctl enable-linger ghost   # so the user service survives logout
systemctl --user daemon-reload
systemctl --user enable --now openclaw-gateway
```

### Emergency kill switch

`~/.openclaw/bin/emergency-stop.sh` (chmod 0755):

```bash
#!/usr/bin/env bash
set +e
LOG=~/.openclaw/logs/emergency-stop.log
mkdir -p "$(dirname "$LOG")"
echo "=== $(date -Iseconds) emergency-stop triggered ===" >> "$LOG"
systemctl --user stop openclaw-gateway 2>>"$LOG"
for u in $(systemctl --user list-units --all --plain --no-legend 2>/dev/null \
            | awk '/^openclaw-agent-/ {print $1}'); do
  echo "stopping $u" >> "$LOG"
  systemctl --user stop "$u" 2>>"$LOG"
done
if systemctl --user is-active --quiet openclaw-gateway; then
  pkill -u "$USER" -f openclaw-gateway 2>>"$LOG"
fi
echo "=== complete ===" >> "$LOG"
```

`~/.config/systemd/user/openclaw-killswitch.path`:

```ini
[Unit]
Description=OpenClaw emergency-stop watcher
StartLimitIntervalSec=0

[Path]
PathExists=%h/.openclaw/KILL
Unit=openclaw-killswitch.service

[Install]
WantedBy=default.target
```

`~/.config/systemd/user/openclaw-killswitch.service`:

```ini
[Unit]
Description=OpenClaw emergency-stop action
StartLimitIntervalSec=0

[Service]
Type=oneshot
ExecStart=%h/.openclaw/bin/emergency-stop.sh
RemainAfterExit=no
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now openclaw-killswitch.path
```

**Use it:**

```bash
touch ~/.openclaw/KILL    # stops gateway + every openclaw-agent-*.service
rm ~/.openclaw/KILL       # then to come back online:
systemctl --user start openclaw-gateway
```

---

## 4. Wire MC to the gateway

After the gateway is running on `127.0.0.1:18800`, register it with Mission Control:

```bash
TOKEN=$(grep '^OPENCLAW_GATEWAY_TOKEN=' ~/.openclaw/.env | cut -d= -f2)
DB=~/services/mission-control-v2/.next/standalone/.data/mission-control.db

python3 - <<PYEOF
import sqlite3, time, os
token = os.environ.get("TOKEN") or open(os.path.expanduser("~/.openclaw/.env")).read().split("OPENCLAW_GATEWAY_TOKEN=")[1].split("\n")[0]
c = sqlite3.connect(os.path.expanduser("$DB"))
now = int(time.time())
c.execute("DELETE FROM gateways")
c.execute("""INSERT INTO gateways
  (name, host, port, token, is_primary, status, created_at, updated_at)
  VALUES (?, ?, ?, ?, ?, ?, ?, ?)""",
  ("primary", "127.0.0.1", 18800, token, 1, "healthy", now, now))
c.commit()
PYEOF

pm2 restart mission-control --update-env
```

Verify:

```bash
KEY=$(grep '^API_KEY=' ~/services/mission-control-v2/.next/standalone/.data/.auto-generated | cut -d= -f2)
curl -s -H "x-api-key: $KEY" \
  http://127.0.0.1:3030/api/status?action=capabilities \
  | python3 -m json.tool | grep gateway
# expect: "gateway": true
```

---

## 5. Firewall (UFW) — defense in depth

The gateway is loopback-bound, so external packets cannot reach it. The UFW rules below are belt-and-braces in case the bind ever drifts:

```bash
sudo ufw deny in on eth0       to any port 18800 proto tcp comment "openclaw loopback only"
sudo ufw deny in on tailscale0 to any port 18800 proto tcp comment "openclaw loopback only"
sudo ufw status verbose | grep 18800
```

Quick external probe to prove isolation:

```bash
# From a different host on the LAN or Tailnet
nc -zv <pi-ip> 18800
# expect: connection refused
```

---

## 6. Verification checklist

```bash
# Gateway alive
systemctl --user is-active openclaw-gateway          # active
curl -s http://127.0.0.1:18800/health                # {"ok":true,"status":"live"}

# Loopback isolation
timeout 2 bash -c 'echo > /dev/tcp/100.x.x.x/18800'  # connection refused

# Built-in security audit
openclaw security audit --deep

# Mission Control sees the gateway
curl -s -H "x-api-key: $API_KEY" \
     http://127.0.0.1:3030/api/status?action=capabilities | jq .gateway
# true

# Token did not leak to logs
grep "$TOKEN" ~/.pm2/logs/*.log ~/.openclaw/logs/*.log    # nothing

# File permissions
stat -c '%a %n' ~/.openclaw/openclaw.json ~/.openclaw/.env \
  ~/services/mission-control-v2/.env \
  ~/services/mission-control-v2/.next/standalone/.env \
  ~/services/mission-control-v2/.next/standalone/.data/mission-control.db
# all 600
```

---

## 7. Public access via Cloudflare Quick Tunnel

```bash
pm2 start ~/services/mission-control/start-tunnel.sh --name mc-tunnel
```

The tunnel URL (`https://<random>.trycloudflare.com`) changes on every restart and is auto-posted to your Telegram chat by your own bot wrapper. To accept it:

- The wildcard `*.trycloudflare.com` is already in `MC_ALLOWED_HOSTS`
- Cloudflare adds TLS + HSTS preload at the edge
- Your admin password + rate-limiter is the auth boundary

For a stable URL with a real auth gate, replace the quick tunnel with a **named tunnel + Cloudflare Access** (email SSO before the request even reaches MC).

---

## 8. What's intentionally *not* done

- No agents are migrated to the gateway by default; PM2-style standalone agents continue to run as-is.
- The `controlUi.allowedOrigins` setting only filters the HTTP dashboard — the WebSocket relies on token auth.
- Strict systemd sandbox flags are off (kernel/userns limitation on this Pi). Promote the unit to system-level if you need them.

---

## 9. Routine operations

| Task | Command |
|---|---|
| Stop everything fast | `touch ~/.openclaw/KILL` |
| Recover after kill | `rm ~/.openclaw/KILL && systemctl --user start openclaw-gateway` |
| Rotate gateway token | `openssl rand -hex 32`, update `openclaw.json` + `.openclaw/.env` + MC `gateways` row, restart both |
| Upgrade OpenClaw | `cd ~/.openclaw && npm install --prefix . openclaw@latest`, then move into `lib/node_modules` and restart |
| Rebuild MC | `pnpm build && cp -r .next/static .next/standalone/.next/static && cp -r public .next/standalone/public && pm2 restart mission-control` |
| Tail live audit | `journalctl --user -u openclaw-gateway -f` and `tail -f ~/.openclaw/logs/emergency-stop.log` |
