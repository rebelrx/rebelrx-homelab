# speedtest

Automated internet speed testing and historical tracking using Speedtest Tracker and Librespeed.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **Speedtest Tracker** | Scheduled speed tests with historical dashboard | `lscr.io/linuxserver/speedtest-tracker` |
| **Librespeed** | Self-hosted HTML5 speed test | `lscr.io/linuxserver/librespeed` |

## Network Architecture

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse proxy access |

## Ports

| Service | Host Port | Container Port | Notes |
|---------|-----------|----------------|-------|
| Speedtest HTTP | `${SPEEDTEST_PORT_HTTP}` | 80 | Web UI HTTP (localhost only) |
| Speedtest HTTPS | `${SPEEDTEST_PORT_HTTPS}` | 443 | Web UI HTTPS (localhost only) |
| Librespeed | `${LIBRESPEED_PORT}` | 80 | Web UI (localhost only) |

## Environment Variables

Create a `.env` file in the same directory as the compose file.

| Variable | Description | Example |
|----------|-------------|---------|
| `PUID` | User ID for file permissions | `1000` |
| `PGID` | Group ID for file permissions | `1000` |
| `TIMEZONE` | Timezone | `America/New_York` |
| `SPEEDTEST_PORT_HTTP` | HTTP host port | `8080` |
| `SPEEDTEST_PORT_HTTPS` | HTTPS host port | `8443` |
| `APP_KEY` | Laravel application key | `base64:...` |
| `SPEEDTEST_SCHEDULE` | Cron schedule for speed tests | `0 */6 * * *` |
| `SPEEDTEST_SERVERS` | Preferred Ookla server IDs | `12345,67890` |
| `DISPLAY_TIMEZONE` | Timezone for UI display | `America/New_York` |
| `SPEEDTEST_CONFIG` | Path to config directory | `/opt/speedtest/config` |
| `LIBRESPEED_PORT` | Librespeed host port | `8090` |
| `LIBRESPEED_CONFIG` | Path to Librespeed config directory | `/opt/librespeed/config` |

## Deployment

1. Create the external network if it doesn't exist:
   ```bash
   docker network create proxy_net
   ```

2. Generate an application key:
   ```bash
   echo "base64:$(openssl rand -base64 32)"
   ```

3. Create your `.env` file with all required variables.

4. Deploy:
   ```bash
   docker compose up -d
   ```

## Notes

- Ports are bound to `127.0.0.1` for reverse proxy access only.
- Uses SQLite (`DB_CONNECTION=sqlite`) — no external database required.
- Results older than 7 days are automatically pruned (`PRUNE_RESULTS_OLDER_THAN=7`).
- Librespeed provides a self-hosted speed test page for testing from within your network without relying on external Ookla servers.
