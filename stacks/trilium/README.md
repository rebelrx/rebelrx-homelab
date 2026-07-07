# trilium

Self-hosted hierarchical note-taking with [TriliumNext](https://github.com/TriliumNext/Trilium) — community-maintained fork of the original Trilium Notes. Browser + desktop apps, single-database multi-device workflow.

> **Different model from `joplin`.** Joplin Server is a sync target for the Joplin desktop/mobile apps (notes live client-side, server stores ciphertext). Trilium is web-first — notes live in the server-side database, accessed via browser or Trilium desktop clients that talk to the same backend. Pick based on whether you want client-side or server-side as the source of truth.

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `trilium` | `triliumnext/trilium:latest` | Trilium Notes web UI, sync API, embedded SQLite store |

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse-proxy access via NPM |

---

## Ports

| Host port (`.env` var) | Container port | Bind | Purpose |
|------------------------|----------------|------|---------|
| `${TRILIUM_PORT}` (`8080`) | `8080` | all interfaces | Web UI + sync API |

Published on all interfaces so it's reachable on your LAN out of the box. To keep it behind a reverse proxy / loopback only, prefix the mapping in `compose.yaml` with `127.0.0.1:`. `8080` is a common port — change `TRILIUM_PORT` if it clashes.

---

## Volumes

| Host path (`.env` var) | Container path | Purpose |
|------------------------|----------------|---------|
| `${TRILIUM_DATA_DIR}` | `/home/node/trilium-data` | SQLite DB, attachments, sync state, user config |

---

## Environment Variables

### `.env` (Compose interpolation)

| Variable | Example | Purpose |
|----------|---------|---------|
| `TIMEZONE` | `America/New_York` | Container TZ |
| `TRILIUM_PORT` | `8080` | Host port published for the web UI / sync API |
| `TRILIUM_DATA_DIR` | `/path/to/trilium` | Host bind for Trilium data |

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme |
|----------|---------------|---------------|--------|
| `trilium.example.com` | `trilium` | `8080` | `http` |

> **Bump `client_max_body_size` in NPM** — Trilium supports attachments (images, PDFs, embedded files). Default 1 MB is too small for normal use. `100M` is a reasonable upper bound.
>
> **Enable WebSocket support in NPM** — Trilium uses WebSockets for live sync between connected clients.

---

## Deployment

```bash
cd /opt/stacks/trilium
cp .env.example .env
nano .env  # set TRILIUM_DATA_DIR and TIMEZONE

# Pre-create the data dir owned by the container user (node = UID 1000)
sudo mkdir -p /path/to/trilium
sudo chown -R 1000:1000 /path/to/trilium

docker compose up -d
```

### First-run setup

1. Open the app (e.g. `https://trilium.example.com` via NPM, or `http://<host-ip>:8080` directly).
2. Choose **New document** to start a fresh Trilium instance, or **Sync from existing server** if migrating from another Trilium.
3. Set the **password** — used by both the web UI and any Trilium desktop/mobile clients that want to connect to this server.
4. Optional: install a [Trilium desktop client](https://github.com/TriliumNext/Trilium/releases) and configure it to sync against your Trilium URL.

---

## Backup

Paths to include in Kopia / Borgmatic snapshots:

- `${TRILIUM_DATA_DIR}` — **the only stateful path.** Contains:
  - `document.db` — the SQLite DB with all notes, attachments, revisions
  - Backup snapshots Trilium itself creates under `backup/`

> Trilium has a built-in automated backup feature that produces a daily snapshot of `document.db` inside `${TRILIUM_DATA_DIR}/backup/`. Trilium backups live within the data dir, so they're captured by Kopia automatically — but they're not a substitute for off-host backup. The combination (Trilium's daily local snapshots + Kopia's off-host snapshots of the whole dir) gives both granular recent recovery and long-term off-host safety.

---

## References

- [TriliumNext on GitHub](https://github.com/TriliumNext/Trilium)
- [Trilium docs (legacy, mostly still applicable to TriliumNext)](https://github.com/zadam/trilium/wiki)
- [Top-level README](../../README.md)
