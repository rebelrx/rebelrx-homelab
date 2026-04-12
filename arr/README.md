# arr

Media automation stack for managing TV, movies, music, books, comics, and subtitles. All download clients (qBittorrent, SABnzbd) and indexers (Prowlarr) are routed through a Gluetun VPN container.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **Gluetun** | WireGuard VPN container — all download traffic is tunneled through this | `qmcgaw/gluetun` |
| **qBittorrent** | Torrent download client (routed through Gluetun) | `lscr.io/linuxserver/qbittorrent` |
| **SABnzbd** | Usenet download client (routed through Gluetun) | `lscr.io/linuxserver/sabnzbd` |
| **Prowlarr** | Indexer manager for Sonarr/Radarr/Lidarr (routed through Gluetun) | `lscr.io/linuxserver/prowlarr` |
| **Sonarr** | TV show management and automation | `lscr.io/linuxserver/sonarr` |
| **Radarr** | Movie management and automation | `lscr.io/linuxserver/radarr` |
| **Lidarr** | Music management and automation | `lscr.io/linuxserver/lidarr` |
| **LazyLibrarian** | Book management with Calibre and FFmpeg mods | `lscr.io/linuxserver/lazylibrarian` |
| **Mylar3** | Comic book management | `lscr.io/linuxserver/mylar3` |
| **Bazarr** | Subtitle management for Sonarr and Radarr | `lscr.io/linuxserver/bazarr` |
| **Overseerr** | Media request management for Plex users | `lscr.io/linuxserver/overseerr` |

## Network Architecture

All services connect to the external `media_net` network. Download clients and Prowlarr use `network_mode: "service:gluetun"` to route all traffic through the VPN. Their ports are published through the Gluetun container.

| Network | Type | Purpose |
|---------|------|---------|
| `media_net` | External | Shared network for all media services |

## Ports

| Service | Host Port | Container Port | Notes |
|---------|-----------|----------------|-------|
| Prowlarr | `${PROWLARR_PORT}` | 9696 | Published via Gluetun |
| qBittorrent WebUI | `${QBITTORRENT_WEBUI_PORT}` | 8090 | Published via Gluetun |
| qBittorrent Torrent | `${QBITTORRENT_TORRENT_PORT}` | 6881 | Published via Gluetun (TCP) |
| qBittorrent UDP | `${QBITTORRENT_UDP_PORT}` | 6881/udp | Published via Gluetun (UDP) |
| SABnzbd | `${SABNZBD_PORT}` | 8080 | Published via Gluetun |
| Sonarr | `${SONARR_PORT}` | 8989 | |
| Radarr | `${RADARR_PORT}` | 7878 | |
| Lidarr | `${LIDARR_PORT}` | 8686 | |
| LazyLibrarian | `${LAZYLIBRARIAN_PORT}` | 5299 | |
| Mylar3 | `${MYLAR_PORT}` | 8090 | |
| Bazarr | `${BAZARR_PORT}` | 6767 | |
| Overseerr | `${OVERSEERR_PORT}` | 5055 | |

## Environment Variables

Create a `.env` file in the same directory as the compose file.

### Global

| Variable | Description | Example |
|----------|-------------|---------|
| `PUID` | User ID for file permissions | `1000` |
| `PGID` | Group ID for file permissions | `1000` |
| `TIMEZONE` | Timezone | `America/New_York` |

### Gluetun (VPN)

| Variable | Description | Example |
|----------|-------------|---------|
| `VPN_SERVICE_PROVIDER` | VPN provider name | `mullvad` |
| `VPN_TYPE` | VPN protocol | `wireguard` |
| `WIREGUARD_PRIVATE_KEY` | WireGuard private key | *(secret)* |
| `WIREGUARD_ADDRESSES` | WireGuard interface addresses | `10.x.x.x/32` |
| `SERVER_CITIES` | Preferred VPN server cities | `New York NY` |
| `DNS_KEEP_NAMESERVER` | Keep existing DNS nameserver | `on` |
| `FIREWALL_INPUT_SUBNETS` | Allowed inbound subnets | `192.168.1.0/24` |
| `FIREWALL_OUTBOUND_SUBNETS` | Allowed outbound subnets | `192.168.1.0/24` |

### Service Ports

| Variable | Description |
|----------|-------------|
| `PROWLARR_PORT` | Prowlarr host port |
| `QBITTORRENT_WEBUI_PORT` | qBittorrent WebUI host port |
| `QBITTORRENT_TORRENT_PORT` | qBittorrent torrent host port |
| `QBITTORRENT_UDP_PORT` | qBittorrent UDP host port |
| `SABNZBD_PORT` | SABnzbd host port |
| `SONARR_PORT` | Sonarr host port |
| `RADARR_PORT` | Radarr host port |
| `LIDARR_PORT` | Lidarr host port |
| `LAZYLIBRARIAN_PORT` | LazyLibrarian host port |
| `MYLAR_PORT` | Mylar3 host port |
| `BAZARR_PORT` | Bazarr host port |
| `OVERSEERR_PORT` | Overseerr host port |

### Volume Paths

Each service uses `${SERVICE_CONFIG}` for its config directory and additional paths for media/downloads. See the compose file for the full list of volume variables per service.

## Deployment

1. Create the external network if it doesn't exist:
   ```bash
   docker network create media_net
   ```

2. Create your `.env` file with all required variables.

3. Deploy:
   ```bash
   docker compose up -d
   ```

4. Gluetun will establish the VPN connection first. qBittorrent, SABnzbd, and Prowlarr will wait for Gluetun's healthcheck to pass before starting.

## Dependencies

```
gluetun (VPN) ──► qbittorrent
               ──► sabnzbd
               ──► prowlarr
```

All other services (Sonarr, Radarr, etc.) start independently and communicate with the download clients over `media_net`.
