# arr

Media automation suite covering TV, movies, music, books, comics, and adult content — with VPN-routed download clients and indexers.

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `gluetun` | `qmcgaw/gluetun:latest` | WireGuard VPN gateway with kill-switch firewall |
| `qbittorrent` | `lscr.io/linuxserver/qbittorrent:latest` | Torrent client *(routed through Gluetun)* |
| `sabnzbd` | `lscr.io/linuxserver/sabnzbd:latest` | Usenet downloader *(routed through Gluetun)* |
| `prowlarr` | `lscr.io/linuxserver/prowlarr:latest` | Indexer manager *(routed through Gluetun)* |
| `radarr` | `lscr.io/linuxserver/radarr:latest` | Movie automation |
| `sonarr` | `lscr.io/linuxserver/sonarr:latest` | TV automation |
| `lidarr` | `lscr.io/linuxserver/lidarr:latest` | Music automation |
| `whisparr` | `ghcr.io/hotio/whisparr:latest` | Adult-content automation |
| `lazylibrarian` | `lscr.io/linuxserver/lazylibrarian:latest` | Book automation (with Calibre + ffmpeg mods) |
| `mylar3` | `lscr.io/linuxserver/mylar3:latest` | Comic automation |
| `bazarr` | `lscr.io/linuxserver/bazarr:latest` | Subtitle management for Sonarr/Radarr |
| `seerr` | `ghcr.io/seerr-team/seerr:latest` | Request management for Plex / Jellyfin (Overseerr fork — Overseerr is deprecated) |

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `media_net` | External | Inter-stack media access (shared with `plex`, `jellyfin`) |
| `proxy_net` | External | Reverse-proxy access via NPM |

> Gluetun joins both `media_net` and `proxy_net`. The three VPN-routed services (`qbittorrent`, `sabnzbd`, `prowlarr`) share Gluetun's network namespace via `network_mode: service:gluetun` and have no direct attachments. The remaining `*arr` apps, Bazarr, and Seerr attach to both `media_net` and `proxy_net` directly.

---

## Ports

All web UIs are published on **all interfaces**, so they're reachable on your LAN without a reverse proxy. They're *also* proxied via NPM over `proxy_net` (see [Reverse Proxy](#reverse-proxy-npm)).

| Service | Host binding | Container | Published on | Notes |
|---------|--------------|-----------|--------------|-------|
| qBittorrent | `0.0.0.0:${QBITTORRENT_WEBUI_PORT}` (`8090`) | `8090` | `gluetun` | Shared namespace |
| SABnzbd | `0.0.0.0:${SABNZBD_PORT}` (`8080`) | `8080` | `gluetun` | Shared namespace |
| Prowlarr | `0.0.0.0:${PROWLARR_PORT}` (`9696`) | `9696` | `gluetun` | Shared namespace |
| Radarr | `0.0.0.0:${RADARR_PORT}` (`7878`) | `7878` | `radarr` | |
| Sonarr | `0.0.0.0:${SONARR_PORT}` (`8989`) | `8989` | `sonarr` | |
| Lidarr | `0.0.0.0:${LIDARR_PORT}` (`8686`) | `8686` | `lidarr` | |
| Whisparr | `0.0.0.0:${WHISPARR_PORT}` (`6969`) | `6969` | `whisparr` | |
| LazyLibrarian | `0.0.0.0:${LAZYLIBRARIAN_PORT}` (`5299`) | `5299` | `lazylibrarian` | |
| Mylar3 | `0.0.0.0:${MYLAR_PORT}` (`8090`) | `8090` | `mylar3` | ⚠️ clashes with qBittorrent — change `MYLAR_PORT` |
| Bazarr | `0.0.0.0:${BAZARR_PORT}` (`6767`) | `6767` | `bazarr` | |
| Seerr | `0.0.0.0:${SEERR_PORT}` (`5055`) | `5055` | `seerr` | |

> **⚠️ Mylar3 / qBittorrent port clash.** Both default to `8090`, and qBittorrent's
> UI is published on the host via `gluetun`. You cannot bind both to `8090` — set
> `MYLAR_PORT` to a free host port (e.g. `8091`) in `.env` before bringing the
> stack up, or the `mylar3` container will fail to start.
>
> **Loopback only.** To keep a service behind the reverse proxy instead of
> exposing it on your LAN, prefix its mapping in `compose.yaml` with `127.0.0.1:`.
>
> **VPN-routed trio.** qBittorrent, SABnzbd, and Prowlarr publish their host
> ports on the `gluetun` service because they share its network namespace. For
> those ports to be reachable, `FIREWALL_INPUT_SUBNETS` must include the range
> the connection arrives from (your LAN and/or the Docker bridge, typically
> `172.x.0.0/16`).

---

## Volumes

### Configs (local NVMe)

| Host path (`.env` var) | Container path | Service |
|------------------------|----------------|---------|
| `${QBITTORRENT_CONFIG}` | `/config` | qBittorrent |
| `${SABNZBD_CONFIG}` | `/config` | SABnzbd |
| `${PROWLARR_CONFIG}` | `/config` | Prowlarr |
| `${RADARR_CONFIG}` | `/config` | Radarr |
| `${SONARR_CONFIG}` | `/config` | Sonarr |
| `${LIDARR_CONFIG}` | `/config` | Lidarr |
| `${WHISPARR_CONFIG}` | `/config` | Whisparr |
| `${LAZYLIBRARIAN_CONFIG}` | `/config` | LazyLibrarian |
| `${MYLAR_CONFIG}` | `/config` | Mylar3 |
| `${BAZARR_CONFIG}` | `/config` | Bazarr |
| `${SEERR_CONFIG}` | `/app/config` | Seerr |

### Media library (NAS NFS)

| Host path (`.env` var) | Container path | Used by |
|------------------------|----------------|---------|
| `${TV_PATH}` | `/tv` | Sonarr, Bazarr |
| `${MOVIES_PATH}` | `/movies` | Radarr, Bazarr |
| `${MUSIC_PATH}` | `/music` | Lidarr |
| `${BOOKS_PATH}` | `/books` | LazyLibrarian |
| `${COMICS_PATH}` | `/comics` | Mylar3 |
| `${XXX_PATH}` | `/xxx` | Whisparr |
| `${COMICS_STAGING}` | `/comics-staging` | Mylar3 |

### Downloads (NAS NFS — shared between clients and *arrs)

| Host path (`.env` var) | Container path | Used by |
|------------------------|----------------|---------|
| `${TORRENTS_PATH}` | `/torrents` | qBittorrent + every *arr (atomic moves) |
| `${USENET_COMPLETE_PATH}` | `/downloads` | SABnzbd + every *arr |
| `${USENET_INCOMPLETE_PATH}` | `/incomplete-downloads` | SABnzbd (in-flight Usenet downloads) |

> `TORRENTS_PATH` and `USENET_COMPLETE_PATH` are referenced directly by both the download clients and every *arr that consumes them, so hardlink / atomic-move behaviour is guaranteed by construction — you cannot accidentally desync them.

---

## Environment Variables

### `.env` (Compose interpolation)

#### General

| Variable | Example | Purpose |
|----------|---------|---------|
| `PUID` | `1000` | UID inside containers (linuxserver convention) |
| `PGID` | `1000` | GID inside containers |
| `TIMEZONE` | `America/New_York` | Container TZ |

#### Gluetun / VPN

| Variable | Example | Purpose |
|----------|---------|---------|
| `VPN_SERVICE_PROVIDER` | `mullvad` | Gluetun provider key — see [providers list](https://github.com/qdm12/gluetun-wiki/blob/main/setup/providers.md) |
| `VPN_TYPE` | `wireguard` | Tunnel protocol |
| `WIREGUARD_PRIVATE_KEY` | *(secret)* | WireGuard client private key |
| `WIREGUARD_ADDRESSES` | `10.x.x.x/32` | WireGuard interface address |
| `SERVER_CITIES` | `New York` | Comma-separated city allowlist |
| `DNS_KEEP_NAMESERVER` | `on` | Keep upstream DNS instead of Gluetun's built-in |
| `FIREWALL_INPUT_SUBNETS` | `192.168.1.0/24,100.64.0.0/10` | LAN + Tailscale subnets allowed inbound (replace with your own LAN range) |
| `FIREWALL_OUTBOUND_SUBNETS` | `192.168.1.0/24,100.64.0.0/10` | LAN + Tailscale subnets allowed outbound (so the *arr containers can reach Gluetun; replace with your own LAN range) |

#### Ports

All of these are **host ports published by compose** (on all interfaces). `QBITTORRENT_TORRENTING_PORT` is the exception — it sets qBittorrent's BitTorrent listen port inside the container and is not published on the host.

| Variable | Example | Purpose |
|----------|---------|---------|
| `QBITTORRENT_WEBUI_PORT` | `8090` | qBittorrent web UI (published via `gluetun`; `8090` avoids SABnzbd's `8080` inside the shared namespace) |
| `QBITTORRENT_TORRENTING_PORT` | `6881` | qBittorrent BitTorrent listen port (set inside the container; forwarded by your VPN provider, **not** published on the host) |
| `SABNZBD_PORT` | `8080` | SABnzbd web UI (published via `gluetun`) |
| `PROWLARR_PORT` | `9696` | Prowlarr web UI (published via `gluetun`) |
| `RADARR_PORT` | `7878` | Radarr web UI |
| `SONARR_PORT` | `8989` | Sonarr web UI |
| `LIDARR_PORT` | `8686` | Lidarr web UI |
| `WHISPARR_PORT` | `6969` | Whisparr web UI |
| `LAZYLIBRARIAN_PORT` | `5299` | LazyLibrarian web UI |
| `MYLAR_PORT` | `8090` | Mylar3 web UI — ⚠️ **change this**; clashes with qBittorrent's `8090` |
| `BAZARR_PORT` | `6767` | Bazarr web UI |
| `SEERR_PORT` | `5055` | Seerr web UI |

#### Volumes

See [Volumes](#volumes) above for the full list — `*_CONFIG` variables, `TV_PATH` / `MOVIES_PATH` / `MUSIC_PATH` / `BOOKS_PATH` / `COMICS_PATH` / `XXX_PATH` / `COMICS_STAGING`, and the shared download paths.

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme | Notes |
|----------|---------------|---------------|--------|-------|
| `qbittorrent.example.com` | `gluetun` | `${QBITTORRENT_WEBUI_PORT}` (`8090`) | `http` | Web UI binds inside Gluetun's namespace |
| `sabnzbd.example.com` | `gluetun` | `8080` | `http` | Web UI binds inside Gluetun's namespace |
| `prowlarr.example.com` | `gluetun` | `9696` | `http` | Web UI binds inside Gluetun's namespace |
| `radarr.example.com` | `radarr` | `7878` | `http` | |
| `sonarr.example.com` | `sonarr` | `8989` | `http` | |
| `lidarr.example.com` | `lidarr` | `8686` | `http` | |
| `whisparr.example.com` | `whisparr` | `6969` | `http` | |
| `lazylibrarian.example.com` | `lazylibrarian` | `5299` | `http` | |
| `mylar3.example.com` | `mylar3` | `8090` | `http` | |
| `bazarr.example.com` | `bazarr` | `6767` | `http` | |
| `seerr.example.com` | `seerr` | `5055` | `http` | |

> NPM must be on `proxy_net` (it is — see the `npm` stack). For the VPN-routed three, NPM resolves `gluetun` over `proxy_net` and reaches their UIs through Gluetun's namespace.

---

## Inter-Service Wiring

Inside the stack, the *arr apps and Bazarr need to reach the download clients and Prowlarr. Because qBittorrent, SABnzbd, and Prowlarr live inside Gluetun's network namespace, they're addressed via Gluetun:

| From (any *arr / Bazarr / Seerr) | To | URL |
|----------------------------------|-----|-----|
| Radarr / Sonarr / Lidarr / Whisparr / LazyLibrarian | qBittorrent | `http://gluetun:8090` |
| Radarr / Sonarr / Lidarr / Whisparr / LazyLibrarian | SABnzbd | `http://gluetun:8080` |
| Radarr / Sonarr / Lidarr / Whisparr / LazyLibrarian | Prowlarr | `http://gluetun:9696` |
| Bazarr | Sonarr | `http://sonarr:8989` |
| Bazarr | Radarr | `http://radarr:7878` |
| Seerr | Sonarr | `http://sonarr:8989` |
| Seerr | Radarr | `http://radarr:7878` |

> The VPN-routed trio's UIs are bound inside Gluetun's namespace, so `gluetun:<port>` is correct over `media_net`. This is the canonical Gluetun + arr pattern.

---

## Deployment

### Prerequisites

- `media_net` and `proxy_net` external networks must exist (`docker network create media_net proxy_net`)
- Your NAS NFS share mounted on the host (e.g. at `/mnt/nas`; see the top-level `system/<hostname>/fstab.example`)
- WireGuard credentials from your VPN provider

### Bring up

```bash
cd /opt/stacks/arr
cp .env.example .env
nano .env

docker compose up -d
```

### First-run setup

The *arr apps and download clients need to be wired together manually after first boot. The recommended order:

1. **Verify Gluetun first.** `docker compose logs gluetun` should show a successful WireGuard handshake. Until Gluetun is healthy, qBittorrent / SABnzbd / Prowlarr won't start (they have `depends_on: condition: service_healthy`).
2. **Verify VPN egress.** `docker compose exec gluetun wget -qO- https://ipinfo.io/ip` should return your VPN provider's IP, not your WAN IP.
3. **qBittorrent default password.** Check `docker compose logs qbittorrent` for the temporary admin password (linuxserver image generates one on first boot). Log in at `https://qbittorrent.example.com`, set a permanent password, and configure the default save path to `/torrents`.
4. **SABnzbd setup wizard.** Visit `https://sabnzbd.example.com`. Set the temp folder to `/incomplete-downloads` and the completed folder to `/downloads`.
5. **Prowlarr.** Visit `https://prowlarr.example.com`, complete auth setup, add indexers, then add each *arr as an "App" using internal URLs (`http://radarr:7878` etc.) — Prowlarr will sync indexers to all of them automatically.
6. **Each *arr.** Add download clients using the URLs in [Inter-Service Wiring](#inter-service-wiring). Set root folders to the appropriate library paths (`/movies`, `/tv`, etc.). Configure remote path mapping if needed (it generally isn't, since the *arrs and download clients see the same `/torrents` and `/downloads` paths).
7. **Bazarr.** Connect to Sonarr and Radarr, configure subtitle providers.
8. **Seerr.** Connect to Plex / Jellyfin, then to Sonarr and Radarr.

---

## Backup

Critical paths to include in Kopia / Borgmatic snapshots:

- `/opt/data/{qbittorrent,sabnzbd,prowlarr,radarr,sonarr,lidarr,whisparr,lazylibrarian,mylar3,bazarr,seerr}/config` — every app's SQLite DB and settings

The NAS media library and download dirs are bulk media — handle their backup at the NAS level (NAS snapshots / replication), not from this host.

> Each *arr stores its DB in `<config>/<app>.db` (SQLite). Hot snapshots are generally fine because *arrs are mostly read; for safety on Sonarr/Radarr specifically, take a full backup via the app's built-in **System → Backup** before any major version upgrade.

---

## Notes & Gotchas

- **Kill switch is implicit.** Gluetun's firewall blocks all traffic that doesn't match `FIREWALL_*_SUBNETS` while the tunnel is down. The three VPN-routed services share Gluetun's namespace, so they go offline cleanly when the tunnel drops — no leaks.
- **Gluetun health = trio availability.** If Gluetun's healthcheck (`ping google.com`) fails, qBittorrent/SAB/Prowlarr stay running but lose all egress. The *arr apps will report download client failures until Gluetun recovers.
- **qBittorrent on `8090`, not `8080`.** This is intentional — SABnzbd's default is `8080`, and both share Gluetun's namespace, so they'd collide. Don't change `QBITTORRENT_WEBUI_PORT` back to `8080` unless you also move SABnzbd.
- **WUD watch is `false` on Gluetun.** VPN container updates can break the tunnel and require config changes; manual updates only.
- **`FIREWALL_OUTBOUND_SUBNETS` matters.** If the *arr containers can't reach `gluetun:8080` etc., this is usually the cause — the value must include the Docker network ranges that `media_net` uses (typically `172.x.0.0/16`).
- **Atomic moves require shared paths.** Both the download clients and the *arrs reference `TORRENTS_PATH` / `USENET_COMPLETE_PATH` directly, so they always see the same physical paths and can hardlink instead of copying. Don't override these per-service — drift would silently double your disk usage.
- **NFS mount must be up.** If your NAS mount (e.g. `/mnt/nas`) isn't mounted, the *arrs will start but every library and download path will be empty inside the containers. Verify with `mountpoint /mnt/nas` before bringing up the stack.
- **Seerr vs Overseerr.** This stack migrated to Seerr because Overseerr is deprecated upstream. Settings layout, API endpoints, and Plex/*arr integrations are largely compatible, but the container config path differs: Seerr uses `/app/config` (Overseerr used `/config`).

---

## References

- [Gluetun wiki](https://github.com/qdm12/gluetun-wiki)
- [TRaSH Guides — recommended *arr setup](https://trash-guides.info/)
- [LinuxServer.io image docs](https://docs.linuxserver.io/)
- [Hotio Whisparr image](https://hotio.dev/containers/whisparr/)
- [Seerr documentation](https://docs.seerr.dev/)
- [Top-level README](../../README.md)
