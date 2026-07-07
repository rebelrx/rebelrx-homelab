# audiobooks

Self-hosted audiobook and podcast server with [Audiobookshelf](https://www.audiobookshelf.org/).

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `audiobookshelf` | `ghcr.io/advplyr/audiobookshelf:latest` | Audiobook + podcast library, web UI, mobile sync API |

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse-proxy access via NPM |

---

## Ports

| Host port (`.env` var) | Container port | Bind | Purpose |
|------------------------|----------------|------|---------|
| `${AUDIOBOOKSHELF_PORT}` (`13378`) | `80` | all interfaces | Web UI + mobile sync API |

Published on all interfaces so the UI is reachable on your LAN out of the box (`13378` matches upstream's compose example). To keep it behind a reverse proxy / loopback only, prefix the mapping in `compose.yaml` with `127.0.0.1:`.

---

## Volumes

| Host path (`.env` var) | Container path | Purpose |
|------------------------|----------------|---------|
| `${AUDIOBOOKSHELF_CONFIG}` | `/config` | Application config, SQLite DB, user accounts |
| `${AUDIOBOOKSHELF_METADATA}` | `/metadata` | Cover art, generated metadata, cache |
| `${AUDIOBOOKS_PATH}` | `/audiobooks` | Audiobook library (NAS NFS) |
| `${PODCASTS_PATH}` | `/podcasts` | Podcast library (NAS NFS) |

---

## Environment Variables

### `.env` (Compose interpolation)

| Variable | Example | Purpose |
|----------|---------|---------|
| `TIMEZONE` | `America/New_York` | Container TZ |
| `AUDIOBOOKSHELF_PORT` | `13378` | Host port published for the web UI |
| `AUDIOBOOKSHELF_CONFIG` | `/path/to/audiobookshelf/config` | Host bind mount for config / DB |
| `AUDIOBOOKSHELF_METADATA` | `/path/to/audiobookshelf/metadata` | Host bind mount for metadata cache |
| `AUDIOBOOKS_PATH` | `/path/to/nas-media/audiobooks` | Host path for audiobook library |
| `PODCASTS_PATH` | `/path/to/nas-media/podcasts` | Host path for podcast library |

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme |
|----------|---------------|---------------|--------|
| `audiobookshelf.example.com` | `audiobookshelf` | `80` | `http` |

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing)
- Your NAS NFS share mounted on the host (e.g. at `/mnt/nas`; see the top-level `system/<hostname>/fstab.example`)

### Bring up

```bash
cd /opt/stacks/audiobooks
cp .env.example .env
nano .env

docker compose up -d
```

### First-run setup

1. Open the app (e.g. `https://audiobookshelf.example.com` via NPM, or `http://<host-ip>:13378` directly) and create the **root user** account.
2. Add libraries:
   - **Audiobooks** → `/audiobooks` (folder type: Books)
   - **Podcasts** → `/podcasts` (folder type: Podcasts)
3. Trigger an initial scan and verify items appear.

---

## Backup

Paths to include in Kopia / Borgmatic snapshots:

- `${AUDIOBOOKSHELF_CONFIG}` — SQLite DB, user accounts, library settings (essential for restore)
- `${AUDIOBOOKSHELF_METADATA}` — cover art and generated metadata (optional; rebuildable from a fresh scan but slow on a large library)

Library files (`${AUDIOBOOKS_PATH}`, `${PODCASTS_PATH}`) are bulk media on the NAS — handle their backup at the NAS level.

---

## Notes & Gotchas

- **NFS mount must be up.** If your NAS mount (e.g. `/mnt/nas`) isn't mounted, the container starts but `/audiobooks` and `/podcasts` will be empty inside the container. Verify with `mountpoint /mnt/nas` before deploying.
- **Port `80` inside the container.** The container listens on `80`; the stack publishes it on the host as `${AUDIOBOOKSHELF_PORT}` (`13378` by default, matching upstream's `13378:80` example). NPM can still proxy it via `proxy_net` regardless of the published host port.

---

## References

- [Audiobookshelf docs — Docker install](https://www.audiobookshelf.org/docs/#docker-compose-install)
- [Top-level README](../../README.md)
