# proxy

Reverse proxy and SSL management using Nginx Proxy Manager, providing a web UI for managing proxy hosts, SSL certificates, and access control.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **Nginx Proxy Manager** | Web-based reverse proxy with Let's Encrypt SSL | `docker.io/jc21/nginx-proxy-manager` |

## Network Architecture

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Shared network for all proxied services |
| `nextcloud-aio` | External | Shared network for Nextcloud AIO routing |

## Ports

| Service | Host Port | Container Port | Notes |
|---------|-----------|----------------|-------|
| HTTP | 80 | 80 | Public HTTP traffic |
| HTTPS | 443 | 443 | Public HTTPS traffic |
| Admin UI | `${NPM_ADMIN_PORT}` | 81 | Management interface (localhost only) |

## Environment Variables

Create a `.env` file in the same directory as the compose file.

| Variable | Description | Example |
|----------|-------------|---------|
| `NPM_ADMIN_PORT` | Admin UI host port | `81` |
| `TIMEZONE` | Timezone | `America/New_York` |
| `NPM_DATA` | Path to NPM data directory | `/opt/npm/data` |
| `NPM_LETSENCRYPT` | Path to Let's Encrypt certificate storage | `/opt/npm/letsencrypt` |

## Volumes

| Container Path | Purpose |
|----------------|---------|
| `/data` | NPM configuration, database, and proxy host configs |
| `/etc/letsencrypt` | SSL certificates managed by Let's Encrypt |

## Deployment

1. Create the external networks if they don't exist:
   ```bash
   docker network create proxy_net
   docker network create nextcloud-aio
   ```

2. Create your `.env` file with all required variables.

3. Deploy:
   ```bash
   docker compose up -d
   ```

4. Access the admin UI at `http://<host>:<NPM_ADMIN_PORT>`. Default credentials on first login are `admin@example.com` / `changeme`.

5. Configure proxy hosts for your services, pointing to their container names and internal ports.

## Notes

- Ports 80 and 443 are exposed publicly for HTTP/HTTPS traffic. The admin UI is bound to `127.0.0.1` only.
- IPv6 is disabled via `DISABLE_IPV6=true`.
- NPM is connected to both `proxy_net` and `nextcloud-aio` networks to route traffic to all proxied services including Nextcloud.
- Any service that needs to be reverse-proxied must be on `proxy_net` (or `nextcloud-aio` for Nextcloud services).
