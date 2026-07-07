# snapotter

Self-hosted file-processing service ([SnapOtter](https://docs.snapotter.com/guide/getting-started.html)) backed by Postgres and a Redis job broker. Reached through NPM over `proxy_net`; the database and broker sit on an internal-only network with no egress and no published host ports.

---

## Architecture

```
client → NPM (proxy_net) → snapotter
                            ├─ postgres  (snapotter net, internal)
                            └─ redis     (snapotter net, internal)
```

| Layer | Where | Role |
|-------|-------|------|
| Reverse proxy + TLS | NPM (same host) | Terminates TLS, proxies to the `snapotter` container over `proxy_net` |
| Application | `snapotter` container | Joined to `proxy_net` (NPM-reachable) and the internal `snapotter` network |
| Data + broker | `postgres`, `redis` | Internal `snapotter` network only — no egress, no published ports |

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `snapotter` | `snapotter/snapotter:latest` | File-processing application (see [docs](https://docs.snapotter.com/guide/getting-started.html)) |
| `snapotter-db` | `postgres:17-alpine` | Postgres backend (`snapotter` database) |
| `snapotter-redis` | `redis:8-alpine` | Redis job broker (AOF-persisted, non-evicting) |

> `snapotter/snapotter:latest` is **not pinned**. Pin to a concrete version tag before relying on this stack — WUD watches for new tags but is notify-only (`wud.trigger.docker.enable=false`), so upgrades are manual.

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External bridge | NPM ↔ `snapotter` ingress, plus the app's outbound egress route |
| `snapotter` | Internal bridge (`internal: true`) | app ↔ Postgres ↔ Redis; no egress |

> The backend network is `internal: true`, so Postgres and Redis have no route off-host. The app still reaches the outside world (and NPM reaches the app) via `proxy_net`, which is a normal bridge with a gateway.

---

## Ports

| Service | Host binding | Container | Notes |
|---------|--------------|-----------|-------|
| `snapotter` | — (not published) | app port (confirm from docs) | NPM targets the container directly on `proxy_net` |

DB and Redis are not published.

> **Why no host port?** NPM runs on the same host and shares `proxy_net`, so it reaches the container container-to-container over the bridge — no host binding needed. Nothing is exposed on the host's LAN or loopback. (Contrast the Seafile stack, where the proxy lives on a separate VPS and the app *must* publish to a reachable interface.) Set the NPM proxy host's upstream to `snapotter:<app-port>` on `proxy_net`; confirm the internal port from the docs.
>
> **To publish on the host** (the repo's default for most stacks) — e.g. for a standalone deployment without NPM — first confirm SnapOtter's internal port from the docs, then add a mapping to the `snapotter` service in `compose.yaml`:
>
> ```yaml
>     ports:
>       - ${SNAPOTTER_PORT}:<container-port>
> ```
>
> and set `SNAPOTTER_PORT` in `.env`. It's left unpublished by default here only because the container port isn't confirmed.

---

## Volumes

| Host path (`.env` var) | Container path | Mode | Purpose |
|------------------------|----------------|------|---------|
| `${SNAPOTTER_DATA_PATH}` | `/data` | rw | SnapOtter app data — processed files and working state |
| `${SNAPOTTER_DB_PATH}` | `/var/lib/postgresql/data` | rw | Postgres data directory (local SSD) |
| `${SNAPOTTER_CACHE_PATH}` | `/data` | rw | Redis AOF persistence — job broker state |

> Redis runs `--appendonly yes` with `--maxmemory-policy noeviction`, i.e. a durable broker rather than an evictable cache. Don't set a `maxmemory` cap that would start rejecting writes. The `${SNAPOTTER_CACHE_PATH}` volume holds queued/in-flight job state, not committed application data.

---

## Environment Variables

### `.env` (Compose interpolation)

#### Service config

| Variable | Example | Purpose |
|----------|---------|---------|
| `SNAPOTTER_ADMIN_USER` | `admin` | Bootstrap admin username (`DEFAULT_USERNAME`). Typically seeded on first boot only — confirm against docs |

#### Secrets

| Variable | Purpose |
|----------|---------|
| `SNAPOTTER_DB_PASSWORD` | Postgres password. Feeds both `POSTGRES_PASSWORD` (db) and `DATABASE_URL` (app) from one var, so they can't drift. Lands in a URL → must be URL-safe: `openssl rand -hex 32` |
| `SNAPOTTER_ADMIN_PASSWORD` | First admin account password (`DEFAULT_PASSWORD`). Not URL-embedded, so `openssl rand -base64 24`. Typically first-boot only |

#### Volumes

| Variable | Example | Purpose |
|----------|---------|---------|
| `SNAPOTTER_DATA_PATH` | `/path/to/snapotter/data` | Host path for the app's `/data` |
| `SNAPOTTER_DB_PATH` | `/path/to/snapotter/pgdata` | Host path for the Postgres data directory |
| `SNAPOTTER_CACHE_PATH` | `/path/to/snapotter/redis` | Host path for Redis `/data` |

> `AUTH_ENABLED`, `DEFAULT_USERNAME`, and `DEFAULT_PASSWORD` are hardcoded/interpolated in `compose.yaml` and came from the original compose — confirm the exact names and their first-boot semantics against the SnapOtter docs. `DATABASE_URL` and `REDIS_URL` are set in `compose.yaml` (Redis is unauthenticated); only the DB password flows in from `.env`.

---

## Security posture

**In place:**

- **Network isolation.** The `snapotter` backend network is `internal: true` — Postgres and Redis have no egress and no published host ports. Ingress reaches the app through NPM over `proxy_net` only.
- **WUD notify-only** on all three services (`wud.trigger.docker.enable=false`).
- **Dependency gating.** `depends_on: service_healthy` on Postgres and Redis (both healthchecked), so the app won't start against an unready backend.
- **Secrets in `.env`.** Admin login and the Postgres password; the latter feeds `POSTGRES_PASSWORD` and `DATABASE_URL` from a single var.

**Intentionally rolled back** (caused runtime issues during bring-up; removed for a stable stack, can be reintroduced incrementally):

- `no-new-privileges: true` on all three services.
- `cap_drop: ALL` + minimal `cap_add` on Postgres and Redis.
- Redis `--requirepass` / `REDISCLI_AUTH` — Redis is unauthenticated, reachable only on the internal network.

---

## Deployment

### Prerequisites

- `proxy_net` external network exists (`docker network create proxy_net` if missing)
- `snapotter` internal network is created automatically by compose
- Data directories exist (match your `SNAPOTTER_*` paths): `mkdir -p /path/to/snapotter/{data,pgdata,redis}`
- NPM proxy host configured to target `snapotter:<app-port>` on `proxy_net` (internal port from the docs)
- Generated values for `SNAPOTTER_DB_PASSWORD` and `SNAPOTTER_ADMIN_PASSWORD`

### Bring up

```bash
cd /opt/stacks/snapotter

# Copy the example env file and fill it in
cp .env.example .env
chmod 600 .env

# Generate secrets, then paste into .env:
openssl rand -hex 32     # -> SNAPOTTER_DB_PASSWORD  (URL-safe; lands in DATABASE_URL)
openssl rand -base64 24  # -> SNAPOTTER_ADMIN_PASSWORD

nano .env  # set the two secrets and confirm the volume paths

docker compose up -d
docker compose logs -f snapotter
```

### First-run setup

1. Watch the logs — Postgres and Redis must report healthy (they gate the app via `depends_on`), then the app finishes its own initialisation. The exact init sequence is in the SnapOtter docs.
2. Confirm all three containers are up: `docker compose ps`.
3. Reach the app through its NPM hostname and log in with `SNAPOTTER_ADMIN_USER` / `SNAPOTTER_ADMIN_PASSWORD`.

---

## Backup

Critical paths to include in Kopia / Borgmatic snapshots:

- `${SNAPOTTER_DATA_PATH}` — app data and processed files.
- `${SNAPOTTER_DB_PATH}` — Postgres data (application state, accounts, job history).
- `.env` — needed to bring the stack back up with the same DB password.

> `${SNAPOTTER_CACHE_PATH}` (Redis AOF) is broker state — worth snapshotting to avoid losing queued jobs, but not application data of record.
>
> For Postgres, a periodic logical dump is safer than relying solely on file-level snapshots of a live data dir:
>
> ```bash
> docker exec -t snapotter-db \
>   pg_dumpall -U snapotter \
>   | gzip > /path/to/snapotter/dumps/snapotter-$(date +%F).sql.gz
> ```

---

## Maintenance

### Enter a container

```bash
docker exec -it snapotter /bin/sh          # app (bash may or may not be present)
docker exec -it snapotter-db psql -U snapotter   # Postgres shell
```

### Logs

```bash
docker compose logs -f snapotter
```

In-container log locations (if any) — confirm against the docs.

### Admin password reset

Rotate via the SnapOtter UI. If a CLI reset path exists, confirm it against the docs — the `DEFAULT_PASSWORD` env var only applies on first boot and won't change an existing account.

### Rotate the DB password

Because `POSTGRES_PASSWORD` only applies on first-init of an empty data dir, changing `SNAPOTTER_DB_PASSWORD` against an existing `pgdata` volume won't re-password the database — the app's `DATABASE_URL` would then present the new value to a DB that still expects the old one. Either:

```bash
# rotate the role in place, then set the same value in .env and recreate the app
docker exec -it snapotter-db psql -U snapotter \
  -c "ALTER USER snapotter PASSWORD 'new-hex-value';"
```

…or, if the volume holds nothing you need, wipe `${SNAPOTTER_DB_PATH}` and let it re-init against the new value.

---

## Notes & Gotchas

- **DB password ↔ volume lifecycle.** `SNAPOTTER_DB_PASSWORD` feeds both `POSTGRES_PASSWORD` and `DATABASE_URL`, so they can't drift — but Postgres only reads `POSTGRES_PASSWORD` when initialising an *empty* `pgdata`. Rotating the secret later requires an in-place `ALTER USER` or a fresh volume (see Maintenance).
- **Redis is a durable broker.** `--appendonly yes` + `noeviction`. Losing `${SNAPOTTER_CACHE_PATH}` drops queued/in-flight jobs but not committed application data. Don't add a `maxmemory` cap that would reject writes. (Note: `yes` is quoted in `compose.yaml` — `--appendonly` `"yes"` — so YAML doesn't coerce it to a boolean.)
- **Redis is unauthenticated.** Reachable only on the internal `snapotter` network. `--requirepass` was rolled back during bring-up.
- **Rolled-back hardening.** `no-new-privileges` and `cap_drop`/`cap_add` were removed after runtime issues. Reintroduce incrementally once the required caps are pinned down for each image.
- **Image pinning.** The app image is `:latest`; Postgres and Redis are pinned to major-`alpine`. Pin the app to a concrete tag before depending on the stack.
- **Unverified app config.** `AUTH_ENABLED` / `DEFAULT_USERNAME` / `DEFAULT_PASSWORD`, the first-boot semantics of the admin vars, and the app's listen port are carried from the original compose, not verified against the docs.
- **No app healthcheck.** `depends_on` gates the app on Postgres/Redis health, but the app container itself has none. Add one once a health endpoint and an in-image tool (`docker exec snapotter sh -c 'command -v wget curl nc'`) are confirmed.

---

## References

- [SnapOtter — getting started](https://docs.snapotter.com/guide/getting-started.html)
- [postgres (Docker Official)](https://hub.docker.com/_/postgres)
- [redis (Docker Official)](https://hub.docker.com/_/redis)
- [Top-level README](../../README.md)
