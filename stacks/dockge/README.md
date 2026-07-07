# dockge

Web UI for managing Docker Compose stacks with [Dockge](https://dockge.kuma.pet/).

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `dockge` | `louislam/dockge:1` | Stack management UI (browse, edit, deploy, monitor compose files) |

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse-proxy access via NPM |

---

## Ports

| Host port (`.env` var) | Container port | Bind | Purpose |
|------------------------|----------------|------|---------|
| `${DOCKGE_PORT}` (`5001`) | `5001` | all interfaces | Web UI |

Published on all interfaces so the UI is reachable on your LAN out of the box. To keep it behind a reverse proxy / loopback only, prefix the mapping in `compose.yaml` with `127.0.0.1:`.

---

## Volumes

| Host path (`.env` var) | Container path | Mode | Purpose |
|------------------------|----------------|------|---------|
| `${DOCKGE_DATA}` | `/app/data` | rw | Dockge settings, user accounts, agent config |
| `${DOCKGE_COMPOSE}` | `/opt/stacks` | rw | **All compose stacks** — Dockge reads, edits, and writes here |
| `/var/run/docker.sock` | `/var/run/docker.sock` | rw | Docker daemon socket — required to control containers |

> Dockge needs the host's compose tree mounted at the path declared by `DOCKGE_STACKS_DIR` (set to `/opt/stacks` in compose env). The host-side path is `${DOCKGE_COMPOSE}`, which **must equal** `/opt/stacks` — Dockge displays and writes compose files using the container path (`/opt/stacks/<stack>/compose.yaml`), and if the host path differs, the paths it writes won't resolve for the host Docker daemon.

---

## Environment Variables

### `.env` (Compose interpolation)

| Variable | Example | Purpose |
|----------|---------|---------|
| `TIMEZONE` | `America/New_York` | Container TZ |
| `DOCKGE_PORT` | `5001` | Host port published for the web UI |
| `DOCKGE_DATA` | `/path/to/dockge/data` | Host bind mount for Dockge state |
| `DOCKGE_COMPOSE` | `/opt/stacks` | Host path to the compose tree Dockge manages — **must equal** the in-container `/opt/stacks` |

> `DOCKGE_STACKS_DIR=/opt/stacks` is set directly in `compose.yaml` (not via `.env`) and refers to the **inside-the-container** path. It must match the right-hand side of the `${DOCKGE_COMPOSE}:/opt/stacks` mount.

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme |
|----------|---------------|---------------|--------|
| `dockge.example.com` | `dockge` | `5001` | `http` |

> Dockge uses WebSockets for live container/log updates. Ensure NPM has "Websockets Support" enabled on this host, or the UI will load but never refresh.

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing)
- `${DOCKGE_COMPOSE}` (`/opt/stacks`) exists on the host and contains the stacks Dockge will manage

### Bring up

```bash
cd /opt/stacks/dockge
cp .env.example .env
nano .env

docker compose up -d
```

### First-run setup

1. Open the app (e.g. `https://dockge.example.com` via NPM, or `http://<host-ip>:5001` directly) and create the first admin account.
2. Dockge auto-discovers existing stacks under `/opt/stacks`. They'll appear in the UI immediately.
3. Optional: configure agents under **Settings → Docker Hosts** if you want to manage other hosts from this Dockge instance.

---

## Backup

Paths to include in Kopia / Borgmatic snapshots:

- `${DOCKGE_DATA}` — admin accounts, agent config, UI preferences
- `${DOCKGE_COMPOSE}` — all compose files for every stack (already covered by your normal `/opt` snapshots)

> Dockge is stateless beyond its own settings. Wiping `${DOCKGE_DATA}` and starting fresh is a 30-second recovery — re-create the admin account, and every existing stack reappears automatically because the compose files themselves are the source of truth.

---

## Notes & Gotchas

- **RW Docker socket.** Dockge manages container lifecycle, so the socket mount is intentionally read-write. This is one of a few stacks with RW socket access (alongside `portainer` and `forgejo-runner`'s indirect access via DinD). Anything that compromises the Dockge container has full Docker access on the host. Dockge is also a deliberate exception to the `no-new-privileges` default for the same reason.
- **Path consistency matters.** If `DOCKGE_COMPOSE` is changed to a path that doesn't match the in-container `/opt/stacks`, Dockge will show stacks but won't deploy/redeploy correctly, because the paths it writes (and tries to mount in stacks it manages) won't resolve on the host.
- **Coexistence with Portainer.** Both Dockge and Portainer manage Docker on this host. Generally fine — they use the same socket and don't fight over containers — but avoid editing the same stack from both UIs in quick succession to prevent stale-state confusion.

---

## References

- [Dockge docs](https://dockge.kuma.pet/)
- [Dockge GitHub](https://github.com/louislam/dockge)
- [Top-level README](../../README.md)
