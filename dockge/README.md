# dockge

Docker Compose stack management UI for deploying and managing stacks from a web interface.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **Dockge** | Web-based Docker Compose stack manager | `louislam/dockge` |

## Network Architecture

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse proxy access |

## Ports

| Service | Host Port | Container Port | Notes |
|---------|-----------|----------------|-------|
| Dockge | `${DOCKGE_PORT}` | 5001 | Web UI |

## Environment Variables

Create a `.env` file in the same directory as the compose file.

| Variable | Description | Example |
|----------|-------------|---------|
| `DOCKGE_PORT` | Dockge web UI host port | `5001` |
| `TIMEZONE` | Timezone | `America/New_York` |
| `DOCKGE_DATA` | Path to Dockge application data | `/opt/dockge/data` |
| `DOCKGE_COMPOSE` | Path to Compose stacks directory | `/opt/stacks` |

## Volumes

| Container Path | Purpose |
|----------------|---------|
| `/var/run/docker.sock` | Docker socket for container management |
| `/app/data` | Dockge application data |
| `/opt/stacks` | Directory where Dockge stores and reads Compose stacks |

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

4. Access the Dockge UI at `http://<host>:<DOCKGE_PORT>`.

## Notes

- Dockge requires access to the Docker socket to manage containers.
- The `DOCKGE_STACKS_DIR` environment variable is hardcoded to `/opt/stacks` inside the container. The `DOCKGE_COMPOSE` host path is mounted to this location.
- Image is pinned to `:1` (major version tag) rather than `:latest`.
