# authentik

Self-hosted SSO, identity provider, and application gateway with [Authentik](https://goauthentik.io/).

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `authentik-server` | `ghcr.io/goauthentik/server:${AUTHENTIK_TAG}` | Authentik web UI, API, and authentication endpoints |
| `authentik-worker` | `ghcr.io/goauthentik/server:${AUTHENTIK_TAG}` | Background tasks, embedded outpost management, certificate handling |
| `authentik-db` | `docker.io/library/postgres:16-alpine` | PostgreSQL database |

> Server and worker share the same image and **must** run the same version, or the worker will refuse to start. Both reference `${AUTHENTIK_TAG}` so a single `.env` edit updates both at once.

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse-proxy access via NPM (server only) |
| `authentik` | Internal (default) | Server ↔ Worker ↔ PostgreSQL |

> Only `authentik-server` joins `proxy_net`. The worker and DB are reachable only on the internal network.

---

## Ports

| Host port (`.env` var) | Container port | Bind | Purpose |
|------------------------|----------------|------|---------|
| `${COMPOSE_PORT_HTTP}` (`9000`) | `9000` | all interfaces | HTTP listener — web UI, API, auth endpoints |
| `${COMPOSE_PORT_HTTPS}` (`9443`) | `9443` | all interfaces | HTTPS listener |

Only `authentik-server` publishes ports; the worker and DB stay on the internal `authentik` network. Published on all interfaces so the UI is reachable on your LAN out of the box. To keep it behind a reverse proxy / loopback only, prefix the mappings in `compose.yaml` with `127.0.0.1:`.

> `9000` is a common default (Portainer uses it too) — change `COMPOSE_PORT_HTTP` / `COMPOSE_PORT_HTTPS` if they clash with another service on the host.

---

## Volumes

| Host path (`.env` var) | Container path | Used by | Purpose |
|------------------------|----------------|---------|---------|
| `${AUTHENTIK_DATA}` | `/data` | server, worker | Application data, media uploads, blueprints |
| `${AUTHENTIK_TEMPLATES}` | `/templates` | server, worker | Custom email templates |
| `${AUTHENTIK_CERTS}` | `/certs` | worker | Certificate storage for outposts |
| `${AUTHENTIK_DATABASE}` | `/var/lib/postgresql/data` | db | PostgreSQL data directory |

---

## Environment Variables

### `.env` (Compose interpolation + container env via `env_file`)

| Variable | Example | Purpose |
|----------|---------|---------|
| `AUTHENTIK_TAG` | `2026.2.3` | Image tag for both server and worker. Pin to a known-good version; bump after reading release notes |
| `PG_DB` | `authentik` | Postgres database name |
| `PG_USER` | `authentik` | Postgres username |
| `PG_PASS` | *(secret)* | Postgres password — required, compose refuses to start if empty. Generate with `openssl rand -base64 32` |
| `AUTHENTIK_SECRET_KEY` | *(secret)* | Authentik signing key — required, compose refuses to start if empty. Generate with `openssl rand -base64 60` |
| `AUTHENTIK_ERROR_REPORTING__ENABLED` | `true` | Sentry error reporting toggle |
| `COMPOSE_PORT_HTTP` | `9000` | Host port published for the HTTP listener |
| `COMPOSE_PORT_HTTPS` | `9443` | Host port published for the HTTPS listener |
| `AUTHENTIK_DATA` | `/path/to/authentik/data` | Host bind for app data |
| `AUTHENTIK_TEMPLATES` | `/path/to/authentik/custom-templates` | Host bind for templates |
| `AUTHENTIK_CERTS` | `/path/to/authentik/certs` | Host bind for certificate storage |
| `AUTHENTIK_DATABASE` | `/path/to/authentik/postgresql/data` | Host bind for Postgres data |

> Both the server and worker use `env_file: .env` to pull secrets into the container, in addition to compose-level interpolation. Keep the `.env` file mode at `600`.

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme |
|----------|---------------|---------------|--------|
| `authentik.example.com` | `authentik-server` | `9000` | `http` |

> If you're proxying applications **through** Authentik (forward-auth or proxy provider), each protected app gets its own NPM host pointing to its respective backend, with NPM's "Custom Locations" or advanced config used to delegate `/outpost.goauthentik.io/` to the Authentik embedded outpost. See the [Authentik proxy provider docs](https://docs.goauthentik.io/docs/providers/proxy/) for the patterns.

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing)
- Generated values for `PG_PASS` and `AUTHENTIK_SECRET_KEY`

### Bring up

```bash
cd /opt/stacks/authentik
cp .env.example .env

# Generate secrets
echo "AUTHENTIK_SECRET_KEY=$(openssl rand -base64 60 | tr -d '\n')" >> .env
echo "PG_PASS=$(openssl rand -base64 32 | tr -d '\n')" >> .env

nano .env  # review, confirm AUTHENTIK_TAG is current
chmod 600 .env

docker compose up -d
```

### First-run setup

1. Wait ~60s for migrations and the initial bootstrap.
2. Open the **Initial Setup URL** at `https://authentik.example.com/if/flow/initial-setup/` — this is a one-time URL that lets you create the `akadmin` user and set its password.
3. Once the admin password is set, the URL is disabled and normal login at `https://authentik.example.com/` takes over.
4. Configure providers, applications, and outposts as needed. The default embedded outpost runs inside `authentik-server`.

---

## Updating

Authentik does NOT use `:latest`. Updates are a deliberate, version-pinned operation. Why:

- Schema migrations are sometimes one-way (no rollback by re-pulling the previous tag).
- `:latest` has shipped regressions before — notably, a recent build broke Redis initialization and refused to start.

The update workflow:

```bash
cd /opt/stacks/authentik

# 1. Take a Postgres dump (essential — one-way migrations can't be reverted)
docker compose exec -T authentik-db pg_dump -U "$(grep ^PG_USER .env | cut -d= -f2)" \
  "$(grep ^PG_DB .env | cut -d= -f2)" \
  | gzip > /path/to/authentik/dumps/pre-update-$(date +%F-%H%M).sql.gz

# 2. Read the release notes for the target version
#    https://goauthentik.io/docs/releases/

# 3. Bump AUTHENTIK_TAG in .env to the new version
nano .env

# 4. Pull and recreate
docker compose pull
docker compose up -d

# 5. Watch the logs for migration completion
docker compose logs -f authentik-server authentik-worker
```

If anything goes sideways, restore the dump and roll `AUTHENTIK_TAG` back to the prior version.

---

## Backup

Critical paths to include in Kopia / Borgmatic snapshots:

- `${AUTHENTIK_DATABASE}` — PostgreSQL data (the source of truth for users, apps, providers, etc.)
- `${AUTHENTIK_DATA}` — uploaded media, blueprints, custom assets
- `${AUTHENTIK_CERTS}` — outpost certificates
- `${AUTHENTIK_TEMPLATES}` — custom email templates (small but easy to lose)
- `.env` — contains `AUTHENTIK_SECRET_KEY` and `PG_PASS`. **A working backup of these is essential** — restoring the DB without the matching secret key won't decrypt stored credentials.

> For Postgres specifically, a `pg_dump` is safer than a hot snapshot of `${AUTHENTIK_DATABASE}`. A weekly dump alongside Kopia snapshots is a reasonable belt-and-braces approach:
>
> ```bash
> docker compose exec -T authentik-db pg_dump -U "$PG_USER" "$PG_DB" \
>   | gzip > /path/to/authentik/dumps/authentik-$(date +%F).sql.gz
> ```

---

## Notes & Gotchas

- **Worker runs as root.** Authentik's worker (`user: root`) needs root to manage embedded outpost containers and write to certificate paths. This is the upstream-recommended configuration; lowering it breaks outpost management. The server and DB do not run as root.
- **Pinned, not `:latest`.** See [Updating](#updating) for why. The TL;DR: Authentik ships occasional bad `:latest` builds (e.g. `2026.2.x → newer` broke Redis initialization), and some migrations are one-way. Pin and bump deliberately.
- **Server and worker versions must match.** Both reference `${AUTHENTIK_TAG}` to guarantee this. If you ever split the tags (e.g. to test a worker version against an older server), expect breakage.

---

## References

- [Authentik docs — Docker Compose install](https://docs.goauthentik.io/install-config/install/docker-compose/)
- [Authentik release notes](https://goauthentik.io/docs/releases/)
- [Authentik proxy provider docs](https://docs.goauthentik.io/docs/providers/proxy/)
- [Top-level README](../../README.md)
