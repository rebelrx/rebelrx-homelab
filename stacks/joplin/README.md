# joplin

Self-hosted note synchronization server with [Joplin Server](https://joplinapp.org/) — sync target for Joplin desktop and mobile clients with end-to-end encryption.

> This stack hosts the **sync server only**. Note-taking happens in the Joplin desktop/mobile apps; this server is what they sync to.

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `joplin-server` (service key `joplin`) | `joplin/server:latest` | Joplin sync API, web admin UI |
| `joplin-db` | `postgres:16` | PostgreSQL database |

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse-proxy access via NPM (server only) |
| `joplin` | Internal bridge | Server ↔ PostgreSQL |

> Only `joplin-server` joins `proxy_net`. The DB is reachable only on the internal `joplin` network.

---

## Ports

| Host port (`.env` var) | Container port | Bind | Purpose |
|------------------------|----------------|------|---------|
| `${JOPLIN_PORT}` (`22300`) | `22300` | all interfaces | Sync API + web admin UI |

Only `joplin-server` publishes a port; the DB stays internal. Published on all interfaces so it's reachable on your LAN out of the box. The container port `22300` is set via `APP_PORT` in `compose.yaml`. To keep it behind a reverse proxy / loopback only, prefix the mapping in `compose.yaml` with `127.0.0.1:`.

---

## Volumes

| Host path (`.env` var) | Container path | Used by | Purpose |
|------------------------|----------------|---------|---------|
| `${JOPLIN_DB_DIR}` | `/var/lib/postgresql/data` | joplin-db | PostgreSQL data — all synced notes, attachments, and user accounts |

> The Joplin server container has **no persistent volume**. All state lives in Postgres. If you destroy the server container, your data is safe; if you destroy `${JOPLIN_DB_DIR}`, everything is gone.

---

## Environment Variables

### `.env` (Compose interpolation)

#### General

| Variable | Example | Purpose |
|----------|---------|---------|
| `TIMEZONE` | `America/New_York` | Container TZ |
| `JOPLIN_PORT` | `22300` | Host port published for the sync API / web UI |
| `JOPLIN_BASE_URL` | `https://joplin.example.com` | **Public URL Joplin advertises to clients.** Critical — used in share links, registration emails, and the URL clients connect to. Must match what NPM serves |

#### Database

| Variable | Example | Purpose |
|----------|---------|---------|
| `POSTGRES_USER` | `joplin` | Postgres username |
| `POSTGRES_DATABASE` | `joplin` | Postgres database name |
| `POSTGRES_HOST` | `joplin-db` | Internal hostname Joplin uses to reach the DB (matches the `joplin-db` container name) |
| `POSTGRES_PASSWORD` | *(secret)* | Postgres password — **the template default `joplin` MUST be replaced.** Generate with `openssl rand -base64 32` |
| `JOPLIN_DB_DIR` | `/path/to/joplin/postgres-data` | Host bind for Postgres data |

> `APP_PORT=22300`, `DB_CLIENT=pg`, and `POSTGRES_PORT=5432` are set directly in `compose.yaml` rather than `.env`.

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme |
|----------|---------------|---------------|--------|
| `joplin.example.com` | `joplin-server` | `22300` | `http` |

> **Bump `client_max_body_size` in NPM** — Joplin attachments (PDFs, images) can be tens of MB. Default 1 MB will reject anything larger. Set `100M` or higher on this host.
>
> **`JOPLIN_BASE_URL` must match the NPM hostname exactly.** If they diverge (typo, trailing slash, http vs https), clients still authenticate but share links and registration emails build broken URLs. Keep them in sync.

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing)
- `joplin` internal network is created automatically by compose
- Generated value for `POSTGRES_PASSWORD`

### Bring up

```bash
cd /opt/stacks/joplin
cp .env.example .env

# Set DB password
sed -i "s|POSTGRES_PASSWORD=.*|POSTGRES_PASSWORD=$(openssl rand -base64 32 | tr -d '\n')|" .env

nano .env  # set JOPLIN_BASE_URL
chmod 600 .env

docker compose up -d
```

### First-run setup

1. Wait ~30s for Postgres to initialize and Joplin to run migrations.
2. Open the app (e.g. `https://joplin.example.com` via NPM, or `http://<host-ip>:22300` directly).
3. Default admin credentials: `admin@localhost` / `admin` — **log in and change them immediately** under the admin profile.
4. Create non-admin user accounts as needed for each client device.
5. **Configure each Joplin client** (desktop, mobile):
   - Sync target: **Joplin Server**
   - URL: your public URL (e.g. `https://joplin.example.com`)
   - Email + password: matching user account from step 4
6. On the first sync from any device, enable **End-to-End Encryption** in the client. This encrypts your notes before they're uploaded; the server only stores ciphertext.

---

## Backup

Paths to include in Kopia / Borgmatic snapshots:

- `${JOPLIN_DB_DIR}` — **the only stateful path.** All notes, attachments, users live here
- `.env` — contains `POSTGRES_PASSWORD`

> For Postgres specifically, a periodic `pg_dump` is safer than file-level snapshots:
>
> ```bash
> docker compose exec -T joplin-db pg_dump -U "$POSTGRES_USER" "$POSTGRES_DATABASE" \
>   | gzip > /path/to/joplin/dumps/joplin-$(date +%F).sql.gz
> ```

> **End-to-end encryption note.** If you've enabled E2EE in the clients, the DB contains only encrypted blobs. Without the encryption keys (stored client-side), a restored DB is unreadable. Make sure at least one client has the E2EE master password backed up safely.

---

## Notes & Gotchas

- **No healthcheck on the joplin server.** Past Joplin Server releases have had inconsistent `/healthz` endpoint behavior, so the standard healthcheck pattern has been omitted here. Container readiness is monitored externally (Uptime Kuma, etc.) rather than via compose.
- **`depends_on` is plain, not `condition: service_healthy`.** On a slow boot, Joplin may race the DB and crash-loop a few times before settling. Docker's `restart: unless-stopped` handles the retry — once Postgres is up, Joplin connects on the next attempt and stays up.
- **PostgreSQL pinned to 16.** Joplin Server supports 12+. The image is pinned to `postgres:16`, but the WUD labels don't yet constrain tag matching with `wud.tag.include=^16`, so WUD will eventually suggest a breaking 16 → 17 upgrade. Ignore those notifications until the constraint is in place or Joplin Server's supported PG version moves up.
- **Default admin credentials.** The image ships with `admin@localhost` / `admin` baked in. Change immediately on first login. There's no way to set initial credentials via env var.
- **One DB, multiple clients per user is fine.** Joplin Server supports multiple devices syncing to the same user account. Two desktops + a phone all pointed at the same public URL with the same credentials is the intended pattern.

---

## References

- [Joplin Server source / docs](https://github.com/laurent22/joplin/tree/main/packages/server)
- [Joplin user guide — Joplin Server](https://joplinapp.org/help/apps/sync/joplin_server/)
- [Joplin E2EE documentation](https://joplinapp.org/help/apps/sync/e2ee/)
- [Top-level README](../../README.md)
