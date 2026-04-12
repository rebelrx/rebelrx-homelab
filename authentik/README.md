# authentik

Self-hosted identity provider and SSO authentication with Authentik, supporting OAuth2, OpenID Connect, SAML, LDAP, and proxy authentication. Backed by PostgreSQL.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **Authentik Server** | Core server handling API, SSO, and frontend | `ghcr.io/goauthentik/server` |
| **Authentik Worker** | Background task runner (emails, notifications, outpost management) | `ghcr.io/goauthentik/server` |
| **PostgreSQL** | Database backend | `postgres:16-alpine` |

## Network Architecture

| Network | Type | Purpose |
|---------|------|---------|
| `authentik` | Internal | Server ↔ Worker ↔ PostgreSQL communication |
| `proxy_net` | External | Reverse proxy access (Server only) |

## Ports

| Service | Host Port | Container Port | Notes |
|---------|-----------|----------------|-------|
| Server | `${COMPOSE_PORT_HTTP}` | 9000 | HTTP |
| Server | `${COMPOSE_PORT_HTTPS}` | 9443 | HTTPS |

## Environment Variables

### `.env`

Create from `.env.example`. Used for Docker Compose interpolation only.

| Variable | Description | Example |
|----------|-------------|---------|
| `COMPOSE_PORT_HTTP` | HTTP host port | `9001` |
| `COMPOSE_PORT_HTTPS` | HTTPS host port | `9444` |
| `PG_DB` | PostgreSQL database name | `authentik` |
| `PG_USER` | PostgreSQL username | `authentik` |
| `AUTHENTIK_DATA` | Authentik data directory | `/mnt/data/authentik/data` |
| `AUTHENTIK_DATABASE` | PostgreSQL data directory | `/mnt/data/authentik/database` |
| `AUTHENTIK_TEMPLATES` | Custom templates directory | `/mnt/data/authentik/custom-templates` |
| `AUTHENTIK_CERTS` | Certificates directory | `/mnt/data/authentik/certs` |

### `docker-compose.env`

Create from `docker-compose.env.example`. Contains secrets passed directly to containers.

| Variable | Description | Example |
|----------|-------------|---------|
| `PG_PASS` | PostgreSQL password (generate with `openssl rand -base64 36`) | *(secret)* |
| `AUTHENTIK_SECRET_KEY` | Authentik secret key (generate with `openssl rand -base64 60`) | *(secret)* |
| `POSTGRES_PASSWORD` | PostgreSQL password (same value as `PG_PASS`) | *(secret)* |

## Dependencies

```
PostgreSQL (authentik-db) ──► Authentik Server
PostgreSQL (authentik-db) ──► Authentik Worker
```

Both the server and worker wait for PostgreSQL's healthcheck to pass before starting.

## Deployment

1. Create the external network if it doesn't exist:
   ```bash
   docker network create proxy_net
   ```

2. Create your environment files:
   ```bash
   cp .env.example .env
   cp docker-compose.env.example docker-compose.env
   nano .env
   nano docker-compose.env
   ```

3. Generate secrets for `docker-compose.env`:
   ```bash
   openssl rand -base64 36 | tr -d '\n'   # Use for PG_PASS and POSTGRES_PASSWORD
   openssl rand -base64 60 | tr -d '\n'   # Use for AUTHENTIK_SECRET_KEY
   ```

4. Deploy:
   ```bash
   docker compose up -d
   ```

5. Navigate to `http://<server-ip>:9001/if/flow/initial-setup/` to create the initial admin account (`akadmin`).

## Post-Setup

1. Create a personal administrator account.
2. Enable multi-factor authentication (MFA).
3. Disable the default `akadmin` account.
4. Configure a proxy host in Nginx Proxy Manager pointing to `authentik-server:9000`.

## Notes

- The worker runs as `root` to manage Docker outposts via the mounted Docker socket. Consider using a Docker Socket Proxy for additional security.
- As of version 2025.10, Authentik no longer requires Redis — all caching is handled by PostgreSQL.
- Authentik containers must use UTC internally. Do not mount `/etc/timezone` or `/etc/localtime` into the containers.
- `PG_PASS` and `POSTGRES_PASSWORD` must be set to the same value.
