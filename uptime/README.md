# uptime

Service uptime monitoring with Uptime Kuma, providing status pages, notifications, and Docker container monitoring.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **Uptime Kuma** | Self-hosted uptime monitoring with status pages | `louislam/uptime-kuma` |

## Network Architecture

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse proxy access |

## Ports

| Service | Host Port | Container Port | Notes |
|---------|-----------|----------------|-------|
| Uptime Kuma | `${UPTIME_PORT}` | 3001 | Web UI (localhost only) |

## Environment Variables

Create a `.env` file in the same directory as the compose file.

| Variable | Description | Example |
|----------|-------------|---------|
| `UPTIME_PORT` | Uptime Kuma web UI host port | `3001` |
| `TIMEZONE` | Timezone | `America/New_York` |
| `UPTIME_DATA` | Path to Uptime Kuma data directory | `/opt/uptime-kuma/data` |

## Volumes

| Container Path | Purpose | Access |
|----------------|---------|--------|
| `/app/data` | Application data and SQLite database | Read-write |
| `/var/run/docker.sock` | Docker socket for container monitoring | Read-write |

## Deployment

1. Create the external network if it doesn't exist:
   ```bash
   docker network create proxy_net
   ```

2. Create your `.env` file with all required variables.

3. Deploy:
   ```bash
   docker compose up -d
   ```

4. Access Uptime Kuma at the reverse proxy URL and create your admin account on first launch.

## Notes

- Port is bound to `127.0.0.1` for reverse proxy access only.
- The Docker socket is mounted for container status monitoring. Consider mounting as `:ro` if you don't need container control.
- `UMASK=0022` is set for standard file permissions.
- Uses SQLite internally — no external database required.
