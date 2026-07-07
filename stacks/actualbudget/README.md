# actualbudget

Self-hosted personal budgeting with [Actual Budget](https://actualbudget.org/).

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `actual` | `docker.io/actualbudget/actual-server:latest` | Actual Budget server (web UI + sync) |

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse-proxy access via NPM |

---

## Ports

| Host port (`.env` var) | Container port | Bind | Purpose |
|------------------------|----------------|------|---------|
| `${ACTUAL_PORT}` (`5006`) | `5006` | all interfaces | Web UI + sync |

The port is published on all interfaces by default, so the web UI is reachable on your LAN out of the box. To keep it behind a reverse proxy / loopback only instead (e.g. NPM reaching it at `actual:5006` over `proxy_net`), prefix the mapping in `compose.yaml` with `127.0.0.1:` → `127.0.0.1:${ACTUAL_PORT}:5006`.

---

## Volumes

| Host path (`.env` var) | Container path | Purpose |
|------------------------|----------------|---------|
| `${ACTUAL_DATA_PATH}` | `/data` | Application data (budget files, SQLite database, user config) |

---

## Environment Variables

### `.env` (Compose interpolation)

| Variable | Example | Purpose |
|----------|---------|---------|
| `ACTUAL_DATA_PATH` | `/path/to/actual/data` | Host bind mount for application data |
| `ACTUAL_PORT` | `5006` | Host port published for the web UI |

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme |
|----------|---------------|---------------|--------|
| `actual.example.com` | `actual` | `5006` | `http` |

---

## Deployment

```bash
cd /opt/stacks/actualbudget
cp .env.example .env
nano .env

docker compose up -d
```

### First-run setup

On first launch, open the app through your reverse proxy (e.g. `https://actual.example.com`) and set the server password through the web UI. There's no env-var bootstrap; the password is stored in the data directory.

---

## Backup

Paths to include in Kopia / Borgmatic snapshots:

- `${ACTUAL_DATA_PATH}` — budget files, SQLite DB, server config

> Actual Budget uses SQLite under the hood. For a hot snapshot, stop the container briefly or rely on Actual's built-in export (Settings → Export) for an extra safety layer alongside the bind-mount snapshot.

---

## References

- [Upstream Docker docs](https://actualbudget.org/docs/install/docker/)
- [Top-level README](../../README.md)
