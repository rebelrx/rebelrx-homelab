# sparkyfitness

Self-hosted fitness, nutrition, and body-composition tracker with [SparkyFitness](https://codewithcj.github.io/SparkyFitness/) — a privacy-first MyFitnessPal alternative with nutrition/exercise/hydration/sleep tracking, goal-setting, multi-user support, OIDC/TOTP/Passkey auth, and optional AI assistant via MCP.

> ⚠️ **Active development.** Upstream warns that breaking changes happen between releases. Always read release notes before pulling new images; `wud.trigger.docker.enable=false` is set on every service in this stack so WUD notifies but never auto-pulls.

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `sparkyfitness-db` | `postgres:18.3-alpine` | PostgreSQL database — schema migrations run on first server start |
| `sparkyfitness-server` | `codewithcj/sparkyfitness_server:latest` | Node.js / Express 5 backend API; handles auth, encryption, integrations |
| `sparkyfitness-frontend` | `codewithcj/sparkyfitness:latest` | React 19 + Vite SPA served by Nginx; proxies `/api` to the backend |
| `sparkyfitness-mcp` | `codewithcj/sparkyfitness_mcp:latest` | Model Context Protocol server — exposes health data to AI clients (Claude Desktop, Cursor, etc.) over HTTP |

> Optional `sparkyfitness-garmin` (dedicated Garmin Connect sync microservice) is **not deployed** in this stack — Garmin sync via the server's built-in connector works for most cases.

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse-proxy access via NPM (frontend only) |
| `sparkyfitness_net` | Internal bridge | db ↔ server ↔ frontend ↔ mcp |

> Only `sparkyfitness-frontend` joins `proxy_net`. The backend, database, and MCP server are reachable only on the internal `sparkyfitness_net`.

---

## Ports

| Service | Host binding | Container port | Purpose |
|---------|--------------|----------------|---------|
| `sparkyfitness-frontend` | `${SPARKY_FITNESS_FRONTEND_PORT}` (`3004`), all interfaces | `80` | Nginx — frontend SPA + `/api` reverse proxy to backend |
| `sparkyfitness-mcp` | `127.0.0.1:${SPARKY_FITNESS_MCP_PORT}` (`3001`), loopback | `3001` | MCP HTTP transport — for AI client connections |

The frontend is published on all interfaces so it's reachable on your LAN out of the box (NPM also proxies it). To keep it behind the reverse proxy / loopback only, prefix its mapping in `compose.yaml` with `127.0.0.1:`.

> **The MCP server is deliberately bound to `127.0.0.1`** — unlike the frontend, it is not published on all interfaces. It's an authenticated API into your full health dataset and the AI/MCP feature is beta upstream. Reach it locally or over a VPN/tailnet; publish it more widely only if you understand the exposure.
>
> **Do not port-forward the frontend to the public internet.** Upstream explicitly states the app is not production-hardened. LAN + reverse proxy, with a VPN/tailnet for remote access, is the intended posture.
>
> `sparkyfitness-server` (port `3010`) and `sparkyfitness-db` (port `5432`) are **not** published to the host — only reachable inside `sparkyfitness_net`.

---

## Volumes

| Host path (`.env` var) | Container path | Used by | Purpose |
|------------------------|----------------|---------|---------|
| `${DB_PATH}` | `/var/lib/postgresql` | sparkyfitness-db | PostgreSQL data directory parent — Postgres uses a `./data` subdir |
| `${SERVER_BACKUP_PATH}` | `/app/SparkyFitnessServer/backup` | sparkyfitness-server | Destination for in-app DB backups (`db_backup.sh` output) |
| `${SERVER_UPLOADS_PATH}` | `/app/SparkyFitnessServer/uploads` | sparkyfitness-server | User-uploaded content — profile pictures, exercise images, AI vision uploads |

---

## Environment Variables

### `.env` (Compose interpolation)

#### Identity & paths

| Variable | Example | Purpose |
|----------|---------|---------|
| `PUID` | `1000` | User ID for file ownership on the bind mounts |
| `PGID` | `1000` | Group ID for file ownership |
| `TZ` | `America/New_York` | Server timezone — affects date/time in logs and records |
| `SPARKY_FITNESS_FRONTEND_PORT` | `3004` | Host port for the frontend (container `80`) |
| `SPARKY_FITNESS_MCP_PORT` | `3001` | Host port for the MCP server (bound to loopback) |
| `DB_PATH` | `/path/to/sparkyfitness/postgresql` | Host bind for Postgres data |
| `SERVER_BACKUP_PATH` | `/path/to/sparkyfitness/backup` | Host bind for server backups |
| `SERVER_UPLOADS_PATH` | `/path/to/sparkyfitness/uploads` | Host bind for user uploads |

#### Database

| Variable | Example | Purpose |
|----------|---------|---------|
| `SPARKY_FITNESS_DB_NAME` | `sparkyfitness_db` | Database name created on init |
| `SPARKY_FITNESS_DB_USER` | `sparky` | **Superuser** for DB init and migrations |
| `SPARKY_FITNESS_DB_PASSWORD` | *(secret)* | Superuser password. Generate with `openssl rand -hex 32` |
| `SPARKY_FITNESS_DB_HOST` | `sparkyfitness-db` | DB hostname — service name on `sparkyfitness_net` |
| `SPARKY_FITNESS_APP_DB_USER` | `sparky_app` | Runtime application user (limited privileges, created on first boot) |
| `SPARKY_FITNESS_APP_DB_PASSWORD` | *(secret)* | App user password — can be rotated post-init |

#### Auth & encryption — **DO NOT CHANGE POST-INIT**

| Variable | Example | Purpose |
|----------|---------|---------|
| `SPARKY_FITNESS_API_ENCRYPTION_KEY` | *(64-char hex)* | Encrypts external-provider API credentials at rest. Generate with `openssl rand -hex 32`. **Changing this invalidates all stored external-provider creds.** |
| `BETTER_AUTH_SECRET` | *(random hex)* | Signs sessions and encrypts TOTP secrets. Generate with `openssl rand -hex 32`. **Changing this locks out every user with 2FA enabled.** |

> Back up `.env` immediately after first boot. Losing these two secrets means losing encrypted data and locking out 2FA users.

#### Application behavior

| Variable | Example | Purpose |
|----------|---------|---------|
| `SPARKY_FITNESS_FRONTEND_URL` | `https://sparkyfitness.example.com` | Public URL — CORS allowlist. **Must match the NPM hostname exactly** |
| `SPARKY_FITNESS_DISABLE_SIGNUP` | `false` → `true` | Blocks new registrations. Start `false` to register the first account, then flip to `true` |
| `SPARKY_FITNESS_FORCE_EMAIL_LOGIN` | `true` | Fail-safe — keeps email/password login working even if OIDC is later misconfigured |
| `SPARKY_FITNESS_LOG_LEVEL` | `ERROR` | Server logging verbosity |
| `ALLOW_PRIVATE_NETWORK_CORS` | `false` | Allows CORS from RFC1918 ranges. Leave `false` — the NPM hostname handles access |

#### Email (optional — for password resets and notifications)

| Variable | Example | Purpose |
|----------|---------|---------|
| `SPARKY_FITNESS_EMAIL_HOST` | `smtp.example.com` | SMTP server hostname (blank = email disabled) |
| `SPARKY_FITNESS_EMAIL_PORT` | `587` | SMTP port |
| `SPARKY_FITNESS_EMAIL_SECURE` | `false` | `true` for TLS/SSL, `false` for STARTTLS/plaintext |
| `SPARKY_FITNESS_EMAIL_USER` | `user@example.com` | SMTP auth username |
| `SPARKY_FITNESS_EMAIL_PASS` | *(secret)* | SMTP auth password / app password |
| `SPARKY_FITNESS_EMAIL_FROM` | `no-reply@example.com` | `From:` header for outbound mail |

> All six are defined blank in `.env.example` so compose doesn't warn about unset variables. Fill them in to enable outbound email; leave blank to disable it.

### Hardcoded in `compose.yaml`

| Variable | Value | Purpose |
|----------|-------|---------|
| `SPARKY_FITNESS_SERVER_HOST` | `sparkyfitness-server` | Internal hostname the frontend's nginx proxies `/api` to |
| `SPARKY_FITNESS_SERVER_PORT` | `3010` | Backend listen port on `sparkyfitness_net` |
| `SPARKY_FITNESS_DB_PORT` | `5432` | DB port on `sparkyfitness_net` (mcp service only) |
| `MCP_TRANSPORT` | `http` | MCP server transport mode. Required for HTTP clients like Claude Desktop |

---

## Healthchecks

| Service | Status | Notes |
|---------|--------|-------|
| `sparkyfitness-db` | (image default) | Postgres image has no built-in healthcheck — relies on TCP reachability |
| `sparkyfitness-server` | (image default) | Healthy on `/health` endpoint |
| `sparkyfitness-frontend` | (image default) | Healthy on Nginx response |
| `sparkyfitness-mcp` | **overridden in compose** | Upstream Dockerfile probes wrong port (`3011` vs actual `3001`). See override below |

The MCP service overrides the broken upstream healthcheck:

```yaml
healthcheck:
  test: ["CMD", "node", "-e", "require('http').get('http://127.0.0.1:3001/health', (r) => process.exit(r.statusCode === 200 ? 0 : 1)).on('error', () => process.exit(1))"]
  interval: 30s
  timeout: 10s
  retries: 5
  start_period: 15s
```

> Same pattern as the SearXNG healthcheck override — fix in compose rather than depending on upstream. Worth filing the typo upstream: the Dockerfile's `HEALTHCHECK` references port `3011` but the server listens on `3001`.

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme |
|----------|---------------|---------------|--------|
| `sparkyfitness.example.com` | `sparkyfitness-frontend` | `80` | `http` |

> **`SPARKY_FITNESS_FRONTEND_URL` must match this hostname exactly.** Mismatched values fail CORS preflight and reject every API request from the SPA.
>
> Enable **Websockets Support** on the NPM proxy host — Better Auth uses websockets for some session flows.
>
> Block Common Exploits + Force SSL + HSTS recommended.

The MCP service (loopback `127.0.0.1:3001`) is intentionally **not** proxied through NPM. Access it locally or over a VPN/tailnet — it's an authenticated API into your full health dataset and the AI/MCP feature is still marked beta upstream.

---

## Security Posture

Stock upstream images run as their built-in non-root users; this stack does not customize caps further. Two pieces of advice from upstream worth re-emphasizing:

- Upstream explicitly warns: *"Do NOT expose this app directly to the internet. It is still under active development, and not production-hardened."* Honor that — keep the frontend on the LAN behind a reverse proxy, with a VPN/tailnet for remote access, rather than public port-forwarding.
- `SPARKY_FITNESS_FORCE_EMAIL_LOGIN=true` is a deliberate fail-safe — if OIDC is later configured and breaks, email/password login still works. Don't disable it without confirming OIDC end-to-end.

---

## Deployment

### Prerequisites

- `proxy_net` external network (already exists if NPM is running)
- `sparkyfitness_net` internal network created automatically by compose
- Generated values for the four secrets (`SPARKY_FITNESS_DB_PASSWORD`, `SPARKY_FITNESS_APP_DB_PASSWORD`, `SPARKY_FITNESS_API_ENCRYPTION_KEY`, `BETTER_AUTH_SECRET`)

### Bring up

```bash
cd /opt/stacks/sparkyfitness
cp .env.example .env

# Generate the secrets (fills the blank values in place)
sed -i "s|SPARKY_FITNESS_DB_PASSWORD=.*|SPARKY_FITNESS_DB_PASSWORD=$(openssl rand -hex 32)|" .env
sed -i "s|SPARKY_FITNESS_APP_DB_PASSWORD=.*|SPARKY_FITNESS_APP_DB_PASSWORD=$(openssl rand -hex 32)|" .env
sed -i "s|SPARKY_FITNESS_API_ENCRYPTION_KEY=.*|SPARKY_FITNESS_API_ENCRYPTION_KEY=$(openssl rand -hex 32)|" .env
sed -i "s|BETTER_AUTH_SECRET=.*|BETTER_AUTH_SECRET=$(openssl rand -hex 32)|" .env

nano .env  # set SPARKY_FITNESS_FRONTEND_URL, TZ, confirm paths
chmod 600 .env

# Pre-create bind-mount dirs (match your DB_PATH / SERVER_* paths)
sudo mkdir -p /path/to/sparkyfitness/{postgresql,backup,uploads}

docker compose up -d
docker compose logs -f sparkyfitness-server  # watch migrations apply
```

### First-run setup

1. Wait for `sparkyfitness-server` to finish migrations and report healthy (~30s on first boot).
2. Open `https://sparkyfitness.example.com` (or `http://<host-ip>:3004` directly) — the registration form will be visible.
3. Sign up with the email you want as the admin account.
4. *(Optional)* To auto-grant admin rights, set `SPARKY_FITNESS_ADMIN_EMAIL=you@example.com` in `.env` **before** first signup and recreate the server.
5. Lock down signups:
   ```bash
   sed -i 's|SPARKY_FITNESS_DISABLE_SIGNUP=false|SPARKY_FITNESS_DISABLE_SIGNUP=true|' .env
   docker compose up -d sparkyfitness-server
   ```
6. *(Optional)* Configure an MCP client to point at `http://<tailnet-ip>:3001` for AI access.

---

## Backup

Paths to include in Kopia / Borgmatic snapshots:

- `${DB_PATH}` — PostgreSQL data directory
- `${SERVER_UPLOADS_PATH}` — user uploads, profile pics, vision-AI inputs
- `${SERVER_BACKUP_PATH}` — server-generated `pg_dump` output if `db_backup.sh` is being run
- `.env` — contains both unrecoverable secrets and DB passwords

> **Live-snapshot caveat.** Kopia backing up `${DB_PATH}` while Postgres is writing risks a torn snapshot. For a clean backup, either:
> - Run upstream's `db_backup.sh` on a schedule into `${SERVER_BACKUP_PATH}` (logical dump, always consistent), and let Kopia grab that, or
> - Accept the small inconsistency risk on a personal homelab — Postgres usually recovers cleanly from a crash-consistent snapshot on next start.

---

## Upgrades

Upstream warns against blanket auto-updates and tags releases as `vX.Y.Z.W` (four-segment, with `v` prefix) which doesn't fit standard semver regex. This stack pins images to `:latest` with `wud.watch=true` and `wud.trigger.docker.enable=false` — WUD notifies on new digests, pulls are manual.

Before pulling a new version:

1. Read the [release notes](https://github.com/CodeWithCJ/SparkyFitness/releases) — breaking changes are common.
2. Take a fresh DB backup: `docker compose exec sparkyfitness-server /app/SparkyFitnessServer/db_backup.sh` or via Kopia.
3. `docker compose pull && docker compose up -d`.
4. Watch `docker compose logs -f sparkyfitness-server` for migration output on first boot of the new version.

Postgres upgrades follow [upstream's Postgres upgrade guide](https://codewithcj.github.io/SparkyFitness/install/postgres-upgrade) — this stack is already on 18.3-alpine, ahead of the upstream sample compose's 15-alpine baseline.

---

## References

- [SparkyFitness docs site](https://codewithcj.github.io/SparkyFitness/)
- [Docker Compose install guide](https://codewithcj.github.io/SparkyFitness/install/docker-compose/)
- [Environment variables reference](https://codewithcj.github.io/SparkyFitness/install/environment-variables)
- [OIDC authentication setup](https://codewithcj.github.io/SparkyFitness/administration/oauth-authentication)
- [Postgres upgrade guide](https://codewithcj.github.io/SparkyFitness/install/postgres-upgrade)
- [MCP server feature](https://codewithcj.github.io/SparkyFitness/features/mcp-server)
- [Top-level README](../../README.md)
