# git

Self-hosted Git server using Forgejo with a PostgreSQL database backend.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **Forgejo** | Lightweight self-hosted Git server (Gitea fork) | `codeberg.org/forgejo/forgejo` |
| **PostgreSQL** | Database backend for Forgejo | `postgres` |

## Network Architecture

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse proxy access |
| `forgejo` | Internal | Private communication between Forgejo and PostgreSQL |

## Ports

| Service | Host Port | Container Port | Notes |
|---------|-----------|----------------|-------|
| Forgejo HTTP | `${FORGEJO_HTTP_PORT}` | 3000 | Web UI (localhost only) |

## Environment Variables

Create a `.env` file in the same directory as the compose file.

### Global

| Variable | Description | Example |
|----------|-------------|---------|
| `USER_UID` | User ID for Forgejo process | `1000` |
| `USER_GID` | Group ID for Forgejo process | `1000` |
| `TIMEZONE` | Timezone | `America/New_York` |

### Forgejo

| Variable | Description | Example |
|----------|-------------|---------|
| `FORGEJO_HTTP_PORT` | Forgejo web UI host port | `3000` |
| `FORGEJO_DB_PASSWORD` | PostgreSQL password (shared with db) | *(secret)* |
| `FORGEJO_DATA` | Path to Forgejo data directory | `/opt/forgejo/data` |

### PostgreSQL

| Variable | Description | Example |
|----------|-------------|---------|
| `FORGEJO_DB_DATA` | Path to PostgreSQL data directory | `/opt/forgejo/db` |

## Dependencies

```
PostgreSQL (db) ──► Forgejo
```

Forgejo waits for PostgreSQL to pass its healthcheck (`pg_isready`) before starting.

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

4. Access Forgejo at `http://<host>:<FORGEJO_HTTP_PORT>` and complete the initial setup wizard.

## Notes

- Database credentials are configured via environment variables in the compose file (`FORGEJO__database__*`). The database name, user, and host are hardcoded to `forgejo`, `forgejo`, and `db:5432` respectively.
- Both Forgejo and PostgreSQL are pinned to major version `:14`.
- Port is bound to `127.0.0.1` for reverse proxy access only.
- PostgreSQL needs to be on `proxy_net` in addition to the internal `forgejo` network to avoid Bad Gateway errors when accessed via Tailnet.
