# uptime

Uptime and service monitoring with [Uptime Kuma](https://uptime.kuma.pet/) — a self-hosted alternative to UptimeRobot / Pingdom, with multi-protocol checks, status pages, and notifications across 90+ channels.

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `uptime-kuma` | `louislam/uptime-kuma:latest` | Web UI, monitor scheduler, notification dispatch, status pages |

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse-proxy access via NPM |

---

## Ports

| Host port (`.env` var) | Container port | Bind | Purpose |
|------------------------|----------------|------|---------|
| `${UPTIME_PORT}` (`3001`) | `3001` | all interfaces | Web UI |

Published on all interfaces so it's reachable on your LAN out of the box. To keep it behind a reverse proxy / loopback only, prefix the mapping in `compose.yaml` with `127.0.0.1:`.

---

## Volumes

| Host path (`.env` var) | Container path | Mode | Purpose |
|------------------------|----------------|------|---------|
| `${UPTIME_DATA}` | `/app/data` | rw | SQLite DB (monitors, history, status pages, users), cert cache |
| `/var/run/docker.sock` | `/var/run/docker.sock` | **ro** | Docker socket — read-only; used by the "Docker Container" monitor type to inspect container status |

> The socket is mounted **read-only**. Uptime Kuma only issues list/inspect calls for "Docker Container" monitors — it never manages containers — so `:ro` is the right mount. If you don't use Docker Container monitors at all, you can remove the mount entirely.

---

## Environment Variables

### `.env` (Compose interpolation)

| Variable | Example | Purpose |
|----------|---------|---------|
| `TIMEZONE` | `America/New_York` | Container TZ |
| `UPTIME_PORT` | `3001` | Host port published for the web UI (container listens on `3001`) |
| `UPTIME_DATA` | `/path/to/uptimekuma` | Host bind for app data |

### Hardcoded in `compose.yaml`

| Variable | Value | Purpose |
|----------|-------|---------|
| `UMASK` | `0022` | Standard file-creation mask inside the container |

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme |
|----------|---------------|---------------|--------|
| `uptime.example.com` | `uptime-kuma` | `3001` | `http` |

> **Enable WebSocket support in NPM** — Uptime Kuma uses WebSockets for live monitor status updates. Without it, the dashboard loads but doesn't refresh.

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing)

### Bring up

```bash
cd /opt/stacks/uptime
cp .env.example .env
nano .env  # set UPTIME_DATA and TIMEZONE

# Pre-create local dir (match your ${UPTIME_DATA})
sudo mkdir -p /path/to/uptimekuma

docker compose up -d
```

### First-run setup

1. Open the app (e.g. `https://uptime.example.com` via NPM, or `http://<host-ip>:3001` directly).
2. Create the **first admin user** — Uptime Kuma has no default credentials; the first browser hit prompts for account creation.
3. Add monitors for the services you care about:
   - **HTTP(s)** for each proxied service (e.g. each `*.example.com` URL)
   - **Docker Container** for direct container health (uses the docker socket)
   - **TCP Port** for non-HTTP services
   - **Ping** for your NAS units
4. Configure notification channels under Settings → Notifications (Telegram, Discord, email, ntfy, etc.) and assign them to monitors as needed.
5. Optional: create a public **status page** under Status Pages → New Status Page. Useful for a single-URL "is the homelab up" view.

### Recommended monitors

A reasonable starter set:

| Monitor type | Target | Why |
|--------------|--------|-----|
| HTTP(s) | each proxied service URL | End-to-end NPM → upstream chain |
| Docker Container | `npm` | If NPM dies, every other HTTP monitor fails — a dedicated container check tells you it's NPM specifically |
| Docker Container | each DB container (`forgejo-db`, `paperless-db`, etc.) | DB outages aren't always visible from HTTP if the app caches |
| Ping | each NAS (e.g. `nas-1`, `nas-2`) | NAS reachability — NFS hangs on the host if these go down |
| TCP Port | `<adguard-host>:53` | DNS is a single point of failure for the whole network |

---

## Backup

Paths to include in Kopia / Borgmatic snapshots:

- `${UPTIME_DATA}` — SQLite DB with all monitors, history, status pages, notification configs

> The DB grows steadily with monitor history. Default retention is unlimited; under Settings → Monitor History you can cap retention to keep the DB size reasonable (~90 days is plenty for most use). Pruning happens during normal operation; no separate maintenance task.

---

## Notes & Gotchas

- **Monitoring its own host.** Uptime Kuma runs on the same host as everything it monitors, which means a host outage takes down the monitoring tool too. For true external monitoring, run a second Uptime Kuma instance on a different host (e.g. a cheap VPS) and have it ping this homelab. The local instance is still useful for fine-grained intra-homelab health visibility.
- **Docker socket is read-only.** Mounted `:ro` — Uptime Kuma only issues list/inspect calls for "Docker Container" monitors. If you don't use those monitors, remove the mount entirely.
- **No healthcheck on Uptime Kuma itself.** Defining a healthcheck on a tool whose job is healthchecks would be ironic. It's externally monitored — either by a second Uptime Kuma instance, by NPM's own upstream health-tracking, or just "did the dashboard load when you looked at it".
- **WebSocket dependency.** Uptime Kuma is essentially unusable without working WebSockets — the dashboard auto-refreshes everything via WS. If you ever see stale data, the first thing to check is NPM's WebSocket support on this proxy host.

---

## References

- [Uptime Kuma docs](https://uptimekuma.org/install-uptime-kuma-docker/)
- [Uptime Kuma GitHub](https://github.com/louislam/uptime-kuma)
- [Top-level README](../../README.md)
