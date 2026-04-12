# notes

Knowledge base and note-taking with Trilium Notes and Joplin Server.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **Trilium Notes** | Hierarchical note-taking app with WYSIWYG editing | `triliumnext/trilium` |
| **Joplin Server** | Note-taking and sync server for Joplin clients | `joplin/server` |
| **Joplin DB** | PostgreSQL database for Joplin Server | `postgres:16` |

## Network Architecture

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse proxy access |
| `joplin` | External | Internal communication between Joplin Server and its database |

## Ports

| Service | Host Port | Container Port | Notes |
|---------|-----------|----------------|-------|
| Trilium | `${TRILIUM_PORT}` | 8080 | Web UI (localhost only) |
| Joplin Server | `${JOPLIN_PORT}` | 22300 | Web UI (localhost only) |
| Joplin DB | `${JOPLIN_DB_PORT}` | 5432 | PostgreSQL (localhost only) |

## Environment Variables

Create a `.env` file in the same directory as the compose file.

| Variable | Description | Example |
|----------|-------------|---------|
| `TRILIUM_PORT` | Trilium web UI host port | `8080` |
| `TIMEZONE` | Timezone | `America/New_York` |
| `TRILIUM_DATA_DIR` | Path to Trilium data directory | `/opt/trilium/data` |
| `JOPLIN_PORT` | Joplin Server host port | `22300` |
| `JOPLIN_BASE_URL` | Joplin Server public URL | `https://joplin.example.com` |
| `POSTGRES_PASSWORD` | Joplin DB password | `secretpassword` |
| `POSTGRES_DATABASE` | Joplin DB name | `joplin` |
| `POSTGRES_USER` | Joplin DB user | `joplin` |
| `POSTGRES_HOST` | Joplin DB hostname | `joplin-db` |
| `JOPLIN_DB_PORT` | Joplin DB host port | `5432` |
| `JOPLIN_DB_DIR` | Path to Joplin DB data directory | `/opt/joplin/db` |

## Volumes

| Container Path | Purpose |
|----------------|---------|
| `/home/node/trilium-data` | Trilium application data, notes database, and attachments |
| `/var/lib/postgresql/data` | Joplin PostgreSQL database storage |

## Deployment

1. Create the external networks if they don't exist:
   ```bash
   docker network create proxy_net
   docker network create joplin
   ```

2. Create your `.env` file with all required variables.

3. Deploy:
   ```bash
   docker compose up -d
   ```

4. Access Trilium at the reverse proxy URL and set your password on first launch.

5. Access Joplin Server at the reverse proxy URL. Default login is `admin@localhost` with password `admin`. Change the password immediately after first login.

## Notes

- Ports are bound to `127.0.0.1` for reverse proxy access only.
- Trilium uses SQLite internally — no external database required.
- This is the [TriliumNext](https://github.com/TriliumNext/Trilium) community fork.
- Joplin Server requires a PostgreSQL database, included as the `joplin-db` service.
- Connect Joplin desktop/mobile clients to the server via the sync settings using the `JOPLIN_BASE_URL`.
