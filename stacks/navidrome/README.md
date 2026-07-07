# navidrome

Self-hosted music server with a clean web UI and full Subsonic API compatibility, via [Navidrome](https://www.navidrome.org/). Drop-in replacement for Plexamp's backend, with a much wider client ecosystem (Symfonium, Amperfy, play:Sub, Feishin, Tempo, Sonixd, etc.).

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `navidrome` | `deluan/navidrome:latest` | Music server + Subsonic API |

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse-proxy access via NPM |

---

## Ports

| Host port (`.env` var) | Container port | Bind | Purpose |
|------------------------|----------------|------|---------|
| `${NAVIDROME_PORT}` (`4533`) | `4533` | all interfaces | Web UI + Subsonic API |

Published on all interfaces so it's reachable on your LAN out of the box (Subsonic clients on the LAN can point straight at it). To keep it behind a reverse proxy / loopback only, prefix the mapping in `compose.yaml` with `127.0.0.1:`.

---

## Volumes

| Host path (`.env` var) | Container path | Mode | Purpose |
|------------------------|----------------|------|---------|
| `${NAVIDROME_DATA_PATH}` | `/data` | rw | SQLite DB, search index, cache, play counts, ratings, starred items |
| `${NAVIDROME_MUSIC_PATH}` | `/music` | **ro** | Music library (NAS NFS) |

> Read-only on the library mount is deliberate — Navidrome never needs to write back, and the `:ro` flag is cheap insurance against any future bug or misconfiguration.

---

## Environment Variables

### `.env` (Compose interpolation)

| Variable | Example | Purpose |
|----------|---------|---------|
| `NAVIDROME_PORT` | `4533` | Host port published for the web UI / Subsonic API |
| `NAVIDROME_DATA_PATH` | `/path/to/navidrome/data` | Host bind mount for persistent data |
| `NAVIDROME_MUSIC_PATH` | `/path/to/nas-media/music` | Host path for music library (mounted RO) |

### Container environment (set in `compose.yaml`)

Only the two non-default settings are declared; everything else inherits Navidrome's upstream defaults.

| Variable | Value | Purpose |
|----------|-------|---------|
| `ND_LISTENBRAINZ_ENABLED` | `true` | Enables per-user ListenBrainz scrobbling |
| `ND_ENABLEINSIGHTSCOLLECTOR` | `false` | Disables anonymous usage telemetry |

> `.env.example` also has commented placeholders for optional Spotify cover-art API, Last.fm scrobbling keys, and `ND_SCANSCHEDULE`. To activate them, uncomment in `.env` and extend `compose.yaml` to pass them into the container.

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme |
|----------|---------------|---------------|--------|
| `navidrome.example.com` | `navidrome` | `4533` | `http` |

> Enable "Websockets Support" on the NPM host for live UI updates.

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing)
- Your NAS NFS share mounted on the host (e.g. `/mnt/nas`; see the top-level `system/<hostname>/fstab.example`)
- Music library readable by UID/GID `1000:1000` (the user the container runs as)

### Bring up

```bash
cd /opt/stacks/navidrome
cp .env.example .env
nano .env

# Ensure data dir exists with correct ownership (match your ${NAVIDROME_DATA_PATH})
sudo mkdir -p /path/to/navidrome/data
sudo chown -R 1000:1000 /path/to/navidrome

docker compose up -d
```

### First-run setup

1. Watch the initial scan — it can take a while on large libraries:
   ```bash
   docker compose logs -f navidrome
   ```
2. Open the app (e.g. `https://navidrome.example.com` via NPM, or `http://<host-ip>:4533` directly) and create the **admin user** on first load.
3. Create additional user accounts as needed via the admin UI.
4. Each user enables ListenBrainz scrobbling individually under **Profile → Personal → ListenBrainz token**.
5. Configure Subsonic clients (Symfonium, Amperfy, etc.) with the same username + password — there's no separate API token.

---

## Backup

Paths to include in Kopia / Borgmatic snapshots:

- `${NAVIDROME_DATA_PATH}` — SQLite DB (play counts, ratings, starred items, smart playlists, user accounts)

The music library itself (`${NAVIDROME_MUSIC_PATH}`) is bulk media on the NAS — handle its backup at the NAS level.

> The DB holds all user-generated state. Losing it means losing every play count, rating, starred item, and user across every client — but the music files themselves are intact and a fresh scan will rebuild the catalog.

---

## Notes & Gotchas

- **Read-only music mount.** Navidrome reads tags from files but never writes back. If your library was previously managed by Plex's database-driven metadata, you may notice gaps. Tools like [MusicBrainz Picard](https://picard.musicbrainz.org/) or [beets](https://beets.io/) can clean tags before Navidrome ingests them.
- **First scan is single-threaded and slow.** A library of tens of thousands of tracks may take an hour or more. Subsequent scans are incremental and fast (the default `@every 1m` re-check is essentially free when nothing has changed).
- **Smart playlists** are defined as `.nsp` JSON files dropped into the music folder (or a subfolder). See the [smart playlist docs](https://www.navidrome.org/docs/usage/smartplaylists/).
- **Container runs as `1000:1000`**, not root. The music share and `${NAVIDROME_DATA_PATH}` must be readable/writable by this UID:GID. Adjust the `user:` directive in compose if your library has different ownership.
- **Coexists with Jellyfin.** Both can index the same library safely since Navidrome's mount is read-only. Jellyfin's music handling is fine for casual use; Navidrome wins for power users (proper Subsonic API, smart playlists, better tag fidelity, large client ecosystem).
- **NFS mount must be up.** If your NAS mount (e.g. `/mnt/nas`) isn't mounted, the container starts but `/music` is empty inside the container. Verify with `mountpoint /mnt/nas` before deploying.
- **Tuning later.** If you ever hit transcoding bottlenecks or want to slow the scan cadence, the relevant knobs are `ND_TRANSCODINGCACHESIZE`, `ND_IMAGECACHESIZE`, and `ND_SCANSCHEDULE`. Defaults are fine for most homelab use.

---

## References

- [Navidrome — Docker install](https://www.navidrome.org/docs/installation/docker/)
- [Navidrome configuration reference](https://www.navidrome.org/docs/usage/configuration-options/)
- [Smart playlists](https://www.navidrome.org/docs/usage/smartplaylists/)
- [Top-level README](../../README.md)
