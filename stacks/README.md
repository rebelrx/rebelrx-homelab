# RebelRx Homelab

Curated Docker Compose stacks and self-hosting infrastructure for modern homelabs.

Designed to be **secure**, **portable**, **reproducible**, and **easy to recover** on a new machine.

> This is the **public mirror** of a private Forgejo repository. Secrets, real
> host/LAN/Tailscale addresses, and machine-specific values are stripped or
> replaced with placeholders before publishing. Every stack ships an
> `.env.example` you fill in for your own environment.

<p align="center">
  <a href="https://docs.rebelrx.tech"><strong>📚 View Guides</strong></a> •
  <a href="#stacks"><strong>📦 Stacks</strong></a> •
  <a href="#quick-start"><strong>🚀 Getting Started</strong></a>
</p>

---

![Docker](https://img.shields.io/badge/docker-ready-blue?logo=docker)
![Compose](https://img.shields.io/badge/docker--compose-supported-blue)
![License](https://img.shields.io/github/license/rebelrx/rebelrx-homelab)
![Last Commit](https://img.shields.io/github/last-commit/rebelrx/rebelrx-homelab)
![Repo Size](https://img.shields.io/github/repo-size/rebelrx/rebelrx-homelab)

---

## Documentation

Full setup guides and deep dives are available at:

👉 https://docs.rebelrx.tech

---

## Overview

This repository contains curated Docker Compose stacks used alongside the [RebelRx Guides](https://docs.rebelrx.tech).

Each stack is designed to be:

- 🔐 Secure by default
- 🔁 Reproducible
- 🧩 Modular
- 🧠 Easy to understand and extend

All sensitive data has been removed or generalized. Use the provided `.env.example` files to configure each stack for your environment.

---

## Repository Philosophy

- Everything reproducible lives in Git
- Secrets never enter Git
- Runtime data is never versioned
- Each stack is self-contained
- **Web UIs published on all interfaces by default** — reachable on your LAN out of the box; prefix a mapping with `127.0.0.1:` to keep it behind the reverse proxy only

This allows the entire homelab to be restored quickly by cloning this repo and recreating only the local `.env` files.

RebelRx homelab stacks prioritize:

- Simplicity over complexity
- Transparency over abstraction
- Real-world usability over perfection

Built for people who want to understand their infrastructure — not just run it.

---

## Quick Start

```bash
git clone https://github.com/rebelrx/rebelrx-homelab.git
cd rebelrx-homelab/stacks/<stack>

cp .env.example .env
# edit .env for your environment
docker compose up -d
```

> On the live host, stacks are deployed to `/opt/stacks/<stack>/` and persistent
> data lives under `/opt/data/<stack>/`. Compose files and their data are always
> in separate trees, never co-located. The repo's `stacks/<stack>/` directory
> mirrors the compose-file layout only — no data or secrets are versioned.

---

## Stacks

| Stack | Description | Services |
|-------|-------------|----------|
| [_template](stacks/_template/) | Boilerplate for new stacks | — |
| [actualbudget](stacks/actualbudget/) | Personal budgeting | Actual Budget |
| [adguardhome](stacks/adguardhome/) | DNS filtering and ad blocking | AdGuard Home |
| [arr](stacks/arr/) | Media automation (TV, movies, music, books, comics) | Gluetun, qBittorrent, SABnzbd, Prowlarr, Radarr, Sonarr, Lidarr, Whisparr, LazyLibrarian, Mylar3, Bazarr, Seerr |
| [audiobooks](stacks/audiobooks/) | Audiobook and podcast media server | Audiobookshelf |
| [authentik](stacks/authentik/) | SSO authentication and identity management | Authentik Server, Authentik Worker, PostgreSQL |
| [backup](stacks/backup/) | Backup management (NAS/remote and local Borg) | Kopia, Borgmatic |
| [bentopdf](stacks/bentopdf/) | PDF tools | BentoPDF |
| [books](stacks/books/) | Book and reading management | Calibre, Calibre-Web, Kavita |
| [cloudflared](stacks/cloudflared/) | Cloudflare Tunnel for outbound-only public access | cloudflared |
| [dockge](stacks/dockge/) | Docker stack management UI | Dockge |
| [documenso](stacks/documenso/) | Document signing platform | Documenso, PostgreSQL |
| [drawio](stacks/drawio/) | Diagramming and flowcharts | draw.io |
| [filebrowser](stacks/filebrowser/) | Web-based file management | Filebrowser Quantum |
| [forgejo](stacks/forgejo/) | Self-hosted Git server with CI runners | Forgejo, PostgreSQL, Forgejo Runner, Docker-in-Docker |
| [glances](stacks/glances/) | System + GPU metrics exporter (host networking → Home Assistant) | Glances |
| [guest](stacks/guest/) | Tailscale sidecar for guest access | Tailscale (RomM) |
| [homebox](stacks/homebox/) | Home inventory management | Homebox |
| [homepage](stacks/homepage/) | Service dashboard | Homepage |
| [immich](stacks/immich/) | Photo/video management with AI | Immich Server, Immich ML, Valkey, PostgreSQL + pgvecto.rs |
| [it-tools](stacks/it-tools/) | Client-side developer/IT utility collection (sharevb fork) | IT-Tools |
| [jellyfin](stacks/jellyfin/) | Media server with NVIDIA GPU transcoding | Jellyfin |
| [joplin](stacks/joplin/) | Note sync server | Joplin Server, PostgreSQL |
| [mealie](stacks/mealie/) | Recipe management | Mealie, PostgreSQL |
| [mkdocs](stacks/mkdocs/) | Documentation site publishing | Material for MkDocs, Nginx |
| [monitor](stacks/monitor/) | Infrastructure monitoring | Grafana, Prometheus, Node Exporter |
| [navidrome](stacks/navidrome/) | Music server with Subsonic API | Navidrome |
| [nextcloud-aio](stacks/nextcloud-aio/) | Cloud storage and collaboration | Nextcloud AIO |
| [npm](stacks/npm/) | Reverse proxy and SSL termination | Nginx Proxy Manager |
| [open-webui](stacks/open-webui/) | Web UI for Ollama on the LLM workstation | Open WebUI |
| [paperless](stacks/paperless/) | Document management with OCR | Paperless-ngx, Redis, PostgreSQL, Gotenberg, Tika |
| [plex](stacks/plex/) | Media server with NVIDIA GPU transcoding (host networking) | Plex |
| [portainer](stacks/portainer/) | Docker container management UI | Portainer |
| [project-nomad](stacks/project-nomad/) | Offline survival computer (knowledge, AI, maps) | Project N.O.M.A.D. Admin, Dozzle, MySQL, Redis, Updater, Disk Collector |
| [romm](stacks/romm/) | ROM library management | RomM, MariaDB |
| [seafile](stacks/seafile/) | File sync and share (public via edge VPS over Tailscale) | Seafile, MariaDB, Redis |
| [searxng](stacks/searxng/) | Privacy-respecting metasearch engine | SearXNG, Valkey |
| [snapotter](stacks/snapotter/) | File processing | SnapOtter, PostgreSQL, Redis |
| [sparkyfitness](stacks/sparkyfitness/) | Fitness, nutrition, and body-composition tracker | SparkyFitness Frontend, Server, MCP, PostgreSQL |
| [speedtest](stacks/speedtest/) | Network speed monitoring | Speedtest Tracker, Librespeed |
| [trilium](stacks/trilium/) | Note-taking and knowledge base | Trilium Notes |
| [uptime](stacks/uptime/) | Uptime monitoring | Uptime Kuma |
| [wud](stacks/wud/) | Container update monitoring | What's Up Docker |

---

## Directory Layout

On the live host, compose files live at `/opt/stacks/<stack>/` and persistent data at `/opt/data/<stack>/`. The repo's `stacks/` tree mirrors the compose-file layout only.

```text
.
├── .gitignore
├── .pre-commit-config.yaml
├── .secrets.baseline
├── README.md
└── stacks/
    ├── _template/
    │   ├── compose.yaml
    │   ├── .env.example
    │   └── README.md
    ├── actualbudget/
    │   ├── compose.yaml
    │   ├── .env.example
    │   └── README.md
    ├── adguardhome/
    ├── arr/
    ├── audiobooks/
    ├── authentik/
    ├── backup/
    ├── bentopdf/
    ├── books/
    ├── cloudflared/
    ├── dockge/
    ├── documenso/
    ├── drawio/
    ├── filebrowser/
    ├── forgejo/
    ├── glances/
    ├── guest/
    ├── homebox/
    ├── homepage/
    ├── immich/
    ├── it-tools/
    ├── jellyfin/
    ├── joplin/
    ├── mealie/
    ├── mkdocs/
    ├── monitor/
    ├── navidrome/
    ├── nextcloud-aio/
    ├── npm/
    ├── open-webui/
    ├── paperless/
    ├── plex/
    ├── portainer/
    ├── project-nomad/
    ├── romm/
    ├── seafile/
    ├── searxng/
    ├── snapotter/
    ├── sparkyfitness/
    ├── speedtest/
    ├── trilium/
    ├── uptime/
    └── wud/
```

Each stack directory contains at minimum:

- `compose.yaml` — the Compose file
- `.env.example` — template for required environment variables
- `README.md` — full documentation (services, env vars, ports, deployment)

Some stacks ship additional committed files (copied into place at deploy time):

- `paperless` — `docker-compose.env.example` (container-level secrets, passed via `env_file:`)
- `filebrowser` — `config.yaml.example` (app config)
- `monitor` — `prometheus.yml.example` (Prometheus scrape config)
- `romm` — `config.yml.example` (optional advanced settings)
- `mkdocs` — `Dockerfile` (extends the base image with the i18n plugin)
- `glances` — `glances.conf` (committed directly, not as `.example` — it's mounted read-only into the container and contains no secrets)

---

## Compose Conventions

All stacks follow a consistent set of conventions:

### Comment Headers

Every service has a descriptive comment and docs link above its service key:

```yaml
  # App Name (Description)
  # https://example.com/docs
  app:
    image: example/image:latest
```

### Key Ordering

Service keys follow a fixed order:

```
image → container_name → restart → network_mode → depends_on →
ports → environment → volumes → networks → deploy → shm_size →
security_opt → healthcheck → labels
```

### Environment Variables

List/array syntax for all environment variables:

```yaml
    environment:
      - TZ=${TIMEZONE}
      - PUID=${PUID}
```

### Port Bindings

Each stack's primary web UI is published on **all interfaces** by default
(`${STACK_PORT}:containerport`), so it's reachable on your LAN out of the box —
whether or not you run a reverse proxy. Every stack that publishes a port also
documents the one-line change to restrict it: prefix the mapping with
`127.0.0.1:` to keep it loopback-only (behind the reverse proxy), or bind it to
a specific interface.

> This is a deliberate difference from the private origin repo, which was
> proxy-only. The public stacks publish so they work standalone; the reverse
> proxy remains fully supported (NPM reaches services by container DNS on
> `proxy_net`, independent of host-port publishing).

**Internal-only services never publish a host port** — databases, Redis/Valkey,
Prometheus + node-exporter, Gotenberg/Tika, and message brokers stay on their
stack's internal network, reachable only by the app that needs them.

Some stacks use a deliberately different binding:

| Stack | Bound | Reason |
|-------|-------|--------|
| `npm` | `0.0.0.0:80`, `0.0.0.0:443`, `127.0.0.1:81` | Public reverse-proxy entrypoints; admin UI loopback-only |
| `adguardhome` | `0.0.0.0:53/tcp+udp` | LAN DNS resolver; must be reachable by clients |
| `plex` | `network_mode: host` | DLNA, GDM discovery, and Plex's port-mapping behavior |
| `glances` | `network_mode: host` | Host-accurate metrics; the listener is UFW-gated on the LAN |
| `nextcloud-aio` | `127.0.0.1:${AIO_MANAGEMENT_PORT}:8080` | Powerful mastercontainer admin UI — loopback-only; reach via SSH tunnel or NPM |
| `sparkyfitness` | frontend all-interfaces; MCP `127.0.0.1:${SPARKY_FITNESS_MCP_PORT}:3001` | MCP is an authenticated API into the full health dataset — loopback-only; the frontend publishes normally |
| `seafile` | `${TAILSCALE_IP}:8088` | Bound to the `tailscale0` interface so the public-edge VPS (running Caddy) can reach it over the tailnet. Not reachable from LAN or the public internet. |
| `documenso` | `${TAILSCALE_IP}:3000` | Same pattern — tailnet-only bind for the public-edge VPS to reach |
| `snapotter` | none (proxy-only) | App's internal port unconfirmed upstream; left unpublished with a documented one-liner to publish once known |

All port mappings are quoted strings where YAML could otherwise coerce the value.

### Hardening Defaults

Every container has `no-new-privileges` enabled:

```yaml
    security_opt:
      - no-new-privileges:true
```

Notable exceptions:

- **`plex`** — runs with `network_mode: host`; `no-new-privileges` is
  intentionally omitted (host networking already breaks namespace isolation,
  and Plex's setuid helpers can interact poorly with the flag).
- **`jellyfin`** — same exception, on different grounds (NVIDIA Container
  Toolkit compatibility on some driver versions).
- **`forgejo-dind`** — runs `privileged: true` because it provides the Docker
  daemon backing CI runners. Scoped to the internal `forgejo` network.
- **`immich-machine-learning`** — `no-new-privileges` omitted; the CUDA stack
  can conflict with the flag on some driver versions (the other Immich services
  keep it).
- **Upstream-managed stacks** (`project-nomad`, `sparkyfitness`) ship their own
  multi-service composes; hardening follows upstream and is left as-is to avoid
  breakage — for `project-nomad`, also because its self-updater rewrites the
  compose in place. `snapotter` rolled back `no-new-privileges` / `cap_drop`
  after runtime issues (documented in its README).
- **`open-webui`** runs as root by upstream design, so `cap_drop` isn't
  applicable — but it keeps `no-new-privileges`.

The `searxng` and `searxng-valkey` containers go further with `cap_drop: ALL`
plus a minimal `cap_add` allowlist (`CHOWN`, `SETGID`, `SETUID`, and
`DAC_OVERRIDE` for valkey).

### Docker Socket Mounts

The Docker socket is mounted **read-only** wherever possible:

```yaml
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
```

The exceptions that need read-write access (and why):

- **`portainer`** — manages containers, requires RW
- **`dockge`** — manages stacks, requires RW
- **`project-nomad` admin / updater** — spawns and updates sibling containers, requires RW
- **`forgejo-runner`** — communicates with `forgejo-dind` over TCP; does not
  mount the host socket at all

### WUD Labels

Every long-running service includes What's Up Docker monitoring labels:

```yaml
    labels:
      - wud.watch=true
      - wud.trigger.docker.enable=false
```

All watched services use `wud.trigger.docker.enable=false` — WUD notifies on
new images, pulls are applied manually. There are no auto-updating containers
in the homelab.

Services intentionally excluded from WUD via `wud.watch=false`:

- `gluetun` — VPN updates can break the tunnel; manual only
- `wud` itself — also excluded; updates applied manually like everything else
- `forgejo-dind` — DinD daemon, not a tracked app (no WUD labels)

### Healthchecks

Database and critical services include healthchecks with explicit timing:

```yaml
    healthcheck:
      test: pg_isready -U postgres
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
```

### Dependencies

Services use condition syntax instead of list form where practical:

```yaml
    depends_on:
      db:
        condition: service_healthy
```

### Image Version Pinning

Most stacks use `:latest` and let WUD report new releases as notifications.
This is the default and works for the vast majority of services.

A few stacks pin to specific versions and use the parameterized-tag pattern
so updates don't require compose edits:

```yaml
    image: ghcr.io/example/app:${APP_TAG:-2026.1.0}
```

The corresponding `.env` line:

```env
APP_TAG=2026.1.0
```

This is used where `:latest` has shipped breaking changes or one-way
migrations — currently:

- **`authentik`** — `:latest` has shipped regressions (e.g. a Redis
  initialization bug in a 2026.x release). Authentik also does one-way schema
  migrations between minors. Pin and read [release notes](https://goauthentik.io/docs/releases/)
  before bumping.

Updates for pinned stacks become: edit one line in `.env`, then `docker compose pull && docker compose up -d`. Take a database dump first for stacks with one-way migrations.

### General

- Avoid commented-out code or template boilerplate
- Avoid trailing slashes on volume paths
- Avoid inline comments where possible (docs links go in the header)
- 2-space indentation throughout

---

## Environment File Conventions

### `.env` (NOT committed)

The primary configuration file for every stack. Used both for **Compose
interpolation** and — where a stack uses `env_file:` — for passing values
straight into the container.

Holds:

- Host paths (volumes)
- Ports
- Timezone, UID/GID, and other non-secret runtime config
- **Secrets** (passwords, API keys, encryption passphrases, signing keys)
- Image version tags for pinned stacks (e.g. `AUTHENTIK_TAG`)

```env
APP_PORT=8080
APP_DATA_DIR=/path/to/app/data
APP_SECRET_KEY=long-random-string
APP_TAG=2026.1.0
```

`.env` files are mode `600` and never leave the host. Committed to the repo
as `.env.example` with placeholder values.

#### `.env.example` Conventions

Across every stack, `.env.example` files follow a consistent layout:

- **Grouped by purpose** — General → Secrets → Service-specific config →
  Volume paths → Reference-only
- **Placeholder paths** use the form `/path/to/<stack>/<purpose>` rather than
  the actual on-disk path, so the example doesn't reveal the deployment layout
- **Secrets are blank** with a generation command (`openssl rand ...`)
  documented inline
- **Required secrets** marked `:?` in compose are called out so it's obvious
  which fields block startup
- **Reference-only section** at the bottom holds commented-out variables that
  are documented for visibility but not consumed by compose (typically port
  numbers hardcoded in the image or upstream defaults)

#### Shared / Canonical Paths

When a single path is consumed by many services in the same stack (e.g. a
shared downloads directory used by both download clients and *arr apps), the
path is defined **once** under a canonical variable and other variables
reference it via indirection:

```env
# Canonical
TORRENTS_PATH=/path/to/nas-media/torrents

# Per-service references — automatically stay in sync
QBITTORRENT_TORRENTS=${TORRENTS_PATH}
RADARR_TORRENTS=${TORRENTS_PATH}
SONARR_TORRENTS=${TORRENTS_PATH}
```

This makes drift impossible by construction — changing one canonical line
updates every consumer at once. Used today in the `arr` stack for torrent
and Usenet download paths.

### `docker-compose.env` (NOT committed)

Used **only** in stacks where the upstream image specifically expects a
separate `.env`-style file passed via `env_file:` for container-level
variables — typically because the image documents that pattern and changing
it would create maintenance friction with upstream's update path.

Committed as `docker-compose.env.example`. Currently used by: `paperless`.

For all other stacks, container-level secrets live in `.env` and are passed
into containers either via `environment:` interpolation or `env_file: .env`.

---

## Networks

| Network | Type | Used by |
|---------|------|---------|
| `proxy_net` | External | Most stacks (reverse-proxy access) |
| `media_net` | External | `arr`, `jellyfin`, `plex` |
| `nextcloud-aio` | External | `nextcloud-aio`, `npm` |
| `guest_net` | External | `guest`, `romm` |
| `project-nomad_default` | External | `project-nomad`, `npm` |
| `authentik` | Internal (default) | `authentik` (Server ↔ Worker ↔ PostgreSQL) |
| `documenso` | Internal bridge | `documenso` (App ↔ PostgreSQL) |
| `forgejo` | Internal (default) | `forgejo` (Forgejo ↔ PostgreSQL ↔ Runner ↔ DinD) |
| `immich` | Internal bridge | `immich` (Server ↔ ML ↔ Valkey ↔ PostgreSQL) |
| `joplin` | Internal bridge | `joplin` (Joplin Server ↔ PostgreSQL) |
| `mealie` | Internal bridge | `mealie` (Mealie ↔ PostgreSQL) |
| `monitor` | Internal bridge | `monitor` (Grafana ↔ Prometheus ↔ Node Exporter) |
| `paperless` | Internal bridge | `paperless` (Webserver ↔ Redis ↔ PostgreSQL ↔ Gotenberg ↔ Tika) |
| `romm` | Internal bridge | `romm` (RomM ↔ MariaDB) |
| `seafile-net` | Internal bridge | `seafile` (Seafile ↔ MariaDB ↔ Redis) |
| `searxng` | Internal bridge | `searxng` (SearXNG ↔ Valkey) |
| `snapotter` | Internal bridge | `snapotter` (App ↔ PostgreSQL ↔ Redis) |
| `sparkyfitness_net` | Internal bridge | `sparkyfitness` (Frontend ↔ Server ↔ MCP ↔ PostgreSQL) |

Create external networks before deploying:

```bash
docker network create proxy_net
docker network create media_net
docker network create guest_net
docker network create nextcloud-aio
```

`project-nomad_default` is created automatically when the `project-nomad` stack first comes up. Internal networks are created automatically by `docker compose up`.

---

## Reverse Proxy Model

Most proxied services use the local **Nginx Proxy Manager** (`npm` stack) on the home machine:

- Services attach to the shared `proxy_net` external network
- NPM reaches each service by container DNS inside `proxy_net` (container name + container port) — independent of any host-port publishing
- Services also publish their web UI on the host for direct LAN access (see [Port Bindings](#port-bindings)); prefix a mapping with `127.0.0.1:` to make NPM the only path
- Termination and SSL are handled by NPM
- NPM upstream config: container name as host, container port, HTTP scheme

### Public-edge VPS bastion (Seafile, Documenso)

The `seafile` and `documenso` stacks use a different topology for **public** access:

- Caddy on a dedicated cloud VPS terminates public TLS
- Caddy proxies over Tailscale to the container on the home machine
- The home machine publishes the app's port to its `tailscale0` interface only — never to LAN or public
- DNS is set to **DNS-only** (no proxy/gray cloud) for the relevant records to preserve end-to-end privacy between user and VPS

This pattern keeps the home machine free of any public-facing port while still
allowing public access to selected services. See [stacks/seafile/README.md](stacks/seafile/README.md)
and [stacks/documenso/README.md](stacks/documenso/README.md) for the full topology and Caddy configuration.

---

## Git and Security

### Secret Protection

- `.env` and `docker-compose.env` are gitignored
- Only `.example` templates are committed
- Pre-commit hooks enforce no secrets in commits

### Pre-commit

```bash
pre-commit install
```

Hooks include:

- `detect-secrets` — scans for accidental secret commits
- Private key detection
- Whitespace and EOF normalization

---

## Creating a New Stack

```bash
cp -a stacks/_template stacks/my-new-stack
cd stacks/my-new-stack

cp .env.example .env
nano .env

docker compose up -d
```

Commit only:

- `compose.yaml`
- `README.md`
- `*.example`

---

## Bringing Up a Stack

```bash
cd /opt/stacks/<stack-name>
cp .env.example .env
cp docker-compose.env.example docker-compose.env  # if applicable
nano .env
nano docker-compose.env

docker compose up -d
```

---

## Disaster Recovery

To restore this homelab on a new machine:

1. Install Docker and Docker Compose
2. Install NVIDIA Container Toolkit (for `jellyfin`, `plex`, and `immich`)
3. Install Tailscale and join the tailnet (required for the `seafile` and `documenso` stacks' port binds)
4. Clone this repository to `/opt/stacks/`
5. Create external Docker networks (`proxy_net`, `media_net`, `guest_net`, `nextcloud-aio`)
6. Apply any host-level settings the stacks reference — notably the `net.ipv4.ip_nonlocal_bind=1` sysctl required for `seafile`'s (and `documenso`'s) Tailscale-IP port bind (see those stack READMEs)
7. Recreate `.env` and `docker-compose.env` files from `.example` templates
8. Restore application data from Kopia / Borgmatic backups
9. Run `docker compose up -d` in each stack directory

No application data or secrets are stored in Git.

---

## Environment & Assumptions

- This is the public GitHub mirror of a private, Tailscale-gated Git instance
- Remote access is restricted via Tailscale
- HTTPS and personal access tokens are used for Git pushes
- Container updates are monitored by WUD with email notifications
- Selected public-facing services are proxied via a dedicated edge VPS over Tailscale (see "Reverse Proxy Model")

---

## Disclaimer

These configurations are provided as examples. Adjust networking, volumes, and security settings to fit your environment.

---

## License

MIT License — see [LICENSE](./LICENSE)
