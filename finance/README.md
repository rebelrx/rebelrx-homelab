# finance

Personal budgeting with Actual Budget, a privacy-focused finance tool with local-first data storage.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **Actual Budget** | Self-hosted personal finance and budgeting app | `docker.io/actualbudget/actual-server` |

## Network Architecture

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse proxy access |

## Ports

| Service | Host Port | Container Port | Notes |
|---------|-----------|----------------|-------|
| Actual Budget | `${PORT}` | 5006 | Web UI (localhost only) |

## Environment Variables

Create a `.env` file in the same directory as the compose file.

| Variable | Description | Example |
|----------|-------------|---------|
| `PORT` | Actual Budget web UI host port | `5006` |
| `ACTUAL_DATA_PATH` | Path to Actual Budget data directory | `/opt/actual/data` |

## Volumes

| Container Path | Purpose |
|----------------|---------|
| `/data` | Application data, database, and uploaded files |

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

4. Access the web UI and create your first budget. See [Actual Budget configuration docs](https://actualbudget.org/docs/config/) for additional environment variables (HTTPS, upload limits, etc.).

## Notes

- Port is bound to `127.0.0.1` for reverse proxy access only.
- Actual Budget uses SQLite by default — no external database required.
