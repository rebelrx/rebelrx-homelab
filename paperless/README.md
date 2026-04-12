# paperless

Document management system using Paperless-ngx with OCR, full-text search, and automated document processing. Backed by PostgreSQL with Redis for task queuing, Gotenberg for document conversion, and Apache Tika for content extraction.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **Paperless-ngx** | Document management web application | `ghcr.io/paperless-ngx/paperless-ngx` |
| **Redis** | Message broker for background task processing | `docker.io/library/redis` |
| **PostgreSQL** | Database backend | `docker.io/library/postgres` |
| **Gotenberg** | Document-to-PDF conversion (HTML, Office docs, emails) | `docker.io/gotenberg/gotenberg` |
| **Apache Tika** | Content extraction and metadata parsing | `docker.io/apache/tika` |

## Network Architecture

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse proxy access and inter-service communication |

## Ports

| Service | Host Port | Container Port | Notes |
|---------|-----------|----------------|-------|
| Paperless-ngx | `${PAPERLESS_PORT}` | 8000 | Web UI (localhost only) |

All other services are internal only and communicate over `proxy_net`.

## Environment Variables

Paperless-ngx uses a separate `docker-compose.env` file for its configuration. Create this file alongside the compose file.

### .env File

| Variable | Description | Example |
|----------|-------------|---------|
| `PAPERLESS_PORT` | Paperless web UI host port | `8000` |
| `PAPERLESS_PGDATA` | Path to PostgreSQL data directory | `/opt/paperless/pgdata` |
| `PAPERLESS_DATA` | Path to Paperless application data | `/opt/paperless/data` |
| `PAPERLESS_MEDIA` | Path to processed document storage | `/opt/paperless/media` |
| `PAPERLESS_EXPORT` | Path to document export directory | `/opt/paperless/export` |
| `PAPERLESS_CONSUME` | Path to document consumption/inbox directory | `/opt/paperless/consume` |

### docker-compose.env File

See [Paperless-ngx configuration docs](https://docs.paperless-ngx.com/configuration/) for the full list of available options. Key variables include `PAPERLESS_URL`, `PAPERLESS_SECRET_KEY`, `PAPERLESS_DBHOST`, `PAPERLESS_REDIS`, `PAPERLESS_TIKA_ENABLED`, and `PAPERLESS_TIKA_GOTENBERG_ENDPOINT`.

## Dependencies

```
PostgreSQL (db) ───┐
Redis (broker) ────┤
Gotenberg ─────────┼──► Paperless-ngx (webserver)
Tika ──────────────┘
```

Paperless-ngx waits for PostgreSQL and Redis healthchecks to pass before starting. Gotenberg and Tika only need to be started.

## Deployment

1. Create the external network if it doesn't exist:
   ```bash
   docker network create proxy_net
   ```

2. Create your `.env` and `docker-compose.env` files with all required variables.

3. Deploy:
   ```bash
   docker compose up -d
   ```

4. Create an admin user:
   ```bash
   docker exec -it paperless_webserver python3 manage.py createsuperuser
   ```

5. Drop documents into the `PAPERLESS_CONSUME` directory or upload through the web UI.

## Notes

- Port is bound to `127.0.0.1` for reverse proxy access only.
- Gotenberg runs with JavaScript disabled and a file-only allowlist for Chromium to prevent external content loading when processing email files.
- Redis uses a named volume (`redisdata`) for persistence.
- Gotenberg is pinned to `8.25` and PostgreSQL to `18`.
