# forgejo

Self-hosted Git server with [Forgejo](https://forgejo.org/) — community fork of Gitea. Includes CI runners with a dedicated Docker-in-Docker daemon.

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `forgejo` | `data.forgejo.org/forgejo/forgejo:15` | Forgejo web UI, API, and Git server |
| `forgejo-db` | `postgres:14` | PostgreSQL database |
| `forgejo-dind` | `docker:dind` | Docker daemon backing the CI runner (privileged) |
| `forgejo-runner` | `data.forgejo.org/forgejo/runner:12` | Forgejo Actions runner, executes CI jobs via DinD |

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse-proxy access via NPM (server only) |
| `forgejo` | Internal (default) | Server ↔ PostgreSQL ↔ Runner ↔ DinD |

> Only `forgejo` joins `proxy_net`. The DB, runner, and DinD daemon are reachable only on the internal `forgejo` network.

---

## Ports

| Host port (`.env` var) | Container port | Bind | Purpose |
|------------------------|----------------|------|---------|
| `${FORGEJO_HTTP_PORT}` (`3000`) | `3000` | all interfaces | Web UI + git-over-HTTPS |

Only the `forgejo` server publishes a port; the DB, runner, and DinD stay on the internal network. Published on all interfaces so it's reachable on your LAN out of the box. To keep it behind a reverse proxy / loopback only, prefix the mapping in `compose.yaml` with `127.0.0.1:`. `3000` is a common port — change `FORGEJO_HTTP_PORT` if it clashes on your host.

> **Git over SSH is not available** — port `22` is not published from the Forgejo container. All git operations go over HTTPS (`https://forgejo.example.com/<user>/<repo>.git`, or `http://<host-ip>:${FORGEJO_HTTP_PORT}/...` directly). To enable SSH cloning, publish a host port mapped to the container's `22` and configure `[server] START_SSH_SERVER` / `SSH_PORT` in `app.ini`.

---

## Volumes

| Host path (`.env` var) | Container path | Used by | Purpose |
|------------------------|----------------|---------|---------|
| `${FORGEJO_DATA}` | `/data` | forgejo | Repositories, LFS storage, attachments, `app.ini` |
| `${FORGEJO_DB_DATA}` | `/var/lib/postgresql/data` | forgejo-db | PostgreSQL data directory |
| `${FORGEJO_RUNNER_DATA}` | `/data` | forgejo-runner | Runner registration token, `runner-config.yml`, job cache |

> The runner expects `${FORGEJO_RUNNER_DATA}` to be owned by **UID/GID 1001** on the host (the runner image runs as `1001:1001`). Set this before first start:
>
> ```bash
> sudo mkdir -p /path/to/forgejo-runner/data
> sudo chown -R 1001:1001 /path/to/forgejo-runner/data
> ```

---

## Environment Variables

### `.env` (Compose interpolation)

| Variable | Example | Purpose |
|----------|---------|---------|
| `USER_UID` | `1000` | UID Forgejo runs as inside the container (write owner for `${FORGEJO_DATA}`) |
| `USER_GID` | `1000` | GID Forgejo runs as inside the container |
| `TIMEZONE` | `America/New_York` | Container TZ |
| `FORGEJO_DB_PASSWORD` | *(secret)* | PostgreSQL password — used by both `forgejo` and `forgejo-db`. Generate with `openssl rand -base64 32` |
| `FORGEJO_HTTP_PORT` | `3000` | Host port published for the web UI + git-over-HTTPS |
| `FORGEJO_DATA` | `/path/to/forgejo/data` | Host bind for Forgejo data |
| `FORGEJO_DB_DATA` | `/path/to/forgejo/db` | Host bind for Postgres data |
| `FORGEJO_RUNNER_DATA` | `/path/to/forgejo-runner/data` | Host bind for runner state |

> Other Forgejo settings (`FORGEJO__*` env vars for DB connection) are set directly in `compose.yaml` rather than `.env`. Everything else lives in `app.ini` inside `${FORGEJO_DATA}/gitea/conf/app.ini`, generated on first run.

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme |
|----------|---------------|---------------|--------|
| `forgejo.example.com` | `forgejo` | `3000` | `http` |

> Forgejo serves both the web UI and git-over-HTTPS at the same hostname. Make sure NPM's `client_max_body_size` is set high enough for any LFS / large repo pushes (the default 1 MB will reject most pushes). A value like `5G` is reasonable.

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing)
- `forgejo` internal network is created automatically by compose
- `${FORGEJO_RUNNER_DATA}` exists on the host and is owned by `1001:1001`
- A generated value for `FORGEJO_DB_PASSWORD`

### Bring up

```bash
cd /opt/stacks/forgejo
cp .env.example .env

# Set DB password
sed -i "s|FORGEJO_DB_PASSWORD=.*|FORGEJO_DB_PASSWORD=$(openssl rand -base64 32 | tr -d '\n')|" .env

# Pre-create runner data dir with correct ownership (match ${FORGEJO_RUNNER_DATA})
sudo mkdir -p /path/to/forgejo-runner/data
sudo chown -R 1001:1001 /path/to/forgejo-runner/data

chmod 600 .env

docker compose up -d forgejo db
```

The runner is brought up separately after Forgejo is registered — see below.

### First-run setup — Forgejo

1. Wait for Forgejo to become healthy (`docker compose ps forgejo` should show `healthy`).
2. Open the app (e.g. `https://forgejo.example.com` via NPM, or `http://<host-ip>:${FORGEJO_HTTP_PORT}` directly) — the install wizard appears on first visit.
3. **Critical install-wizard settings:**
   - Database type: `PostgreSQL` (already pre-filled from env)
   - Server domain: your public hostname (e.g. `forgejo.example.com`)
   - Forgejo base URL: the matching URL (e.g. `https://forgejo.example.com/`)
   - Disable open registration unless you want public sign-ups
4. Create the **first user** at the bottom of the install page — this user becomes the site admin.

### First-run setup — Runner registration

1. In Forgejo, navigate to **Site Administration → Actions → Runners** and generate a registration token.
2. Register the runner:

   ```bash
   docker compose run --rm forgejo-runner \
     forgejo-runner register --no-interactive \
       --instance https://forgejo.example.com \
       --token <REGISTRATION_TOKEN> \
       --name homelab-runner \
       --labels docker:docker://node:lts,ubuntu:docker://ubuntu:22.04
   ```

   This creates `runner-config.yml` and `.runner` inside `${FORGEJO_RUNNER_DATA}`.

3. Start the runner:

   ```bash
   docker compose up -d forgejo-runner docker-in-docker
   ```

4. Confirm the runner shows as `Online` under **Site Administration → Actions → Runners**.

---

## Backup

Critical paths to include in Kopia / Borgmatic snapshots:

- `${FORGEJO_DB_DATA}` — PostgreSQL data (users, issues, PRs, all metadata)
- `${FORGEJO_DATA}` — repositories, LFS objects, attachments, `app.ini`
- `${FORGEJO_RUNNER_DATA}` — runner registration (`.runner` file). Without this, you'd need to re-register the runner after a restore
- `.env` — contains `FORGEJO_DB_PASSWORD`

> Forgejo also has a built-in admin backup command that bundles everything into a single archive:
>
> ```bash
> docker compose exec forgejo \
>   su git -c 'forgejo dump --type tar.gz --file /data/dump.tar.gz'
> ```
>
> Useful for occasional snapshots ahead of major-version upgrades, alongside the regular file-level backups.

---

## Notes & Gotchas

- **`forgejo-dind` runs privileged.** The DinD daemon backs the runner's job execution. It's scoped to the internal `forgejo` network (no exposure to `proxy_net` or the host network), but anything that compromises the DinD container has root on the host. This is a deliberate tradeoff — running CI without DinD usually means either binding the host docker socket (worse blast radius) or running rootless CI (less compatible). DinD is **intentionally not WUD-watched** and is a deliberate exception to the `no-new-privileges` default — it's an infrastructure daemon, not a tracked app.
- **CI uses Quad9 DNS.** `forgejo-dind` is configured with `--dns 9.9.9.9 --dns 149.112.112.112` rather than the host's resolver. This bypasses any DNS filtering (e.g. AdGuard) for runner-spawned containers — without it, CI jobs that pull from image registries or external HTTP sources could be silently blocked. If you block specific domains that CI legitimately needs, no compose change is required.
- **Runner data ownership.** The runner image runs as `1001:1001`. If `${FORGEJO_RUNNER_DATA}` is owned by `root` or `1000:1000` (the default for everything else here), the runner won't be able to write its config and will crash-loop with permission errors. Pre-create the directory with the right ownership before starting.
- **PostgreSQL pinned to 14.** Forgejo supports PostgreSQL 12+. The DB image is pinned to `postgres:14`, but the WUD labels don't yet constrain tag matching with `wud.tag.include=^14`, so WUD will suggest 15.x and 16.x upgrades — a major Postgres jump requires a `pg_dump`/restore migration, not a straight image bump. Ignore those notifications until the constraint is in place or you deliberately migrate.
- **Git over SSH is unavailable.** SSH (port 22) is not published, so the SSH clone URL Forgejo displays in the UI won't work. All git operations go over HTTPS using a personal access token. Setting `[server] DISABLE_SSH = true` in `app.ini` hides the SSH clone option from the UI.

---

## References

- [Forgejo docs — installation](https://forgejo.org/docs/next/admin/installation/)
- [Forgejo Actions docs](https://forgejo.org/docs/next/user/actions/)
- [Forgejo Runner docs](https://forgejo.org/docs/next/admin/runner-installation/)
- [Top-level README](../../README.md)
