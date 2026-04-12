# recipes

Recipe management with Mealie, a self-hosted recipe organizer with meal planning, shopping lists, and web scraping. Backed by PostgreSQL.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **Mealie** | Recipe management web application | `ghcr.io/mealie-recipes/mealie` |
| **PostgreSQL** | Database backend | `postgres` |

## Network Architecture

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse proxy access and inter-service communication |

## Ports

| Service | Host Port | Container Port | Notes |
|---------|-----------|----------------|-------|
| Mealie | `${MEALIE_PORT}` | 9000 | Web UI |

## Environment Variables

Create a `.env` file in the same directory as the compose file.

### Global

| Variable | Description | Example |
|----------|-------------|---------|
| `PUID` | User ID for file permissions | `1000` |
| `PGID` | Group ID for file permissions | `1000` |
| `TIMEZONE` | Timezone | `America/New_York` |

### Mealie

| Variable | Description | Example |
|----------|-------------|---------|
| `MEALIE_PORT` | Mealie web UI host port | `9000` |
| `URL` | Public base URL for Mealie | `https://mealie.example.com` |
| `MEALIE_DATA` | Path to Mealie data directory | `/opt/mealie/data` |

### PostgreSQL

| Variable | Description | Example |
|----------|-------------|---------|
| `POSTGRES_USER` | PostgreSQL username | `mealie` |
| `POSTGRES_PASSWORD` | PostgreSQL password | *(secret)* |
| `POSTGRES_PORT` | PostgreSQL port (internal) | `5432` |
| `POSTGRES_DB` | PostgreSQL database name | `mealie` |
| `PGUSER` | User for `pg_isready` healthcheck | `mealie` |
| `MEALIE_PGDATA` | Path to PostgreSQL data directory | `/opt/mealie/pgdata` |

## Dependencies

```
PostgreSQL (postgres) ──► Mealie
```

Mealie waits for PostgreSQL's healthcheck (`pg_isready`) to pass before starting.

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

4. Access Mealie at `http://<host>:<MEALIE_PORT>` and create your admin account.

## Notes

- Mealie has a memory limit of 1000MB set via `deploy.resources.limits`.
- `ALLOW_SIGNUP` is set to `false` — new users must be created by an admin.
- PostgreSQL is pinned to `:17`.
- Mealie connects to PostgreSQL using the internal service name `postgres` on `POSTGRES_PORT`.
