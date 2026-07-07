# jellyfin

Self-hosted media server with [Jellyfin](https://jellyfin.org/) — free and open-source alternative to Plex/Emby. NVIDIA GPU-accelerated transcoding via the host's NVIDIA GPU.

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `jellyfin` | `ghcr.io/linuxserver/jellyfin:latest` | Media server, web UI, transcoding engine |

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse-proxy access via NPM |
| `media_net` | External | Shared with `arr` and `plex` — *arr apps reach Jellyfin here for library refresh hooks |

---

## Ports

| Binding | Container port | Reachable from | Purpose |
|---------|----------------|----------------|---------|
| `${JELLYFIN_HTTP_PORT}:8096` | `8096` | All host interfaces (0.0.0.0) | Direct HTTP access for LAN/TV apps that don't go through NPM |

Jellyfin also listens internally on `8920` (HTTPS, unused since NPM terminates TLS). It is not published.

> The host port mapping is bound to **all interfaces** — Jellyfin is reachable on `http://<host-ip>:${JELLYFIN_HTTP_PORT}` from the LAN as well as via NPM. This differs from the reverse-proxied stacks' `127.0.0.1`-only pattern; the open binding is intentional to support TV/console apps that can't be pointed at a reverse-proxied HTTPS endpoint.
>
> DLNA, SSDP discovery, and AirPlay are still not exposed (they'd require `network_mode: host` or specific UDP ports — neither is configured here). The `plex` stack handles those for the local network.

---

## Volumes

| Host path (`.env` var) | Container path | Purpose |
|------------------------|----------------|---------|
| `${JELLYFIN_CONFIG}` | `/config` | Jellyfin config, SQLite DB, users, plugins, metadata cache |
| `${JELLYFIN_TRANSCODE}` | `/transcode` | Transcode scratch space — **must be local NVMe**, never NFS |
| `${JELLYFIN_MOVIES}` | `/movies` | Movies library |
| `${JELLYFIN_TV}` | `/tv` | TV library |
| `${JELLYFIN_MUSIC}` | `/music` | Music library |

> Each media library is mounted explicitly at its own container path. To add another library (audiobooks, comics, etc.), add a new explicit mount and a matching `.env` variable rather than reaching into a catch-all root — keeps Jellyfin's library boundaries cleanly aligned with what you actually want indexed.

---

## Environment Variables

### `.env` (Compose interpolation)

| Variable | Example | Purpose |
|----------|---------|---------|
| `PUID` | `1000` | UID inside container (linuxserver convention) |
| `PGID` | `1000` | GID inside container |
| `TIMEZONE` | `America/New_York` | Container TZ |
| `JELLYFIN_HTTP_PORT` | `8096` | Host port published to all interfaces, mapping to container port `8096` |
| `JELLYFIN_PUBLISHED_SERVER_URL` | `https://jellyfin.example.com` | Public URL Jellyfin advertises to clients (needed for proper deep linking and remote-streaming session reporting) |
| `JELLYFIN_CONFIG` | `/path/to/jellyfin/config` | Host bind for config |
| `JELLYFIN_TRANSCODE` | `/path/to/jellyfin/transcode` | Host bind for transcode scratch (local NVMe) |
| `JELLYFIN_MOVIES` | `/path/to/nas-media/movies` | Movies library |
| `JELLYFIN_TV` | `/path/to/nas-media/tv` | TV library |
| `JELLYFIN_MUSIC` | `/path/to/nas-media/music` | Music library |

### Hardcoded in `compose.yaml`

| Variable | Value | Purpose |
|----------|-------|---------|
| `NVIDIA_VISIBLE_DEVICES` | `all` | Expose all visible GPUs to the container |
| `NVIDIA_DRIVER_CAPABILITIES` | `all` | Enable all driver capability classes (broader than the minimum `video,utility` Jellyfin actually needs — cosmetic) |

---

## GPU Acceleration

Jellyfin uses NVENC/NVDEC for hardware transcoding. Two mechanisms are declared:

1. **`runtime: nvidia`** (legacy nvidia-docker v1 approach)
2. **`deploy.resources.reservations.devices` block** (modern NVIDIA Container Toolkit approach)

With both present, the modern `deploy:` block is what actually grants GPU access. The legacy `runtime: nvidia` line is redundant but harmless.

**Prerequisites:**
- NVIDIA proprietary driver + DKMS on the host
- NVIDIA Container Toolkit installed; Docker daemon configured with the `nvidia` runtime
- If documenting host-level NVIDIA setup in this repo, `system/<hostname>/README.md` is a sensible place for it

**Enable in Jellyfin:**

1. Admin Dashboard → **Playback** → Hardware acceleration: **NVENC**
2. Enable codec-specific options (H.264, HEVC, AV1, etc.) as your library demands
3. Set **Hardware decoding** to NVDEC for the codecs you have content in
4. Optional: enable tone mapping if you stream HDR to SDR clients

**Verify:**

```bash
# Start a transcode in Jellyfin (force one with a remux/format change)
nvidia-smi
# Look for `jellyfin` (or `ffmpeg` under the jellyfin container) in the process list
```

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme |
|----------|---------------|---------------|--------|
| `jellyfin.example.com` | `jellyfin` | `8096` | `http` |

> **Enable WebSocket support in NPM** — Jellyfin uses WebSockets for live session updates and SyncPlay. Without it, playback works but admin features feel laggy.
>
> **Bump `client_max_body_size` for poster uploads** — default 1 MB is fine for casual use but rejects large custom artwork. `100M` is reasonable.
>
> The `JELLYFIN_PUBLISHED_SERVER_URL` env var must match the public NPM URL — Jellyfin uses it to build absolute links sent to mobile/TV apps.

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing)
- `media_net` external network (`docker network create media_net` if missing)
- Your NAS NFS share mounted on the host (e.g. `/mnt/nas`; see the top-level `system/<hostname>/fstab.example`)
- NVIDIA Container Toolkit installed and Docker daemon configured with `nvidia` runtime

### Bring up

```bash
cd /opt/stacks/jellyfin
cp .env.example .env
nano .env  # set JELLYFIN_PUBLISHED_SERVER_URL to public URL, confirm paths

# Pre-create local dirs (match your ${JELLYFIN_CONFIG} / ${JELLYFIN_TRANSCODE})
sudo mkdir -p /path/to/jellyfin/config /path/to/jellyfin/transcode

docker compose up -d
```

### First-run setup

1. Open the app (e.g. `https://jellyfin.example.com` via NPM, or `http://<host-ip>:8096` directly) — the setup wizard appears.
2. Create the **admin user**.
3. Add libraries:
   - **Movies** → `/movies` (folder type: Movies)
   - **TV Shows** → `/tv` (folder type: Shows)
   - **Music** → `/music` (folder type: Music)
   - To add other libraries (audiobooks, comics, etc.), add a new explicit mount in `compose.yaml` first.
4. **Enable NVENC** under Dashboard → Playback (see [GPU Acceleration](#gpu-acceleration) above).
5. Verify a transcode triggers the GPU with `nvidia-smi`.

---

## Backup

Paths to include in Kopia / Borgmatic snapshots:

- `${JELLYFIN_CONFIG}` — users, library DB, watch history, plugin config, metadata cache

Skip from snapshots:
- `${JELLYFIN_TRANSCODE}` — volatile scratch, regenerated as needed
- The media paths themselves — bulk media on the NAS, backed up at the NAS level

> Jellyfin's metadata cache inside `${JELLYFIN_CONFIG}/data/metadata` can grow to several GB on large libraries. It's rebuildable but expensive (re-downloads from TMDB/MusicBrainz). Keep it in your snapshots.

---

## Notes & Gotchas

- **No `no-new-privileges`.** Intentionally omitted because the NVIDIA Container Toolkit can interact poorly with the flag on some driver versions. The `plex` stack has the same exception for a different reason (host networking).
- **Host port published on all interfaces.** `${JELLYFIN_HTTP_PORT}:8096` deliberately binds to `0.0.0.0` so TV/console apps that don't accept self-signed certs or non-standard ports can reach Jellyfin directly over HTTP on the LAN. Unlike the reverse-proxied stacks, this is not a `127.0.0.1`-only or Tailscale-IP binding. To lock it down to one interface, change the mapping to `<ip>:${JELLYFIN_HTTP_PORT}:8096`.
- **`runtime: nvidia` is legacy.** The modern way is the `deploy.resources.reservations.devices` block, which is also present. The two together are harmless but redundant; you can remove `runtime: nvidia` next time you touch the compose.
- **Transcode dir on local NVMe — non-negotiable.** Jellyfin writes intermediate transcode segments at high frequency; NFS will introduce latency that produces playback stutter. Keep `${JELLYFIN_TRANSCODE}` on local disk, never a NAS share.
- **No DLNA/SSDP exposed.** Mobile and web apps work fine over NPM; TV apps that rely on local-network DLNA discovery won't find Jellyfin. If you need it, that's a separate compose change (host networking or a DLNA proxy).
- **`JELLYFIN_PUBLISHED_SERVER_URL` matters for mobile apps.** If it doesn't match the actual reachable URL, the Jellyfin mobile app builds broken URLs when reporting playback state back to the server.

---

## References

- [LinuxServer.io — docker-jellyfin](https://docs.linuxserver.io/images/docker-jellyfin/)
- [Jellyfin docs — hardware acceleration](https://jellyfin.org/docs/general/administration/hardware-acceleration/)
- [Jellyfin docs — NVIDIA setup](https://jellyfin.org/docs/general/administration/hardware-acceleration/nvidia/)
- [Top-level README](../../README.md)
