# wud

Container image update monitoring with [What's Up Docker](https://getwud.github.io/wud/) — watches every container in the homelab and notifies on new images. Notify-only across the board, including WUD itself.

> **The single source of truth for "what needs updating".** Every other stack has WUD labels (`wud.watch=true`) that opt them in to monitoring; this stack is what actually does the watching and sends the notifications.

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `wud` (service key `whatsupdocker`) | `getwud/wud:latest` | Container image update monitor, web UI, notification dispatcher |

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse-proxy access via NPM |

---

## Ports

| Host port (`.env` var) | Container port | Bind | Purpose |
|------------------------|----------------|------|---------|
| `${WUD_PORT}` (`3000`) | `3000` | all interfaces | Web UI |

Published on all interfaces so it's reachable on your LAN out of the box. To keep it behind a reverse proxy / loopback only, prefix the mapping in `compose.yaml` with `127.0.0.1:`. `3000` is a common port — change `WUD_PORT` if it clashes.

---

## Volumes

| Host path (`.env` var) | Container path | Mode | Purpose |
|------------------------|----------------|------|---------|
| `${WUD_STORE}` | `/store` | rw | WUD persistent state — scan history, notification dedup state, dashboard data |
| `/var/run/docker.sock` | `/var/run/docker.sock` | **ro** | Docker daemon socket — read-only; WUD inspects container metadata to determine images and labels, never modifies containers directly |

> The `/store` volume keeps notification dedup state across restarts — without it, every restart can re-emit notifications for updates WUD has already flagged. The dashboard's recent-events view also survives restarts.

---

## Environment Variables

### `.env` (Compose interpolation)

#### General

| Variable | Example | Purpose |
|----------|---------|---------|
| `TIMEZONE` | `America/New_York` | Container TZ — used for scan-time log entries and notification timestamps |
| `WUD_PORT` | `3000` | Host port published for the web UI (container listens on `3000`) |
| `WUD_STORE` | `/path/to/wud/store` | Host bind for WUD persistent state |

#### SMTP notification trigger

| Variable | Example | Purpose |
|----------|---------|---------|
| `WUD_TRIGGER_SMTP_EMAIL_HOST` | `smtp.example.com` | SMTP server hostname |
| `WUD_TRIGGER_SMTP_EMAIL_PORT` | `587` | SMTP port |
| `WUD_TRIGGER_SMTP_EMAIL_USER` | *(secret)* | SMTP auth username |
| `WUD_TRIGGER_SMTP_EMAIL_PASS` | *(secret)* | SMTP auth password |
| `WUD_TRIGGER_SMTP_EMAIL_FROM` | `wud@example.com` | From address on outbound notification emails |
| `WUD_TRIGGER_SMTP_EMAIL_TO` | `you@example.com` | Recipient address (multiple separated by `;`) |

### Hardcoded in `compose.yaml`

| Variable | Value | Purpose |
|----------|-------|---------|
| `WUD_TRIGGER_SMTP_EMAIL_TLS_ENABLED` | `false` | Disables explicit TLS handshake. Most modern SMTP providers on port `587` use STARTTLS, which doesn't need this flag set. If you use a provider that requires TLS-from-connect (e.g. port `465` direct TLS), set to `true` in compose |

> `WUD_SERVER_PORT` is referenced only by the healthcheck (`curl -f http://localhost:${WUD_SERVER_PORT:-3000}/health`) and defaults to `3000` when unset.

---

## How WUD Watches Other Stacks

WUD discovers containers automatically via the Docker socket. Per-container labels control whether each container is watched and how WUD handles updates:

| Label | Purpose |
|-------|---------|
| `wud.watch=true` | Monitor this container's image for updates |
| `wud.watch=false` | Explicitly exclude (used on `gluetun` in the `arr` stack — and on `wud` itself, see below) |
| `wud.trigger.docker.enable=true` | **Auto-update**: WUD pulls the new image and recreates the container when it sees a newer tag |
| `wud.trigger.docker.enable=false` | **Notify-only**: WUD reports the update available, you apply it manually |
| `wud.tag.include=<regex>` | Restrict the update detection to tags matching the regex — e.g. `^16(\.\d+)*$$` pins to PostgreSQL 16.x and won't propose a breaking 17 upgrade |

**Notify-only across the entire homelab.** Every stack with `wud.watch=true` also has `wud.trigger.docker.enable=false` — updates surface as notifications, never as automatic pulls. The rationale: image releases occasionally include breaking changes (DB schema migrations, config format changes, env-var renames), and getting paged about them is far less disruptive than waking up to a crash-looping container.

**WUD itself uses `wud.watch=false`** — it's explicitly excluded from monitoring itself. WUD updates are entirely manual: `docker compose pull && docker compose up -d` whenever you want a new version. The reasoning is the same as for any other stack — read the upstream changelog before pulling, especially for a tool with this much access to the host.

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme |
|----------|---------------|---------------|--------|
| `wud.example.com` | `wud` | `3000` | `http` |

> **Enable WebSocket support in NPM** — WUD uses WebSockets for live scan progress and update events on the dashboard.

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing)
- An SMTP relay you can authenticate against — Gmail with app password, Fastmail, Mailgun, or a local relay all work. The relay only needs to send outbound to your inbox

### Bring up

```bash
cd /opt/stacks/wud
cp .env.example .env

# Fill in SMTP credentials
nano .env  # set WUD_TRIGGER_SMTP_EMAIL_* values
chmod 600 .env

# Pre-create the store dir (match your ${WUD_STORE})
sudo mkdir -p /path/to/wud/store

docker compose up -d
```

### First-run setup

1. Open the app (e.g. `https://wud.example.com` via NPM, or `http://<host-ip>:3000` directly) — the dashboard loads with the current list of watched containers (every container with `wud.watch=true`).
2. **No login required by default.** WUD trusts whatever's in front of it (in this deployment, a reverse proxy on a private network). If you want app-level auth, see the [WUD auth docs](https://getwud.github.io/wud/#/configuration/authentications/anonymous/).
3. **Send a test notification** to verify SMTP: dashboard → **Containers → click any container → Trigger** → SMTP. You should receive a test email within a few seconds.

### Updating WUD itself

WUD doesn't monitor itself (`wud.watch=false`), so no notification fires when a new WUD release lands. Periodically:

```bash
cd /opt/stacks/wud
docker compose pull
docker compose up -d
```

Watch the changelog at [github.com/getwud/wud/releases](https://github.com/getwud/wud/releases) — breaking changes are infrequent but worth knowing about.

---

## Backup

Paths to include in Kopia / Borgmatic snapshots:

- `${WUD_STORE}` — WUD's persistent state (scan history, notification dedup state, dashboard data)
- `.env` — contains SMTP credentials

> WUD can recover from a wiped store — the watched-containers list rebuilds on the next scan from the labels on other containers, and notifications start fresh. Losing the store just means the dashboard's recent-events view resets and you may briefly get one duplicated notification for any already-pending updates.

---

## References

- [WUD docs — quickstart](https://getwud.github.io/wud/#/quickstart/)
- [WUD docs — Docker watcher labels](https://getwud.github.io/wud/#/configuration/watchers/docker/)
- [WUD docs — triggers (SMTP, Discord, ntfy, MQTT, webhook, etc.)](https://getwud.github.io/wud/#/configuration/triggers/)
- [Top-level README](../../README.md)
