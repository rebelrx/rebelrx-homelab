# media

Media server stack with Plex and Jellyfin, both configured with NVIDIA GPU hardware acceleration for transcoding.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **Plex** | Media server with client apps and streaming | `lscr.io/linuxserver/plex` |
| **Jellyfin** | Open-source media server | `lscr.io/linuxserver/jellyfin` |

## Network Architecture

Plex uses `network_mode: host` for direct network access (required for DLNA and client discovery). Jellyfin connects to the external `media_net` network.

| Network | Type | Purpose |
|---------|------|---------|
| `media_net` | External | Shared media network (Jellyfin) |

## Ports

| Service | Host Port | Container Port | Notes |
|---------|-----------|----------------|-------|
| Plex | Host networking | 32400 | All ports exposed via host mode |
| Jellyfin | `${JELLYFIN_PORT}` | 8096 | Web UI |

## Environment Variables

Create a `.env` file in the same directory as the compose file.

### Global

| Variable | Description | Example |
|----------|-------------|---------|
| `PUID` | User ID for file permissions | `1000` |
| `PGID` | Group ID for file permissions | `1000` |
| `TIMEZONE` | Timezone | `America/New_York` |

### Plex

| Variable | Description | Example |
|----------|-------------|---------|
| `PLEX_CLAIM` | Plex claim token (from https://plex.tv/claim) | `claim-xxxx` |
| `PLEX_CONFIG` | Path to Plex config directory | `/opt/plex/config` |
| `PLEX_MEDIA` | Path to general media directory | `/mnt/media` |
| `PLEX_MOVIES` | Path to movies directory | `/mnt/movies` |
| `PLEX_TV` | Path to TV shows directory | `/mnt/tv` |
| `PLEX_MUSIC` | Path to music directory | `/mnt/music` |
| `PLEX_TRANSCODE` | Path to transcode temp directory | `/opt/plex/transcode` |

### Jellyfin

| Variable | Description | Example |
|----------|-------------|---------|
| `JELLYFIN_PORT` | Jellyfin web UI host port | `8096` |
| `JELLYFIN_PUBLISHED_SERVER_URL` | Public URL for Jellyfin | `https://jellyfin.example.com` |
| `JELLYFIN_CONFIG` | Path to Jellyfin config directory | `/opt/jellyfin/config` |
| `JELLYFIN_TRANSCODE` | Path to transcode temp directory | `/opt/jellyfin/transcode` |
| `JELLYFIN_MEDIA` | Path to general media directory | `/mnt/media` |
| `JELLYFIN_MOVIES` | Path to movies directory | `/mnt/movies` |
| `JELLYFIN_TV` | Path to TV shows directory | `/mnt/tv` |
| `JELLYFIN_MUSIC` | Path to music directory | `/mnt/music` |

## GPU Acceleration

Both services are configured for NVIDIA GPU transcoding:

- `runtime: nvidia` is set on both containers
- `NVIDIA_VISIBLE_DEVICES=all` and `NVIDIA_DRIVER_CAPABILITIES=all` are set in the environment
- GPU resources are reserved via `deploy.resources.reservations.devices` (1 NVIDIA GPU)

### Prerequisites

- NVIDIA GPU with drivers installed on the host
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html) installed
- Docker configured to use the `nvidia` runtime

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

4. Access Plex at `http://<host>:32400/web` and Jellyfin at `http://<host>:<JELLYFIN_PORT>`.

## Notes

- Plex uses host networking, so all Plex ports are exposed directly on the host. Ensure no port conflicts.
- The `PLEX_CLAIM` token is only needed for initial setup and expires after 4 minutes.
- Transcode directories should ideally be on fast storage (SSD) for best performance.
