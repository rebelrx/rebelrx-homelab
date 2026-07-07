# mealie

Self-hosted recipe management with [Mealie](https://mealie.io/) — recipe library, meal planning, shopping lists, and one-click recipe import from any URL.

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `mealie-server` (service key `mealie`) | `ghcr.io/mealie-recipes/mealie:latest` | Web UI, API, recipe parser |
| `mealie-db` (service key `postgres`) | `postgres:17` | PostgreSQL database |

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse-proxy access via NPM (server only) |
| `mealie` | Internal bridge | Server ↔ PostgreSQL |

> Only `mealie-server` joins `proxy_net`. The DB is reachable only on the internal `mealie` network.

---

## Ports

| Host port (`.env` var) | Container port | Bind | Purpose |
|------------------------|----------------|------|---------|
| `${MEALIE_PORT}` (`9925`) | `9000` | all interfaces | Web UI + API |

Only `mealie-server` publishes a port; the DB stays internal. `9925` is the host port from Mealie's own docs (`9925:9000`); the container listens on `9000`. Published on all interfaces so it's reachable on your LAN out of the box. To keep it behind a reverse proxy / loopback only, prefix the mapping in `compose.yaml` with `127.0.0.1:`.

---

## Volumes

| Host path (`.env` var) | Container path | Used by | Purpose |
|------------------------|----------------|---------|---------|
| `${MEALIE_DATA}` | `/app/data` | mealie | Recipe images, attachments, backup files, app state |
| `${MEALIE_PGDATA}` | `/var/lib/postgresql/data` | mealie-db | PostgreSQL data — all recipes, meal plans, users, shopping lists |

---

## Environment Variables

### `.env` (Compose interpolation)

#### General

| Variable | Example | Purpose |
|----------|---------|---------|
| `PUID` | `1000` | UID inside container |
| `PGID` | `1000` | GID inside container |
| `TIMEZONE` | `America/New_York` | Container TZ |
| `MEALIE_PORT` | `9925` | Host port published for the web UI (container listens on `9000`) |
| `URL` | `https://mealie.example.com` | Public URL Mealie advertises to clients (used in share links, email invitations) |

#### Database

| Variable | Example | Purpose |
|----------|---------|---------|
| `POSTGRES_USER` | `mealie` | Postgres username (also used as `PGUSER` for `pg_isready` healthcheck) |
| `POSTGRES_PASSWORD` | *(secret)* | Postgres password — **the template default `mealie` MUST be replaced.** Generate with `openssl rand -base64 32` |
| `POSTGRES_DB` | `mealie` | Postgres database name |
| `PGUSER` | `mealie` | Used by Postgres CLI tools (`pg_isready`) — same value as `POSTGRES_USER` |

#### Volumes

| Variable | Example | Purpose |
|----------|---------|---------|
| `MEALIE_DATA` | `/path/to/mealie/data` | Host bind for app data |
| `MEALIE_PGDATA` | `/path/to/mealie/pgdata` | Host bind for Postgres data |

### Hardcoded in `compose.yaml`

| Variable | Value | Purpose |
|----------|-------|---------|
| `ALLOW_SIGNUP` | `false` | Disables open registration — only admins can create users (typical for a personal instance) |
| `DB_ENGINE` | `postgres` | Use PostgreSQL instead of the default SQLite |
| `POSTGRES_SERVER` | `postgres` | Internal hostname for the DB (matches the compose service key) |
| `POSTGRES_PORT` | `5432` | DB port (internal) |

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme |
|----------|---------------|---------------|--------|
| `mealie.example.com` | `mealie-server` | `9000` | `http` |

> **Bump `client_max_body_size` in NPM** — Mealie attachments (PDF cookbook scans, large recipe images) can exceed the default 1 MB. Set `50M` or higher.
>
> **`URL` must match the NPM hostname exactly.** Diverging values still authenticate, but share links and email invitations build broken URLs.

---

## Resource Limits

Mealie is configured with a 1 GB memory ceiling in `compose.yaml`:

```yaml
deploy:
  resources:
    limits:
      memory: 1000M
```

This is a soft guardrail against runaway memory use during recipe imports (the parser can balloon temporarily when ingesting recipes from sites with heavy embedded media or long ingredient lists). Bump it if you see OOM kills in the logs — start with `2000M`.

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing)
- `mealie` internal network created automatically by compose
- Generated value for `POSTGRES_PASSWORD`

### Bring up

```bash
cd /opt/stacks/mealie
cp .env.example .env

# Set DB password
sed -i "s|POSTGRES_PASSWORD=.*|POSTGRES_PASSWORD=$(openssl rand -base64 32 | tr -d '\n')|" .env

nano .env  # set URL
chmod 600 .env

docker compose up -d
```

### First-run setup

1. Wait ~30s for Postgres to initialize and Mealie to run migrations.
2. Open the app (e.g. `https://mealie.example.com` via NPM, or `http://<host-ip>:9925` directly).
3. Default admin credentials: `changeme@example.com` / `MyPassword` — **log in and change them immediately** under the admin profile.
4. Create non-admin user accounts as needed.
5. Optionally configure SMTP under Admin → Site Settings for invitation emails and password resets.

---

## Backup

Paths to include in Kopia / Borgmatic snapshots:

- `${MEALIE_PGDATA}` — all recipes, meal plans, users, shopping lists
- `${MEALIE_DATA}` — recipe images and attachments
- `.env` — contains `POSTGRES_PASSWORD`

> Mealie has a built-in backup feature (Admin → **Backups → Create Heartbeat**) that produces a portable archive containing both the DB dump and asset bundle. Useful before major Mealie upgrades alongside file-level snapshots.
>
> For pure Postgres dumps:
>
> ```bash
> docker compose exec -T postgres pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB" \
>   | gzip > /path/to/mealie/dumps/mealie-$(date +%F).sql.gz
> ```

---

## References

- [Mealie docs — Postgres install](https://docs.mealie.io/documentation/getting-started/installation/postgres/)
- [Mealie docs — configuration](https://docs.mealie.io/documentation/getting-started/installation/backend-config/)
- [Top-level README](../../README.md)
