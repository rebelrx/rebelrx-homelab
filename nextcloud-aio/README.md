# nextcloud-aio

Self-hosted cloud storage and collaboration platform using Nextcloud's All-in-One deployment, which manages its own sub-containers for the full Nextcloud stack.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **Nextcloud AIO Master Container** | Orchestrator that manages all Nextcloud sub-containers | `ghcr.io/nextcloud-releases/all-in-one` |

## Network Architecture

| Network | Type | Purpose |
|---------|------|---------|
| `nextcloud-aio` | External | Shared network for AIO-managed containers and reverse proxy |

## Ports

| Service | Host Port | Container Port | Notes |
|---------|-----------|----------------|-------|
| AIO Management | `${AIO_MANAGEMENT_PORT}` | 8080 | AIO admin interface |

The AIO master container manages additional ports for the Nextcloud web interface via its own Apache container, configured through `APACHE_PORT` and `APACHE_IP_BINDING`.

## Environment Variables

Create a `.env` file in the same directory as the compose file.

| Variable | Description | Example |
|----------|-------------|---------|
| `AIO_MANAGEMENT_PORT` | AIO management UI host port | `8080` |
| `NEXTCLOUD_DATADIR` | Host path for Nextcloud user data | `/mnt/nextcloud/data` |
| `APACHE_PORT` | Port for the AIO-managed Apache container | `11000` |
| `APACHE_IP_BINDING` | IP binding for the Apache container | `0.0.0.0` |
| `SKIP_DOMAIN_VALIDATION` | Skip domain validation during setup | `true` |

## Volumes

| Container Path | Purpose |
|----------------|---------|
| `nextcloud_aio_mastercontainer` (named volume) | AIO configuration and state |
| `/var/run/docker.sock` | Docker socket for managing sub-containers (read-only) |
| `/mnt` | Host mount point for Nextcloud data access |

## Deployment

1. Create the external network if it doesn't exist:
   ```bash
   docker network create nextcloud-aio
   ```

2. Create your `.env` file with all required variables.

3. Deploy:
   ```bash
   docker compose up -d
   ```

4. Access the AIO management interface at `https://<host>:<AIO_MANAGEMENT_PORT>` and follow the setup wizard. The master container will pull and manage additional containers (database, Redis, Apache, etc.) automatically.

## Notes

- The AIO master container manages its own sub-containers — do not manually create or manage them.
- `NEXTCLOUD_ENABLE_NVIDIA_GPU=true` is set for GPU-accelerated features (e.g., Memories face recognition). Requires NVIDIA Container Toolkit on the host.
- The `nextcloud-aio` network must be shared with your reverse proxy (Nginx Proxy Manager) for proper routing.
- `SKIP_DOMAIN_VALIDATION` should only be set to `true` if you're handling domain/SSL at the reverse proxy level.
- Uses `init: true` to ensure proper signal handling and zombie process reaping.
