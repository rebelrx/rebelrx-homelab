# paperless

Self-hosted document management with [Paperless-ngx](https://docs.paperless-ngx.com/) — OCR, full-text search, tagging, and auto-filing for scanned PDFs and Office documents.

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `paperless-webserver` (service key `webserver`) | `ghcr.io/paperless-ngx/paperless-ngx:latest` | Web UI, API, document consumer, OCR pipeline |
| `paperless-db` (service key `db`) | `postgres:18` | PostgreSQL database |
| `paperless-broker` (service key `broker`) | `redis:8` | Task queue broker for async document processing |
| `paperless-gotenberg` (service key `gotenberg`) | `gotenberg/gotenberg:8.25` | Office document → PDF conversion (Word, Excel, PowerPoint, etc.) |
| `paperless-tika` (service key `tika`) | `apache/tika:latest` | Content extraction from Office documents (for indexing) |

> Gotenberg and Tika together allow Paperless to ingest Office docs — without them, Paperless can only handle images and native PDFs.

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse-proxy access via NPM (webserver only) |
| `paperless` | Internal bridge | Webserver ↔ DB ↔ Redis ↔ Gotenberg ↔ Tika |

> Only `paperless-webserver` joins `proxy_net`. Everything else is internal-only.

---

## Ports

| Host port (`.env` var) | Container port | Bind | Purpose |
|------------------------|----------------|------|---------|
| `${PAPERLESS_PORT}` (`8000`) | `8000` | all interfaces | Web UI + API |

Only `paperless-webserver` publishes a port; the DB, broker, Gotenberg, and Tika stay internal. Published on all interfaces so it's reachable on your LAN out of the box. To keep it behind a reverse proxy / loopback only, prefix the mapping in `compose.yaml` with `127.0.0.1:`. `8000` is a common port — change `PAPERLESS_PORT` if it clashes.

---

## Volumes

### Webserver bind mounts

| Host path (`.env` var) | Container path | Purpose |
|------------------------|----------------|---------|
| `${PAPERLESS_DATA}` | `/usr/src/paperless/data` | Search index, classifier model, internal state |
| `${PAPERLESS_MEDIA}` | `/usr/src/paperless/media` | **Stored documents (originals + archive PDFs)** — the actual library |
| `${PAPERLESS_EXPORT}` | `/usr/src/paperless/export` | Drop-zone for `document_exporter` runs |
| `${PAPERLESS_CONSUME}` | `/usr/src/paperless/consume` | **Watch folder** — files dropped here get auto-ingested by the consumer |

### Database

| Host path (`.env` var) | Container path | Purpose |
|------------------------|----------------|---------|
| `${PAPERLESS_PGDATA}` | `/var/lib/postgresql` | PostgreSQL root (note: mounts the *parent* of `data/`, per Paperless's official recipe) |

### Redis

| Volume | Container path | Purpose |
|--------|----------------|---------|
| `redisdata` (named Docker volume) | `/data` | Redis persistence; lives under `/var/lib/docker/volumes/paperless_redisdata/_data` |

> **Library + bulk data on the NAS; DB on local NVMe.** Same pattern as `immich`: PostgreSQL needs fast local disk and breaks on NFS, the document library benefits from NAS redundancy.

---

## Environment Variables

Paperless splits config across **two env files** — `.env` for compose interpolation (paths, ports), `docker-compose.env` for container runtime config (Paperless-ngx settings, OCR languages, DB credentials). Both the webserver **and** the db service load `docker-compose.env` via `env_file:` in compose.

### `.env` (Compose interpolation)

| Variable | Example | Purpose |
|----------|---------|---------|
| `PAPERLESS_PORT` | `8000` | Host port published for the web UI |
| `PAPERLESS_DATA` | `/path/to/nas-data/paperless/data` | Host bind for app state |
| `PAPERLESS_MEDIA` | `/path/to/nas-data/paperless/media` | Host bind for document library |
| `PAPERLESS_EXPORT` | `/path/to/nas-data/paperless/export` | Host bind for export output |
| `PAPERLESS_CONSUME` | `/path/to/nas-data/paperless/consume` | Host bind for watch folder |
| `PAPERLESS_PGDATA` | `/path/to/paperless/pgdata` | Host bind for Postgres (local NVMe) |

### `docker-compose.env` (Container runtime)

#### Critical (must be set before first run)

| Variable | Purpose |
|----------|---------|
| `PAPERLESS_URL` | **Must match the public URL exactly** (e.g. `https://paperless.example.com`). Used for CSRF/CORS — wrong value = login broken |
| `PAPERLESS_SECRET_KEY` | Django secret key — session signing. **Blank in the template; you MUST set it.** Generate with `openssl rand -base64 64` |
| `POSTGRES_PASSWORD` | Postgres account password (used by the db container to initialize). **Blank in the template; you MUST set it.** Generate with `openssl rand -base64 32` |
| `PAPERLESS_DBPASS` | The webserver's DB client password. **Must be IDENTICAL to `POSTGRES_PASSWORD`** — same account, two sides. Set both to the same value |

#### General

| Variable | Example | Purpose |
|----------|---------|---------|
| `PAPERLESS_TIME_ZONE` | `America/New_York` | Container TZ for log timestamps and consumer scheduling |
| `POSTGRES_USER` | `paperless` | Postgres username |
| `POSTGRES_DB` | `paperless` | Postgres database name |

#### Wiring (don't change unless restructuring)

| Variable | Value | Purpose |
|----------|-------|---------|
| `PAPERLESS_REDIS` | `redis://broker:6379` | Internal Redis URL — resolves over `paperless` network |
| `PAPERLESS_DBHOST` | `db` | Internal DB hostname (matches compose service key) |
| `PAPERLESS_TIKA_ENABLED` | `1` | Enable Tika for Office docs |
| `PAPERLESS_TIKA_GOTENBERG_ENDPOINT` | `http://gotenberg:3000` | Internal Gotenberg URL |
| `PAPERLESS_TIKA_ENDPOINT` | `http://tika:9998` | Internal Tika URL |

#### Behavior / filing rules

| Variable | Value | Purpose |
|----------|-------|---------|
| `PAPERLESS_FILENAME_FORMAT` | `{{ document_type }}/{{ correspondent }}/{{ created_year }}/{{ title }}` | How filed documents are organized inside `${PAPERLESS_MEDIA}/documents/archive/`. Changes apply only to newly-filed or re-filed documents — use `document_renamer` from the management UI to apply retroactively |
| `PAPERLESS_CONSUMER_RECURSIVE` | `true` | Watch consume folder subdirectories too |
| `PAPERLESS_CONSUMER_SUBDIRS_AS_TAGS` | `true` | Auto-tag documents based on the subfolder they were dropped into |
| `PAPERLESS_CONSUMER_POLLING` | `10` | Poll the consume folder every 10s (inotify is finicky on NFS, polling is more reliable) |
| `PAPERLESS_APP_TITLE` | `Paperless-ngx` | Browser tab title |

#### Optional / commented in template

| Variable | Default if unset | Purpose |
|----------|------------------|---------|
| `USERMAP_UID` | `1000` | UID Paperless runs as (must own the bind-mounted host dirs) |
| `USERMAP_GID` | `1000` | GID Paperless runs as |
| `PAPERLESS_OCR_LANGUAGE` | `eng` | Primary OCR language |
| `PAPERLESS_OCR_LANGUAGES` | (none extra) | Additional Tesseract language packs to install |

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme |
|----------|---------------|---------------|--------|
| `paperless.example.com` | `paperless-webserver` | `8000` | `http` |

> **`PAPERLESS_URL` must match this hostname exactly** — Django uses it for CSRF token validation. Mismatched values cause login to fail with cryptic CSRF errors.
>
> **Bump `client_max_body_size` in NPM** — Paperless ingests PDFs that can run tens of MB for high-res scans. Set `100M` or higher.
>
> **Enable WebSocket support in NPM** — Paperless uses WebSockets for live task progress updates.

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing)
- `paperless` internal network is created automatically by compose
- Your NAS NFS share mounted on the host (e.g. `/mnt/nas`) if pointing the library paths at NAS storage
- Generated values for `PAPERLESS_SECRET_KEY`, `POSTGRES_PASSWORD`, and `PAPERLESS_DBPASS`

### Bring up

```bash
cd /opt/stacks/paperless
cp .env.example .env
cp docker-compose.env.example docker-compose.env

# Set required secrets in docker-compose.env.
# POSTGRES_PASSWORD and PAPERLESS_DBPASS MUST be identical — generate once, set both.
DBPASS=$(openssl rand -base64 32 | tr -d '\n')
sed -i "s|PAPERLESS_SECRET_KEY=.*|PAPERLESS_SECRET_KEY=$(openssl rand -base64 64 | tr -d '\n')|" docker-compose.env
sed -i "s|POSTGRES_PASSWORD=.*|POSTGRES_PASSWORD=$DBPASS|" docker-compose.env
sed -i "s|PAPERLESS_DBPASS=.*|PAPERLESS_DBPASS=$DBPASS|" docker-compose.env

# Set PAPERLESS_URL to match your NPM hostname
sed -i "s|PAPERLESS_URL=.*|PAPERLESS_URL=https://paperless.example.com|" docker-compose.env

nano docker-compose.env  # review other settings (OCR language, filename format)

# Pre-create host dirs (match your ${PAPERLESS_PGDATA} / library paths)
sudo mkdir -p /path/to/paperless/pgdata
sudo mkdir -p /path/to/nas-data/paperless/{data,media,export,consume}

chmod 600 .env docker-compose.env

docker compose up -d
```

### First-run setup

1. Wait ~60s for Postgres to initialize, migrations to run, and the search index to build.
2. Create the **first superuser** (Paperless has no default admin):

   ```bash
   docker compose exec webserver python manage.py createsuperuser
   ```

   Prompts for username, email, and password.

3. Open `https://paperless.example.com` (or `http://<host-ip>:8000` directly) and log in.
4. Configure tags, document types, correspondents to match how you want documents filed.
5. Drop a test PDF into `${PAPERLESS_CONSUME}` and verify it's picked up within ~10s (polling interval).

---

## Backup

Paths to include in Kopia / Borgmatic snapshots:

- `${PAPERLESS_PGDATA}` — DB (users, tags, correspondents, metadata, search index pointers)
- `${PAPERLESS_DATA}` — search index files, classifier model, log
- `${PAPERLESS_MEDIA}` — **the actual documents.** Bulk content on the NAS, already covered by NAS-level snapshots — list here for completeness
- `.env` and `docker-compose.env` — both contain secrets

`${PAPERLESS_CONSUME}` and `${PAPERLESS_EXPORT}` are transient working directories and don't need backup.

> **For pre-upgrade snapshots, use Paperless's built-in `document_exporter`:**
>
> ```bash
> docker compose exec webserver document_exporter ../export
> ```
>
> This produces a self-contained dump in `${PAPERLESS_EXPORT}` (manifest + original files + archive PDFs) that can be imported into any Paperless instance via `document_importer`. Far more portable than DB-level dumps if you ever need to migrate or restore selectively.

---

## Notes & Gotchas

- **Two env files split by purpose.** Paperless is the only stack in this homelab that uses `env_file: docker-compose.env` for container runtime config alongside the standard `.env` for compose interpolation. Both the webserver and the db service load `docker-compose.env`. Both files are committed only as `*.example` — never check in the actual files.
- **`POSTGRES_PASSWORD` and `PAPERLESS_DBPASS` must be identical.** One is the account password the db container initializes with; the other is the password the webserver connects with. They default to `paperless` in upstream examples (which is why the defaults "work"), but once you set a real password you must set it in both places or the webserver can't authenticate to Postgres.
- **`PAPERLESS_URL` is a CSRF gate.** Setting it to anything other than the URL users actually visit causes login attempts to fail with `Forbidden (CSRF cookie not set.)` or similar in the logs. If you ever change the public hostname or proxy setup, update this in lockstep.
- **Filename format is opinionated.** The current `{{ document_type }}/{{ correspondent }}/{{ created_year }}/{{ title }}` template organizes the on-disk archive into a `<type>/<correspondent>/<year>/<title>.pdf` hierarchy. Useful if you ever browse the underlying files directly, but means re-tagging a correspondent or document type moves files around. Use the `document_renamer` management command to re-apply the template after schema changes.
- **Consumer polling, not inotify.** `PAPERLESS_CONSUMER_POLLING=10` because the consume folder is on NFS, where inotify is unreliable. Trade-off: up to 10s delay between dropping a file and Paperless noticing it. Bump higher if you don't care about latency and want to reduce idle scan load.
- **`pgdata` mounts the parent, not `data/`.** Per Paperless's official compose, `/var/lib/postgresql` is mounted (not `/var/lib/postgresql/data`). This persists Postgres's run-time files too — slightly unusual but matches upstream. Don't "fix" it.
- **Gotenberg is version-pinned.** `gotenberg:8.25` rather than `:latest`, because Paperless has a specific compatibility matrix. WUD will notify on new Gotenberg releases — verify against Paperless's compatibility notes before updating.
- **PostgreSQL pinned to 18.** The image is pinned to `postgres:18`, but the WUD labels don't yet constrain tag matching with `wud.tag.include=^18`, so WUD will eventually suggest a breaking 18 → 19 upgrade. Ignore those notifications until the constraint is in place or Paperless's supported PG version moves up.

---

## References

- [Paperless-ngx docs — Docker install](https://docs.paperless-ngx.com/setup/#docker)
- [Paperless-ngx docs — configuration](https://docs.paperless-ngx.com/configuration/)
- [Paperless-ngx docs — backup and restore](https://docs.paperless-ngx.com/administration/#backup)
- [Top-level README](../../README.md)
