# homebox

Self-hosted home inventory management with [Homebox](https://homebox.software/) — track what you own, where it lives, warranties, photos, and maintenance.

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `homebox` | `ghcr.io/sysadminsmedia/homebox:latest` | Web UI, API, SQLite-backed data store |

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse-proxy access via NPM |

---

## Ports

| Host port (`.env` var) | Container port | Bind | Purpose |
|------------------------|----------------|------|---------|
| `${HOMEBOX_PORT}` (`7745`) | `7745` | all interfaces | Web UI + API |

Published on all interfaces so the UI is reachable on your LAN out of the box. To keep it behind a reverse proxy / loopback only, prefix the mapping in `compose.yaml` with `127.0.0.1:`.

---

## Volumes

| Host path (`.env` var) | Container path | Purpose |
|------------------------|----------------|---------|
| `${HOMEBOX_DATA}` | `/data` | SQLite DB, uploaded images, attachments |

---

## Environment Variables

### `.env` (Compose interpolation)

| Variable | Example | Purpose |
|----------|---------|---------|
| `HOMEBOX_DATA` | `/path/to/homebox/homebox-data` | Host bind mount for app data |
| `HOMEBOX_PORT` | `7745` | Host port published for the web UI |

#### Secrets — **DO NOT CHANGE POST-INIT**

| Variable | Purpose |
|----------|---------|
| `HBOX_AUTH_API_KEY_PEPPER` | Per-instance pepper mixed into API-key hashing. Generate with `openssl rand -hex 32`. **Changing this invalidates every existing API key issued by this instance.** Back up `.env` after first boot. |

### Hardcoded in `compose.yaml`

These are baked into `compose.yaml` rather than `.env`. Tweak directly in compose if you need to change them:

| Variable | Value | Purpose |
|----------|-------|---------|
| `HBOX_LOG_LEVEL` | `info` | Log verbosity (`debug`, `info`, `warn`, `error`) |
| `HBOX_LOG_FORMAT` | `text` | Log format (`text` or `json`) |
| `HBOX_WEB_MAX_UPLOAD_SIZE` | `10` | Max upload size in MB — bump if uploading large photo attachments |
| `HBOX_OPTIONS_ALLOW_ANALYTICS` | `false` | Telemetry opt-out |

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme |
|----------|---------------|---------------|--------|
| `homebox.example.com` | `homebox` | `7745` | `http` |

> If you bump `HBOX_WEB_MAX_UPLOAD_SIZE` in compose, also bump NPM's `client_max_body_size` on this host — both limits apply, and the smaller one wins. NPM's default 1 MB will reject anything above that even if Homebox would accept it.

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing)
- Generated value for `HBOX_AUTH_API_KEY_PEPPER`

### Bring up

```bash
cd /opt/stacks/homebox
cp .env.example .env

# Generate the API-key pepper
echo "HBOX_AUTH_API_KEY_PEPPER=$(openssl rand -hex 32)" >> .env

nano .env  # set HOMEBOX_DATA path
chmod 600 .env

docker compose up -d
```

### First-run setup

1. Open the app (e.g. `https://homebox.example.com` via NPM, or `http://<host-ip>:7745` directly).
2. Click **Register** to create the first user — this becomes the group owner.
3. Optionally disable further registrations from **Profile → Settings** if this is a single-user instance.

---

## Backup

Paths to include in Kopia / Borgmatic snapshots:

- `${HOMEBOX_DATA}` — SQLite DB, all uploaded images, attachments
- `.env` — contains `HBOX_AUTH_API_KEY_PEPPER`; without it, all existing API keys break on restore

> Homebox stores everything inside `/data` — there's no separate DB service or external state. A snapshot of the data dir is a complete backup. SQLite hot snapshots are generally fine; for extra safety before any major upgrade, the app has a built-in export (Profile → **Tools → Export Inventory**) that produces a JSON+CSV bundle.

---

## Notes & Gotchas

- **`HBOX_AUTH_API_KEY_PEPPER` is one-way.** Once API keys are issued under a given pepper value, changing it invalidates all of them — any external integrations or scripts using those keys will need new keys generated. Treat it like an encryption key: generate once, back up, never rotate casually. The variable was added in a recent Homebox release; older instances may not have set it explicitly and were using an upstream-default value.

---

## References

- [Homebox docs — installation](https://homebox.software/en/installation)
- [Homebox docs — configuration](https://homebox.software/en/configure)
- [Top-level README](../../README.md)
