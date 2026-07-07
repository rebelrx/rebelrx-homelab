# project-nomad

Self-contained offline "survival computer" with [Project N.O.M.A.D.](https://github.com/Crosstalk-Solutions/project-nomad) — Node for Offline Media, Archives, and Data. Bundles encyclopedic knowledge (Kiwix), local AI (Ollama + RAG), offline maps, medical references, and more behind a single Command Center UI. Designed to keep working when the internet doesn't.

> **Mastercontainer-style architecture.** This compose defines six services, but the `admin` Command Center can dynamically spawn additional containers (Ollama, Kiwix, Qdrant for RAG, etc.) at runtime when you install content packs through the UI. Similar pattern to Nextcloud AIO — the compose tree only shows the bootstrap.

---

## Services

| Service | Container | Image | Purpose |
|---------|-----------|-------|---------|
| `admin` | `nomad_admin` | `ghcr.io/crosstalk-solutions/project-nomad:latest` | Command Center web UI, API, container orchestrator |
| `dozzle` | `nomad_dozzle` | `amir20/dozzle:v10.0` | Real-time log viewer for all containers |
| `mysql` | `nomad_mysql` | `mysql:8.0` | MySQL database for the Command Center |
| `redis` | `nomad_redis` | `redis:7-alpine` | Redis cache |
| `updater` | `nomad_updater` | `ghcr.io/crosstalk-solutions/project-nomad-sidecar-updater:latest` | Pulls upstream releases and applies them to this compose tree |
| `disk-collector` | `nomad_disk_collector` | `ghcr.io/crosstalk-solutions/project-nomad-disk-collector:latest` | Reads host disk-usage metrics for the dashboard |

### Containers spawned by `admin` at runtime

When you install content packs or AI tooling through the Command Center, `admin` spawns additional containers via the Docker socket. Common ones include:

| Container | Purpose |
|-----------|---------|
| `nomad_ollama` | Local LLM inference (when AI Assistant is enabled with a local model) |
| `nomad_kiwix` | Wikipedia / Khan Academy / encyclopedic ZIM file server |
| `nomad_qdrant` | Vector database backing RAG (Retrieval-Augmented Generation) |

> The exact set depends on which features you enable. Don't try to manage these manually with `docker compose` — let the Command Center spawn/destroy them. Stopping `admin` will not stop the spawned containers.

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse-proxy access via NPM (admin and dozzle only) |
| `nomad` (named `project-nomad_default`) | Internal bridge | Admin ↔ MySQL ↔ Redis ↔ updater ↔ disk-collector ↔ spawned-containers |

> The `nomad` network is named `project-nomad_default` explicitly so spawned containers can join it. Admin and dozzle join `proxy_net`; everything else is internal-only.

---

## Ports

| Host port (`.env` var) | Container port | Bind | Purpose |
|------------------------|----------------|------|---------|
| `${NOMAD_PORT}` (`8080`) | `8080` | all interfaces | Command Center web UI |

Only `admin` publishes a host port; MySQL, Redis, updater, and disk-collector stay internal. Published on all interfaces so the Command Center is reachable on your LAN out of the box. To keep it behind a reverse proxy / loopback only, prefix the mapping in `compose.yaml` with `127.0.0.1:`. `8080` is a common port — change `NOMAD_PORT` if it clashes.

> **Dozzle** (also `8080` internally) stays proxy-only in this compose — reach it via NPM. To publish it directly, add a `ports:` mapping to the `dozzle` service on a *different* host port (both admin and dozzle listen on `8080` inside their containers, so they can't share a host port).

---

## Volumes

### Bind mounts

| Host path (`.env` var) | Container path | Used by | Mode | Purpose |
|------------------------|----------------|---------|------|---------|
| `${NOMAD_STORAGE_PATH}` | `/app/storage` | admin | rw | Content packs, downloaded knowledge, AI models, user data |
| `${NOMAD_STORAGE_PATH}` | `/storage` | disk-collector | rw | Disk-usage metrics |
| `${NOMAD_MYSQL_PATH}` | `/var/lib/mysql` | mysql | rw | MySQL data |
| `${NOMAD_REDIS_PATH}` | `/data` | redis | rw | Redis persistence |
| `${NOMAD_UPDATER_PATH}` | `/opt/project-nomad` | updater | rw | **The stack's own compose tree** — updater rewrites `compose.yaml` during version upgrades |
| `/` | `/host` | disk-collector | `ro,rslave` | Host root for filesystem-usage metrics (read-only; `rslave` propagation) |
| `/var/run/docker.sock` | `/var/run/docker.sock` | admin | rw | Required to spawn/manage sibling containers |
| `/var/run/docker.sock` | `/var/run/docker.sock` | dozzle | **ro** | Read-only — Dozzle only reads logs |
| `/var/run/docker.sock` | `/var/run/docker.sock` | updater | rw | Required to recreate the `admin` container during self-update |

### Named Docker volumes

| Volume | Used by | Purpose |
|--------|---------|---------|
| `nomad-update-shared` | admin, updater | Coordination state between admin and updater during upgrades |

> **`NOMAD_UPDATER_PATH` is unusual.** It mounts this compose stack's own directory inside the updater container. When you trigger an upgrade from the Command Center, the updater rewrites `compose.yaml` in place to pin new image tags, then signals admin to recreate. Don't manually edit `compose.yaml` while an upgrade is in progress.

---

## Environment Variables

### `.env` (Compose interpolation)

#### Secrets

| Variable | Purpose |
|----------|---------|
| `NOMAD_APP_KEY` | Application signing key. Generate with `openssl rand -base64 32` |
| `NOMAD_DB_PASSWORD` | MySQL password for the `nomad_user` account |
| `NOMAD_MYSQL_ROOT_PASSWORD` | MySQL root password |

#### General

| Variable | Example | Purpose |
|----------|---------|---------|
| `NOMAD_URL` | `https://nomad.example.com` | Public URL Command Center advertises. Used in share links and admin-generated URLs |
| `NOMAD_PORT` | `8080` | Host port published for the Command Center web UI |

#### Volumes

See [Volumes](#volumes) above for all `NOMAD_*_PATH` variables.

### Hardcoded in `compose.yaml`

Project NOMAD bakes most of its config directly into compose. Key values:

| Variable | Value | Purpose |
|----------|-------|---------|
| `NODE_ENV` | `production` | Express/Node runtime mode |
| `PORT` | `8080` | Command Center internal listen port |
| `LOG_LEVEL` | `info` | Admin log verbosity |
| `DB_HOST` | `mysql` | Internal DB hostname (matches container name) |
| `REDIS_HOST` | `redis` | Internal Redis hostname |
| `DOZZLE_ENABLE_ACTIONS` | `false` | Disables Dozzle's container start/stop buttons |
| `DOZZLE_ENABLE_SHELL` | `false` | Disables Dozzle's web-shell feature |
| `DISABLE_COMPRESSION` | `false` | HTTP response compression |

---

## Reverse Proxy (NPM)

Two NPM hosts proxy this stack:

| Hostname | Upstream host | Upstream port | Scheme | Purpose |
|----------|---------------|---------------|--------|---------|
| `nomad.example.com` | `nomad_admin` | `8080` | `http` | Command Center UI |
| `dozzle.example.com` | `nomad_dozzle` | `8080` | `http` | Live log viewer |

> **`NOMAD_URL` must match the Command Center hostname exactly.**
>
> **Enable WebSocket support in NPM on both hosts.** Command Center uses WebSockets for live status updates and container management events; Dozzle uses WebSockets for log streaming (it's basically a thin WebSocket UI over `docker logs -f`).
>
> **Bump `client_max_body_size` in NPM** on the Command Center host — Project NOMAD accepts large content-pack uploads (Kiwix ZIM files can be 50+ GB, AI models can be 10+ GB). `100G` is reasonable if you upload pre-staged files via the dashboard. Direct host-side downloads via the Command Center's download UI don't pass through NPM, so this only matters for uploads.

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing)
- `nomad` internal network created automatically by compose (named `project-nomad_default`)
- Generated values for `NOMAD_APP_KEY`, `NOMAD_DB_PASSWORD`, and `NOMAD_MYSQL_ROOT_PASSWORD`
- **x86_64 host** — Project NOMAD warns loudly on non-x86_64 architectures (ARM support is limited)
- For AI features: a GPU-backed host is strongly recommended (an NVIDIA GPU handles small/medium models comfortably)

### Bring up

```bash
cd /opt/stacks/project-nomad
cp .env.example .env

# Generate secrets
sed -i "s|NOMAD_APP_KEY=.*|NOMAD_APP_KEY=$(openssl rand -base64 32 | tr -d '\n')|" .env
sed -i "s|NOMAD_DB_PASSWORD=.*|NOMAD_DB_PASSWORD=$(openssl rand -base64 32 | tr -d '\n')|" .env
sed -i "s|NOMAD_MYSQL_ROOT_PASSWORD=.*|NOMAD_MYSQL_ROOT_PASSWORD=$(openssl rand -base64 32 | tr -d '\n')|" .env

nano .env  # set NOMAD_URL, confirm NOMAD_UPDATER_PATH points at this dir
chmod 600 .env

# Pre-create dirs (storage lives on the dedicated NVMe drive)
sudo mkdir -p /path/to/project-nomad/{mysql,redis}
sudo mkdir -p /path/to/nvme-storage/project-nomad/storage

docker compose up -d
```

### First-run setup

1. Wait ~60s for MySQL to initialize and the Command Center to run migrations.
2. Open the app (e.g. `https://nomad.example.com` via NPM, or `http://<host-ip>:8080` directly) — Project NOMAD has **no authentication** by default (upstream design choice — it's meant for offline/airgapped use). A reverse proxy + network gating (firewall / VPN / Tailscale) provide the access boundary.
3. Browse the Command Center to install content packs (Wikipedia ZIM, Khan Academy, etc.), configure AI Assistant, set up maps, and enable RAG.
4. Open the Dozzle host (e.g. `https://dozzle.example.com`) separately to view live logs for all containers.

### Updating

**Do not use `docker compose pull && docker compose up -d` to update Project NOMAD.** Project NOMAD maintains a compatibility matrix across its image fleet and applies version-coupled updates atomically via the `updater` sidecar:

1. Command Center → **Updates → Check for updates**
2. Apply via the UI. The updater rewrites this stack's `compose.yaml` to pin new image tags, then recreates `admin` (and any other affected services) in sequence.

WUD will still notify about new upstream images, but treat those notifications as "Project NOMAD has a new release available, go apply it via the Command Center" rather than "go run `docker compose pull`".

---

## Backup

Paths to include in Kopia / Borgmatic snapshots:

- `${NOMAD_MYSQL_PATH}` — Command Center DB (users, settings, content-pack metadata, notes/markers)
- `${NOMAD_STORAGE_PATH}` — content packs, downloaded knowledge, AI models, RAG knowledge base
- `${NOMAD_REDIS_PATH}` — small; ephemeral but cheap to back up
- `.env` — contains all three secrets

`${NOMAD_UPDATER_PATH}` is the compose tree itself, already part of any backup of your stacks directory.

> **Content packs can be huge.** ZIM files for full Wikipedia are tens of GB; AI models can be 10-100 GB. Consider excluding `${NOMAD_STORAGE_PATH}/downloads/` from frequent backups if storage cost is a concern — the content is re-downloadable from the Command Center, though doing so requires internet.

---

## Notes & Gotchas

- **Mastercontainer pattern.** Compose only defines six services, but `admin` spawns more at runtime. Containers like `nomad_ollama`, `nomad_kiwix`, and `nomad_qdrant` appear in `docker ps` but not in this compose file — they're managed by the Command Center. Don't try to `docker compose up` them individually.
- **Storage lives on a separate NVMe drive.** `${NOMAD_STORAGE_PATH}` should point at a dedicated drive, not the root drive — content packs and AI models run to hundreds of GB. If you're mounting a dedicated disk for it, carry a `RequiresMountsFor=` dependency in Docker's service drop-in so the stack won't write to a bare mountpoint if that disk is ever unmounted. The other `NOMAD_*_PATH` dirs (mysql, redis) can stay on the root/`/opt` filesystem.
- **The updater can rewrite `compose.yaml`.** When you trigger an upgrade via the Command Center, the `updater` sidecar rewrites this stack's compose file in place. If you have local customizations, they may be overwritten. Apply customizations as `compose.override.yaml` if you want them to survive upgrades.
- **No built-in authentication.** Project NOMAD is intentionally open by design (built for offline/airgapped scenarios). Anyone who can reach the Command Center has full access. Gate it at the network layer (firewall / VPN / reverse proxy). If you need finer-grained auth, consider putting Authentik in front via NPM forward-auth.
- **No `no-new-privileges` anywhere in this compose.** Project NOMAD spawns and manages containers, mounts host paths, and the disk-collector reads the host root — adding the flag may conflict with these operations. This follows the upstream-recommended config, and because the `updater` rewrites `compose.yaml` in place, local `security_opt` additions may not survive upgrades anyway. Tightening is left to your evaluation (via `compose.override.yaml`).
- **`mysql` is missing WUD labels intentionally.** WUD label management for the MySQL service is deferred — since the `updater` sidecar manages MySQL upgrades atomically with the rest of the stack, automated WUD notifications could be noise.
- **Internal updates vs WUD.** WUD watches the admin/dozzle/redis/updater/disk-collector images and will notify on new releases. The correct response is to apply updates via the Command Center, not via `docker compose pull` — Project NOMAD coordinates version-matched updates across its image fleet, and bypassing that can produce broken states.
- **x86_64 only.** Upstream install scripts warn loudly on non-x86_64 architectures. If you migrate to an ARM host, this stack will likely fail to start.

---

## References

- [Project NOMAD on GitHub](https://github.com/Crosstalk-Solutions/project-nomad)
- [Project NOMAD website](https://www.projectnomad.us/)
- [Installation guide (upstream)](https://github.com/Crosstalk-Solutions/project-nomad/blob/main/install/README.md)
- [Dozzle docs](https://dozzle.dev/)
- [Top-level README](../../README.md)
