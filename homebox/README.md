# homebox

Home inventory management for tracking household items, warranties, and assets.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **Homebox** | Self-hosted home inventory management system | `ghcr.io/sysadminsmedia/homebox` |

## Network Architecture

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse proxy access |

## Ports

| Service | Host Port | Container Port | Notes |
|---------|-----------|----------------|-------|
| Homebox | `${HOMEBOX_PORT}` | 7745 | Web UI (localhost only) |

## Environment Variables

Create a `.env` file in the same directory as the compose file.

| Variable | Description | Example |
|----------|-------------|---------|
| `HOMEBOX_PORT` | Homebox web UI host port | `7745` |
| `HOMEBOX_DATA` | Path to Homebox data directory | `/opt/homebox/data` |

### Hardcoded Environment

The following are set directly in the compose file:

| Variable | Value | Description |
|----------|-------|-------------|
| `HBOX_LOG_LEVEL` | `info` | Logging level |
| `HBOX_LOG_FORMAT` | `text` | Log output format |
| `HBOX_WEB_MAX_UPLOAD_SIZE` | `10` | Max upload size in MB |
| `HBOX_OPTIONS_ALLOW_ANALYTICS` | `false` | Disable analytics |

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

4. Access Homebox at the reverse proxy URL and create your first account.

## Notes

- Port is bound to `127.0.0.1` for reverse proxy access only.
- Includes a healthcheck that polls the web UI endpoint.
