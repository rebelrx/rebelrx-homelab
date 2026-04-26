# RebelRx Homelab

Curated Docker Compose stacks and self-hosting infrastructure for modern homelabs.

Designed to be **secure**, **portable**, **reproducible**, and **easy to recover** on a new machine.

<p align="center">
  <a href="https://docs.rebelrx.tech"><strong>📚 View Guides</strong></a> •
  <a href="#stacks"><strong>📦 Stacks</strong></a> •
  <a href="#getting-started"><strong>🚀 Getting Started</strong></a>
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
- Reverse proxy is the default access method

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
docker compose up -d
```

---

## Stacks

| Stack | Description | Services |
|-------|-------------|----------|
| [template](stacks/_template/) | Boilerplate for new stacks | — |
| [arr](stacks/arr/) | Media automation (TV, movies, music, books, comics) | Gluetun, qBittorrent, SABnzbd, Prowlarr, Sonarr, Radarr, Lidarr, LazyLibrarian, Mylar3, Bazarr, Overseerr, Whisparr |
| [audiobooks](stacks/audiobooks/) | Audiobook and podcast media server | Audiobookshelf |
| [authentik](stacks/authentik/) | SSO authentication and identity management | Authentik Server, Authentik Worker, PostgreSQL |
| [backup](stacks/backup/) | Backup management (NAS/remote and local Borg) | Kopia, Borgmatic |
| [books](stacks/books/) | Book and reading management | Calibre, Calibre-Web, Kavita |
| [cloudflared](stacks/cloudflared/) | Cloudflare Tunnel client | Cloudflared |
| [dns](stacks/dns/) | DNS filtering and ad blocking | AdGuard Home |
| [dockge](stacks/dockge/) | Docker stack management UI | Dockge |
| [filebrowser](stacks/filebrowser/) | Web-based file management | Filebrowser Quantum |
| [finance](stacks/finance/) | Personal budgeting | Actual Budget |
| [git](stacks/git/) | Self-hosted Git server | Forgejo, PostgreSQL |
| [guest](stacks/guest/) | Tailscale sidecar for guest access | Tailscale (RomM) |
| [hadockermon](stacks/hadockermon/) | Docker monitoring for Home Assistant | HA Dockermon, Docker Socket Proxy |
| [homebox](stacks/homebox/) | Home inventory management | Homebox |
| [homepage](stacks/homepage/) | Service dashboard | Homepage |
| [media](stacks/media/) | Media servers with NVIDIA GPU transcoding | Plex, Jellyfin |
| [mkdocs](stacks/mkdocs/) | Documentation site | MkDocs Material, Nginx |
| [monitor](stacks/monitor/) | Infrastructure monitoring | Grafana, Prometheus, Node Exporter |
| [nextcloud-aio](stacks/nextcloud-aio/) | Cloud storage and collaboration | Nextcloud AIO |
| [notes](stacks/notes/) | Note-taking and knowledge base | Trilium Notes, Joplin Server, PostgreSQL |
| [paperless](stacks/paperless/) | Document management with OCR | Paperless-ngx, Redis, PostgreSQL, Gotenberg, Tika |
| [pdf](stacks/pdf/) | PDF tools | BentoPDF |
| [photo](stacks/photo/) | Photo/video management with AI | Immich Server, Immich ML, Valkey, PostgreSQL |
| [portainer](stacks/portainer/) | Docker container management UI | Portainer |
| [proxy](stacks/proxy/) | Reverse proxy and SSL | Nginx Proxy Manager |
| [recipes](stacks/recipes/) | Recipe management | Mealie, PostgreSQL |
| [roms](stacks/roms/) | ROM library management | RomM, MariaDB |
| [search](stacks/search/) | Privacy-respecting metasearch engine | SearXNG, Valkey |
| [speedtest](stacks/speedtest/) | Network speed monitoring | Speedtest Tracker, Librespeed |
| [uptime](stacks/uptime/) | Uptime monitoring | Uptime Kuma |
| [wud](stacks/wud/) | Container update monitoring | What's Up Docker |

---

## Directory Layout

```text
.
├── .gitignore
├── .pre-commit-config.yaml
├── .secrets.baseline
├── README.md
└── stacks/
    ├── _template/
    │   ├── docker-compose.yml
    │   ├── .env.example
    │   └── README.md
    ├── arr/
    │   ├── docker-compose.yml
    │   ├── .env.example
    │   └── README.md
    ├── audiobooks/
    ├── authentik/
    ├── backup/
    ├── books/
    ├── cloudflared/
    ├── dns/
    ├── dockge/
    ├── filebrowser/
    ├── finance/
    ├── git/
    ├── guest/
    ├── hadockermon/
    ├── homebox/
    ├── homepage/
    ├── media/
    ├── mkdocs/
    ├── monitor/
    ├── nextcloud-aio/
    ├── notes/
    ├── paperless/
    ├── pdf/
    ├── photo/
    ├── portainer/
    ├── proxy/
    ├── recipes/
    ├── roms/
    ├── search/
    ├── speedtest/
    ├── uptime/
    └── wud/
```

Each stack directory contains at minimum:

- `docker-compose.yml` — the Compose file
- `.env.example` — template for required environment variables
- `README.md` — full documentation (services, env vars, ports, deployment)

Some stacks also include `docker-compose.env.example` for container-level secrets.

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
healthcheck → labels
```

### Environment Variables

List/array syntax for all environment variables:

```yaml
    environment:
      - TZ=${TIMEZONE}
      - PUID=${PUID}
```

### Port Mappings

Always quoted. Localhost-bound when behind a reverse proxy:

```yaml
    ports:
      - "127.0.0.1:${APP_PORT}:8080"
```

### WUD Labels

Every service includes What's Up Docker monitoring labels:

```yaml
    labels:
      - wud.watch=true
      - wud.trigger.docker.enable=false
```

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

Services use condition syntax instead of list form:

```yaml
    depends_on:
      db:
        condition: service_healthy
```

### General

- Avoid commented-out code or template boilerplate
- Avoid trailing slashes on volume paths
- Avoid inline comments where possible (docs links go in the header)
- 2-space indentation throughout

---

## Environment File Conventions

### `.env` (NOT committed)

Used **only** for Docker Compose interpolation:

- Host paths (volumes)
- Ports
- Timezone
- Non-secret configuration

```env
APP_PORT=8080
APP_DATA_DIR=/mnt/data/app
```

Committed as `.env.example`.

### `docker-compose.env` (NOT committed)

Used for **container environment variables**, including secrets for certain containers that require them:

- Passwords
- Tokens
- Application secrets

```env
POSTGRES_PASSWORD=super-secret
APP_SECRET_KEY=long-random-string
```

Committed as `docker-compose.env.example`.


---

## Networks

| Network | Type | Used by |
|---------|------|---------|
| `proxy_net` | External | Most stacks (reverse proxy access) |
| `media_net` | External | arr, media |
| `nextcloud-aio` | External | nextcloud-aio, proxy |
| `authentik` | Internal | authentik (Server ↔ Worker ↔ PostgreSQL) |
| `forgejo` | Internal | git (Forgejo ↔ PostgreSQL) |
| `guest_net` | External | guest (Tailscale sidecar ↔ RomM) |
| `joplin` | External | notes (Joplin Server ↔ PostgreSQL) |
| `monitor` | Bridge | monitor (Grafana ↔ Prometheus ↔ Node Exporter) |
| `searxng_net` | Internal | search (SearXNG ↔ Valkey) |

Create external networks before deploying:

```bash
docker network create proxy_net
docker network create media_net
docker network create guest_net
docker network create joplin
docker network create nextcloud-aio
```

---

## Reverse Proxy Model

- All proxied services attach to the shared `proxy_net` external network
- Services bind to `127.0.0.1` and are not directly accessible from the network
- Access is handled via **Nginx Proxy Manager** (NPM)
- NPM upstream config: container name as host, container port, HTTP scheme

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

- `docker-compose.yml`
- `README.md`
- `*.example`

---

## Bringing Up a Stack

```bash
cd stacks/<stack-name>
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
2. Install NVIDIA Container Toolkit (for media and photo stacks)
3. Clone this repository
4. Create external Docker networks (`proxy_net`, `media_net`, `guest_net`, `joplin`, `nextcloud-aio`)
5. Recreate `.env` and `docker-compose.env` files from `.example` templates
6. Restore application data from Kopia backups
7. Run `docker compose up -d` in each stack directory

No application data or secrets are stored in Git.

---

## Environment & Assumptions

These stacks were developed and tested against the following environment. They will work in other setups, but you may need to adjust accordingly:

- Docker Compose v2 on Linux
- A reverse proxy (Nginx Proxy Manager) handling all public-facing traffic
- Services bound to `127.0.0.1` and reached only through the reverse proxy
- Container updates monitored by WUD with email notifications
- An NVIDIA GPU with the Container Toolkit installed (only required for the `media` and `photo` stacks)

---

## Disclaimer

These configurations are provided as examples. Adjust networking, volumes, and security settings to fit your environment.

---

## License

MIT License — see [LICENSE](./LICENSE)
