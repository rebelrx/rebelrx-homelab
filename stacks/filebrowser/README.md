# filebrowser

Web-based file management with [Filebrowser Quantum](https://github.com/gtsteffaniak/filebrowser) — a fork of the original Filebrowser with multi-source support, improved UI, and modern config.

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `filebrowserquantum` | `gtstef/filebrowser:stable` | Multi-source web file manager |

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse-proxy access via NPM |

---

## Ports

| Host port (`.env` var) | Container port | Bind | Purpose |
|------------------------|----------------|------|---------|
| `${FILEBROWSER_PORT}` (`8080`) | `80` | all interfaces | Web UI |

Published on all interfaces so the UI is reachable on your LAN out of the box. The container port `80` is hardcoded in the image. To keep it behind a reverse proxy / loopback only, prefix the mapping in `compose.yaml` with `127.0.0.1:`.

> **Security note.** Filebrowser has read/write access to broad host paths (see [Notes & Gotchas](#notes--gotchas)) — effectively host-shell-equivalent. Consider binding it to `127.0.0.1` (behind an authenticated reverse proxy) or to your Tailscale IP rather than all interfaces. Also note `8080` is a common port — change `FILEBROWSER_PORT` if it clashes on your host, and never set it to `80` (privileged, and collides with the `npm` stack).

---

## Volumes

| Host path (`.env` var) | Container path | Mode | Purpose |
|------------------------|----------------|------|---------|
| `${FILEBROWSER_HOME_PATH}` | `/hosthome` | rw | User home directory — first browseable source |
| `${FILEBROWSER_SRV_PATH}` | `/srv` | rw | Legacy `/srv` tree — second browseable source |
| `${FILEBROWSER_OPT_PATH}` | `/opt` | rw | Stacks and container data tree — third browseable source |
| `${FILEBROWSER_DATA_PATH}` | `/home/filebrowser/data` | rw | Filebrowser app state — `config.yaml`, user DB, file index |
| `${FILEBROWSER_TMP_PATH}` | `/home/filebrowser/tmp/` | rw | Temp working space (uploads in flight, archive extraction, etc.) |

> Filebrowser Quantum supports **multiple sources** in a single instance, defined inside `config.yaml`. Each source is a path Filebrowser exposes to users. The three source mounts here (`/hosthome`, `/srv`, and `/opt`) get registered as named sources in `config.yaml` and appear in the UI as separate browseable trees. To add another source, mount it in `compose.yaml` **and** register it in `config.yaml`.

---

## Environment Variables

### `.env` (Compose interpolation)

| Variable | Example | Purpose |
|----------|---------|---------|
| `TIMEZONE` | `America/New_York` | Container TZ |
| `FILEBROWSER_PORT` | `8080` | Host port published for the web UI (container listens on `80`) |
| `FILEBROWSER_HOME_PATH` | `/path/to/user-home` | Host path mounted as the home source |
| `FILEBROWSER_SRV_PATH` | `/path/to/srv` | Host path mounted as the `/srv` source |
| `FILEBROWSER_OPT_PATH` | `/path/to/opt` | Host path mounted as the `/opt` source |
| `FILEBROWSER_DATA_PATH` | `/path/to/filebrowserquantum/data` | Host bind for app state and `config.yaml` |
| `FILEBROWSER_TMP_PATH` | `/path/to/filebrowserquantum/tmp` | Host bind for temp / upload scratch |

> `FILEBROWSER_CONFIG=data/config.yaml` is set directly in `compose.yaml` (not via `.env`). The path is **relative to the working directory inside the container**, resolving to `/home/filebrowser/data/config.yaml`. On the host, that's `${FILEBROWSER_DATA_PATH}/config.yaml`.

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme |
|----------|---------------|---------------|--------|
| `filebrowser.example.com` | `filebrowserquantum` | `80` | `http` |

> Filebrowser handles uploads up to the size limit set in `config.yaml`. NPM's default `client_max_body_size` (1 MB) will reject large file uploads — bump it via NPM's "Advanced" → custom `client_max_body_size 10G;` (or whatever ceiling makes sense) on this host.

---

## `config.yaml`

Filebrowser Quantum **requires a `config.yaml`** — unlike the legacy Filebrowser, it doesn't run with implicit defaults. This stack ships a template at [`config.yaml.example`](config.yaml.example); copy it into the app data dir before first start:

```bash
cp config.yaml.example ${FILEBROWSER_DATA_PATH}/config.yaml
```

It registers the three source mounts from `compose.yaml` as named sources:

```yaml
server:
  database: "/home/filebrowser/data/database.db"
  cacheDir: "tmp"
  sources:
    - path: "/hosthome"
      name: "Home"
      config:
        defaultEnabled: true
        denyByDefault: false
        createUserDir: false
    - path: "/srv"
      name: "Srv"
      config:
        defaultEnabled: true
        denyByDefault: false
        createUserDir: false
    - path: "/opt"
      name: "Opt"
      config:
        defaultEnabled: true
        denyByDefault: false
        createUserDir: false
auth:
  adminUsername: admin
```

Each `sources` entry's `path` is the **container-side** path and must match a volume mount in `compose.yaml`. To add a source, mount it in `compose.yaml` **and** add a matching entry here. Full schema: [Filebrowser Quantum wiki — Configuration](https://github.com/gtsteffaniak/filebrowser/wiki/Configuration).

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing)
- `${FILEBROWSER_DATA_PATH}` exists with a `config.yaml` inside — copy the shipped [`config.yaml.example`](config.yaml.example) into it (see [`config.yaml`](#configyaml)); the container won't start cleanly without it

### Bring up

```bash
cd /opt/stacks/filebrowser
cp .env.example .env
nano .env

# Place the config before first start (required — no implicit defaults)
mkdir -p /path/to/filebrowserquantum/data
cp config.yaml.example /path/to/filebrowserquantum/data/config.yaml
nano /path/to/filebrowserquantum/data/config.yaml   # review sources / admin user

docker compose up -d
```

### First-run setup

1. Open the app (e.g. `https://filebrowser.example.com` via NPM, or `http://<host-ip>:8080` directly) and log in. Filebrowser Quantum prompts for an **initial admin account** on first visit if none exists in the user DB.
2. Verify all three sources (`home`, `srv`, and `opt`) appear in the source switcher.
3. Add additional users with scoped permissions as needed (e.g., a read-only user that only sees `opt`).

---

## Backup

Paths to include in Kopia / Borgmatic snapshots:

- `${FILEBROWSER_DATA_PATH}` — `config.yaml`, user DB, file index, sessions

The browseable trees (`${FILEBROWSER_HOME_PATH}`, `${FILEBROWSER_SRV_PATH}`, `${FILEBROWSER_OPT_PATH}`) are part of your normal `/home`, `/srv`, and `/opt` snapshots — they're already covered by Kopia (which reads `/opt` directly) and any home-directory backup you have.

`${FILEBROWSER_TMP_PATH}` is volatile scratch — exclude it from snapshots.

---

## Notes & Gotchas

- **Broad filesystem access — intentional.** Filebrowser has read/write access to the user home, `/srv`, and `/opt` (which includes every other stack's config, every `.env` with secrets, and the compose tree itself). This is intentional: Filebrowser is the homelab's remote sysadmin file manager, ideally accessed only over Tailscale and gated by Filebrowser's own auth. Treat the Filebrowser admin login as equivalent to host-shell access — which is why the [Ports](#ports) section recommends binding it to loopback or Tailscale rather than all interfaces.
- **Schema-breaking updates.** Filebrowser Quantum has changed its `config.yaml` schema between releases. Always read the upstream release notes before pulling a new image — a config that worked on the old version may fail validation on the new one and prevent the container from starting. WUD will notify on new images, but updates should be applied manually after reviewing the changelog.
- **`config.yaml` required.** Unlike the legacy Filebrowser image, Quantum has no implicit defaults. If `${FILEBROWSER_DATA_PATH}/config.yaml` is missing or invalid, the container exits at startup. The healthcheck will fail and `docker compose logs filebrowserquantum` will show the parse error.

---

## References

- [Filebrowser Quantum wiki — Getting Started](https://github.com/gtsteffaniak/filebrowser/wiki/Getting-Started)
- [Filebrowser Quantum wiki — Configuration](https://github.com/gtsteffaniak/filebrowser/wiki/Configuration)
- [Top-level README](../../README.md)
