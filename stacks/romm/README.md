# romm

Self-hosted ROM library management with [RomM](https://romm.app/) — browse, scan, edit metadata, and serve emulator ROMs via web UI and apps (e.g. EmulationStation, RomM mobile).

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `romm` | `rommapp/romm:latest` | Web UI, scanner, metadata fetcher, ROM API |
| `romm-db` | `mariadb:latest` | MariaDB database |

> `mariadb:latest` is intentional — matches the [official RomM docker-compose example](https://docs.romm.app/4.5.0/Getting-Started/Quick-Start-Guide/#build).

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse-proxy access via NPM |
| `guest_net` | External | For an optional guest-sharing sidecar (a separate Tailscale-based stack) to reach RomM as `http://romm:8080`. Harmless if unused |
| `romm` | Internal bridge | RomM ↔ MariaDB |

> RomM joins three networks: `proxy_net` for normal NPM access, `romm` for internal DB connectivity, and `guest_net` for an optional guest-sharing sidecar. The DB is reachable only on the internal `romm` network.
>
> **If you don't run a guest-sharing sidecar**, `guest_net` is just an empty external network — either create it (see [Prerequisites](#prerequisites)) or remove `guest_net` from both the `romm` service and the `networks:` block in `compose.yaml`.

---

## Ports

| Host port (`.env` var) | Container port | Bind | Purpose |
|------------------------|----------------|------|---------|
| `${ROMM_PORT}` (`8080`) | `8080` | all interfaces | Web UI + API |

Only `romm` publishes a port; the DB stays internal. Published on all interfaces so it's reachable on your LAN out of the box. To keep it behind a reverse proxy / loopback only, prefix the mapping in `compose.yaml` with `127.0.0.1:`. `8080` is a common port — change `ROMM_PORT` if it clashes.

---

## Volumes

### RomM

| Host path (`.env` var) | Container path | Purpose |
|------------------------|----------------|---------|
| `${ROMM_RESOURCES}` | `/romm/resources` | Cover art, screenshots, and metadata images fetched from IGDB / Screenscraper / etc. |
| `${ROMM_DATA}` | `/redis-data` | Embedded Redis state for background tasks |
| `${ROMM_CONFIG}` | `/romm/config` | Optional `config.yml` for advanced settings (see [Configuration](#configuration)) |
| `${ROMM_LIBRARY}` | `/romm/library` | **ROM library** — the actual game files, organized per [RomM folder structure](https://docs.romm.app/latest/Getting-Started/Folder-Structure/) |
| `${ROMM_ASSETS}` | `/romm/assets` | Uploaded saves, save states, screenshots from users |

### Database

| Host path (`.env` var) | Container path | Purpose |
|------------------------|----------------|---------|
| `${ROMM_MYSQL_DATA}` | `/var/lib/mysql` | MariaDB data — users, game metadata, library state |

> **Library on NAS, DB and small state on local NVMe.** Same pattern as Immich and Paperless: bulk files on NAS, database on fast local disk.

---

## Environment Variables

### `.env` (Compose interpolation)

#### General

| Variable | Example | Purpose |
|----------|---------|---------|
| `TIMEZONE` | `America/New_York` | Container TZ |
| `ROMM_PORT` | `8080` | Host port published for the web UI (container listens on `8080`) |

#### Database

| Variable | Example | Purpose |
|----------|---------|---------|
| `MARIADB_DATABASE` | `romm` | MariaDB database name |
| `MARIADB_USER` | `romm-user` | MariaDB application user |
| `MARIADB_PASSWD` | *(secret)* | MariaDB password for `MARIADB_USER`. Note: spelled `PASSWD`, not `PASSWORD` — RomM convention. Interpolated into MariaDB's actual env var `MARIADB_PASSWORD` inside compose |
| `MARIADB_ROOT_PASSWORD` | *(secret)* | MariaDB root password |

#### Secrets

| Variable | Purpose |
|----------|---------|
| `ROMM_AUTH_SECRET_KEY` | Session-signing secret. Generate with `openssl rand -hex 32` |

#### Metadata Providers (all optional, but recommended)

| Variable | Purpose | Get key from |
|----------|---------|--------------|
| `IGDB_CLIENT_ID` / `IGDB_CLIENT_SECRET` | IGDB metadata (preferred primary provider) | [Twitch developer console](https://docs.romm.app/4.5.0/Getting-Started/Metadata-Providers/#igdb) |
| `SCREENSCRAPER_USER` / `SCREENSCRAPER_PASSWORD` | Screenscraper.fr account | [Screenscraper account page](https://docs.romm.app/4.5.0/Getting-Started/Metadata-Providers/#screenscraper) |
| `MOBYGAMES_API_KEY` | MobyGames API key | [MobyGames API docs](https://docs.romm.app/4.5.0/Getting-Started/Metadata-Providers/#mobygames) |
| `RETROACHIEVEMENTS_API_KEY` | RetroAchievements integration | [RA account page](https://docs.romm.app/latest/Getting-Started/Metadata-Providers/#retroachievements) |
| `STEAMGRIDDB_API_KEY` | SteamGridDB art (custom posters / heroes) | [SteamGridDB profile](https://docs.romm.app/latest/Getting-Started/Metadata-Providers/#steamgriddb) |

#### Volumes

See [Volumes](#volumes) above for every `ROMM_*` and `ROMM_MYSQL_DATA` variable.

### Hardcoded in `compose.yaml`

These are baked into `compose.yaml` rather than `.env` — tweak directly in compose if you need to change them.

| Variable | Value | Purpose |
|----------|-------|---------|
| `DB_HOST` | `romm-db` | Internal DB hostname (matches container name) |
| `ENABLE_RESCAN_ON_FILESYSTEM_CHANGE` | `true` | Auto-rescan library when files change |
| `ENABLE_SCHEDULED_RESCAN` | `true` | Periodic library rescan |
| `HASHEOUS_API_ENABLED` | `true` | Hasheous content-hash matching |
| `PLAYMATCH_API_ENABLED` | `true` | Playmatch metadata provider |
| `LAUNCHBOX_API_ENABLED` | `true` | LaunchBox metadata |
| `ENABLE_SCHEDULED_UPDATE_LAUNCHBOX_METADATA` | `true` | Periodic LaunchBox metadata refresh |
| `SCHEDULED_UPDATE_LAUNCHBOX_METADATA_CRON` | `0 4 * * *` | Daily at 4am |
| `FLASHPOINT_API_ENABLED` | `true` | Flashpoint (web games) metadata |
| `HLTB_API_ENABLED` | `true` | How Long to Beat integration |

---

## Configuration

RomM works out of the box with no config file. For advanced behavior — excluding folders from scans, mapping custom platform-folder names to RomM's platform IDs, tuning metadata/artwork provider priority, or EmulatorJS per-core options — RomM reads an optional `config.yml` from `${ROMM_CONFIG}` (mounted at `/romm/config`).

This stack ships a fully-commented reference at [`config.yml.example`](config.yml.example) (the upstream RomM default). To use it:

```bash
# Copy the template into your ROMM_CONFIG directory as config.yml
cp config.yml.example /path/to/romm/config/config.yml
nano /path/to/romm/config/config.yml   # uncomment only what you want to change
docker compose restart romm
```

Everything in the file is commented out by default, so an unedited copy changes nothing. Full schema: [RomM docs](https://docs.romm.app/latest/).

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme |
|----------|---------------|---------------|--------|
| `romm.example.com` | `romm` | `8080` | `http` |

> **Bump `client_max_body_size` in NPM** — RomM accepts ROM uploads via the web UI. ROM files for newer consoles (Switch, PS3) can be tens of GB. Set `100G` or higher if you'll upload via web. For bulk imports, drop files directly into `${ROMM_LIBRARY}` on the NAS and trigger a rescan — faster than HTTP upload.
>
> **Enable WebSocket support in NPM** — RomM uses WebSockets for live scan progress.

> If you run a separate guest-sharing sidecar (a Tailscale-based stack), guest access goes through it at a tailnet hostname, independent of NPM — NPM for primary users, the sidecar for shared guests. Not required for normal operation.

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing)
- `guest_net` external network (`docker network create guest_net` if missing) — or remove `guest_net` from the compose if you're not using a guest-sharing sidecar
- `romm` internal network is created automatically by compose
- Your NAS NFS share mounted on the host (e.g. `/mnt/nas`) for the library paths
- Generated values for `MARIADB_ROOT_PASSWORD`, `MARIADB_PASSWD`, and `ROMM_AUTH_SECRET_KEY`
- Optional but recommended: API keys for IGDB and at least one other metadata provider

### Bring up

```bash
cd /opt/stacks/romm
cp .env.example .env

# Generate secrets (fill the blank values in .env)
sed -i "s|MARIADB_ROOT_PASSWORD=.*|MARIADB_ROOT_PASSWORD=$(openssl rand -base64 32 | tr -d '\n')|" .env
sed -i "s|MARIADB_PASSWD=.*|MARIADB_PASSWD=$(openssl rand -base64 32 | tr -d '\n')|" .env
sed -i "s|ROMM_AUTH_SECRET_KEY=.*|ROMM_AUTH_SECRET_KEY=$(openssl rand -hex 32)|" .env

# Optionally add metadata provider keys
nano .env  # paste IGDB_CLIENT_ID, IGDB_CLIENT_SECRET, etc.

# Pre-create local dirs (match your ROMM_* paths)
sudo mkdir -p /path/to/romm/{resources,redisdata,config,mysqldata}

# Pre-create library tree on the NAS (RomM expects per-platform subdirs)
sudo mkdir -p /path/to/nas-roms/romm/rommsaves

chmod 600 .env

docker compose up -d
```

### First-run setup

1. Wait ~60s for MariaDB to initialize and RomM to run migrations.
2. Open the app (e.g. `https://romm.example.com` via NPM, or `http://<host-ip>:8080` directly).
3. Create the **first admin user** through the web UI.
4. **Organize the library:** RomM expects ROMs under platform-named subdirectories. See the [Folder Structure docs](https://docs.romm.app/latest/Getting-Started/Folder-Structure/). Example:

   ```
   /path/to/nas-roms/romm/
   ├── n64/
   │   ├── Mario 64.z64
   │   └── ...
   ├── snes/
   ├── psx/
   └── ...
   ```

5. Trigger an initial scan from the web UI (Settings → Library → Scan all). Watch progress via the WebSocket-driven UI.
6. Configure metadata providers under Settings → Providers — IGDB is the recommended primary; SteamGridDB adds nice cover art.

---

## Backup

Paths to include in Kopia / Borgmatic snapshots:

- `${ROMM_MYSQL_DATA}` — MariaDB (users, custom metadata, library index state)
- `${ROMM_RESOURCES}` — cover art and screenshots (~GB-scale; rebuildable from providers but slow)
- `${ROMM_CONFIG}` — `config.yml` if present
- `${ROMM_ASSETS}` — uploaded saves, save states (small but irreplaceable to users)
- `.env` — contains all the secrets

Skip:
- `${ROMM_DATA}` — embedded Redis state; ephemeral
- `${ROMM_LIBRARY}` — the ROMs themselves; bulk data on the NAS, backed up at NAS level (you can also re-rip from physical media if it ever comes to that)

> RomM's metadata DB is keyed by IGDB IDs and file hashes — restoring the DB without the ROM files at the matching paths produces orphan metadata. Always restore library + DB together.

---

## Notes & Gotchas

- **`mariadb:latest` is unpinned by design.** Matches the official RomM compose example. MariaDB generally handles major version jumps gracefully, but be aware that WUD-tracked image bumps occasionally surface 12.x → 13.x-style transitions. Read the MariaDB release notes before pulling if you see a major version change in WUD notifications.
- **MariaDB on local NVMe — non-negotiable.** `${ROMM_MYSQL_DATA}` on NFS is unsupported and breaks. Same constraint as every other database service in the homelab.
- **Library + DB must be restored together.** RomM keys metadata by IGDB IDs and file hashes. If the DB references files that aren't at the expected paths, you get orphaned metadata. Snapshot and restore as a unit.
- **`MARIADB_PASSWD` vs `MARIADB_PASSWORD`.** The misspelled `PASSWD` is RomM convention, not a typo to fix. Compose interpolates it into MariaDB's actual `MARIADB_PASSWORD` env. Renaming the variable breaks the password handoff between the two services.

---

## References

- [RomM docs — Quick Start](https://docs.romm.app/latest/Getting-Started/Quick-Start-Guide/)
- [RomM docs — Folder Structure](https://docs.romm.app/latest/Getting-Started/Folder-Structure/)
- [RomM docs — Metadata Providers](https://docs.romm.app/latest/Getting-Started/Metadata-Providers/)
- [RomM mobile app](https://github.com/rommapp/mobile-app)
- [Top-level README](../../README.md)
