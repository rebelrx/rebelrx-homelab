# books

Book and reading management — full Calibre desktop app, Calibre-Web for browser-based reading, and Kavita for comics/manga/audiobooks.

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `calibre` | `lscr.io/linuxserver/calibre:latest` | Full Calibre desktop app (KasmVNC web session); manages the master library |
| `calibre-web` | `lscr.io/linuxserver/calibre-web:latest` | Lightweight web reader/browser; reads the same library as Calibre |
| `kavita` | `lscr.io/linuxserver/kavita:latest` | Web reader for comics, manga, books, and audiobooks; independent library |

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse-proxy access via NPM |

---

## Ports

All three apps' web UIs are published on **all interfaces**, so they're reachable on your LAN out of the box. To keep any behind a reverse proxy / loopback only, prefix its mapping in `compose.yaml` with `127.0.0.1:`.

| Service | Host binding | Container | Purpose |
|---------|--------------|-----------|---------|
| Calibre | `0.0.0.0:${CALIBRE_PORT_HTTP}` (`8080`) | `8080` | KasmVNC desktop session |
| Calibre-Web | `0.0.0.0:${CALIBREWEB_PORT}` (`8083`) | `8083` | Web reader UI |
| Kavita | `0.0.0.0:${KAVITA_PORT}` (`5000`) | `5000` | Web reader UI |

> Calibre also has an HTTPS KasmVNC port (`8181`) and an embedded content server (`8081`) for sending books to e-readers. Neither is published by default — `8181` is redundant when NPM terminates TLS, and the content server is opt-in (enable it inside the Calibre session, then add a `${CALIBRE_PORT_WEBSERVER}:8081` mapping). `8080` is a common port — change `CALIBRE_PORT_HTTP` if it clashes on your host.

---

## Volumes

### Calibre

| Host path (`.env` var) | Container path | Purpose |
|------------------------|----------------|---------|
| `${CALIBRE_CONFIG}` | `/config` | Calibre app config and KasmVNC session state |
| `${CALIBRE_PLUGINS}` | `/plugins` | Calibre plugins directory |
| `${CALIBRE_LIBRARY}` | `/books` | **Master library** on NAS — managed by Calibre, also read by Calibre-Web |

### Calibre-Web

| Host path (`.env` var) | Container path | Purpose |
|------------------------|----------------|---------|
| `${CALIBREWEB_CONFIG}` | `/config` | Calibre-Web settings DB, user accounts, OPDS config |
| `${CALIBRE_LIBRARY}` | `/books` | Same library as Calibre — read by Calibre-Web for browsing/serving |

### Kavita

| Host path (`.env` var) | Container path | Purpose |
|------------------------|----------------|---------|
| `${KAVITA_CONFIG}` | `/config` | Kavita app DB, user accounts, library metadata |
| `${KAVITA_LIBRARY}` | `/data` | Broad media root — Kavita libraries point at `/data/books`, `/data/comics`, `/data/audiobooks` |

> **Calibre and Calibre-Web share `${CALIBRE_LIBRARY}`** at `/books`. Calibre is the writer (adds/edits/converts), Calibre-Web is the reader. Both must agree on the same `metadata.db` location inside that directory.
>
> **Kavita is independent** — its `/data` mount intentionally exposes the wider media tree so a single Kavita instance can serve books, comics, and audiobooks from their existing locations under `${KAVITA_LIBRARY}`.

---

## Environment Variables

### `.env` (Compose interpolation)

#### General

| Variable | Example | Purpose |
|----------|---------|---------|
| `PUID` | `1000` | UID inside containers (linuxserver convention) |
| `PGID` | `1000` | GID inside containers |
| `TIMEZONE` | `America/New_York` | Container TZ |

#### Ports

| Variable | Example | Purpose |
|----------|---------|---------|
| `CALIBRE_PORT_HTTP` | `8080` | **Published** — Calibre KasmVNC session |
| `CALIBREWEB_PORT` | `8083` | **Published** — Calibre-Web UI |
| `KAVITA_PORT` | `5000` | **Published** — Kavita UI |
| `CALIBRE_PORT_HTTPS` | `8181` | Reference only — HTTPS KasmVNC (not published) |
| `CALIBRE_PORT_WEBSERVER` | `8081` | Reference only — Calibre content server (opt-in) |

#### Volumes

| Variable | Example | Purpose |
|----------|---------|---------|
| `CALIBRE_CONFIG` | `/path/to/calibre/config` | Calibre app config / KasmVNC state |
| `CALIBRE_PLUGINS` | `/path/to/calibre/plugins` | Calibre plugins |
| `CALIBREWEB_CONFIG` | `/path/to/calibre-web/config` | Calibre-Web settings DB |
| `KAVITA_CONFIG` | `/path/to/kavita/config` | Kavita app DB |
| `CALIBRE_LIBRARY` | `/path/to/nas-media/books/calibre-library` | Shared Calibre master library |
| `KAVITA_LIBRARY` | `/path/to/nas-media` | Kavita media root |

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme | Notes |
|----------|---------------|---------------|--------|-------|
| `calibre.example.com` | `calibre` | `8080` | `http` | KasmVNC desktop session — needs WebSocket support enabled in NPM |
| `calibreweb.example.com` | `calibre-web` | `8083` | `http` | |
| `kavita.example.com` | `kavita` | `5000` | `http` | |

> **Calibre's KasmVNC requires WebSockets.** In NPM's host config, ensure "Websockets Support" is enabled — without it, the desktop session won't render and you'll see a blank or partially-loaded page.

---

## Implementation Details

A few things worth knowing about how these images are wired:

- **`shm_size: 1gb` on Calibre** — required for the embedded Chromium that renders the KasmVNC desktop. Reducing it causes browser tab crashes inside the session.
- **`DOCKER_MODS=linuxserver/mods:universal-calibre` on Calibre-Web** — installs the Calibre binaries inside Calibre-Web so it can convert between formats (epub ↔ mobi ↔ azw3 etc.). Without this mod, the "Convert" feature in Calibre-Web silently fails.
- **Library agreement** — Calibre and Calibre-Web both need to see the same `metadata.db` inside `/books`. If you ever move the library, update both bind mounts in lockstep, then re-point Calibre-Web at the new location via its admin settings.

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing)
- Your NAS NFS share mounted on the host (e.g. `/mnt/nas`; see the top-level `system/<hostname>/fstab.example`)
- Existing Calibre library at `${CALIBRE_LIBRARY}` (e.g. `/path/to/nas-media/books/calibre-library`) — or let Calibre create a new one on first run

### Bring up

```bash
cd /opt/stacks/books
cp .env.example .env
nano .env

docker compose up -d
```

### First-run setup

#### Calibre

1. Open the app (e.g. `https://calibre.example.com` via NPM, or `http://<host-ip>:8080` directly) — a KasmVNC session loads with the Calibre desktop app.
2. On first launch, Calibre prompts for a library location. Point at `/books` (which maps to `${CALIBRE_LIBRARY}` on the host).
3. If you have an existing library, copy/move it into `${CALIBRE_LIBRARY}` on the host **before** starting the container — Calibre will detect and open it automatically.

#### Calibre-Web

1. Open the app (e.g. `https://calibreweb.example.com`).
2. Default credentials: `admin` / `admin123` — change immediately under **Admin → Edit Users**.
3. **Database location**: `/books` (must match Calibre's library path inside the container).
4. Configure features (OPDS, Kobo sync, magic link login, etc.) under Admin settings as desired.

#### Kavita

1. Open the app (e.g. `https://kavita.example.com`).
2. Create the first admin account.
3. Add libraries pointing at the appropriate subpaths under `/data`:
   - **Books** → `/data/books`
   - **Comics** → `/data/comics`
   - **Audiobooks** → `/data/audiobooks` *(if you want Kavita to serve audiobooks alongside Audiobookshelf)*
4. Trigger an initial scan and verify items appear.

---

## Backup

Paths to include in Kopia / Borgmatic snapshots:

- `${CALIBRE_CONFIG}` — Calibre app preferences and KasmVNC state
- `${CALIBRE_PLUGINS}` — installed plugins (small but easy to lose)
- `${CALIBREWEB_CONFIG}` — user accounts, settings DB
- `${KAVITA_CONFIG}` — user accounts, library metadata, reading progress

The library files themselves (`${CALIBRE_LIBRARY}`, `${KAVITA_LIBRARY}`) are bulk media on the NAS — handle their backup at the NAS level.

> **Calibre's `metadata.db`** lives inside `${CALIBRE_LIBRARY}/metadata.db` and is the source of truth for the library's collections, tags, and book metadata. It's part of the NAS-level backup, but worth being aware that losing it is far worse than losing the book files themselves — the books are unstructured without it.

---

## References

- [LinuxServer.io — docker-calibre](https://docs.linuxserver.io/images/docker-calibre/)
- [LinuxServer.io — docker-calibre-web](https://docs.linuxserver.io/images/docker-calibre-web/)
- [LinuxServer.io — docker-kavita](https://docs.linuxserver.io/images/docker-kavita/)
- [Top-level README](../../README.md)
