# homepage

Dashboard for organizing and accessing all self-hosted services from a single page.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **Homepage** | Modern, customizable application dashboard | `ghcr.io/gethomepage/homepage` |

## Network Architecture

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse proxy access |

## Ports

| Service | Host Port | Container Port | Notes |
|---------|-----------|----------------|-------|
| Homepage | `${HOMEPAGE_PORT}` | 3000 | Web UI (localhost only) |

## Environment Variables

Create a `.env` file in the same directory as the compose file.

| Variable | Description | Example |
|----------|-------------|---------|
| `HOMEPAGE_PORT` | Homepage web UI host port | `3000` |
| `TIMEZONE` | Timezone | `America/New_York` |
| `HOMEPAGE_ALLOWED_HOSTS` | Allowed hostnames for access through reverse proxy | `homepage.example.com` |
| `HOMEPAGE_CONFIG` | Path to Homepage config directory | `/opt/homepage/config` |

## Volumes

| Container Path | Purpose | Access |
|----------------|---------|--------|
| `/app/config` | Dashboard configuration files (services, widgets, etc.) | Read-write |
| `/var/run/docker.sock` | Docker socket for container status widgets | **Read-only** |

## Deployment

1. Create the external network if it doesn't exist:
   ```bash
   docker network create proxy_net
   ```

2. Create your `.env` file with all required variables.

3. Create your Homepage configuration files in the `HOMEPAGE_CONFIG` directory. At minimum you'll need `services.yaml` and `settings.yaml`. See [Homepage docs](https://gethomepage.dev/installation/docker/).

4. Deploy:
   ```bash
   docker compose up -d
   ```

## Notes

- Port is bound to `127.0.0.1` for reverse proxy access only.
- `HOMEPAGE_ALLOWED_HOSTS` must be set to your domain/hostname for access through Nginx Proxy Manager.
- The Docker socket is mounted read-only for container status widgets.
