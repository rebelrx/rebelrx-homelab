# seafile

Self-hosted file sync and share platform with [Seafile](https://www.seafile.com/) — Nextcloud alternative built around content-addressed block storage. File data lives on a NAS over NFS; the database lives on local SSD. Public access is brokered through a dedicated edge VPS over Tailscale; the home machine never opens a port to the internet.

---

## Architecture

```
Internet → Caddy (VPS, public IP) → Tailscale → seafile container (home server, tailnet IP only)
```

| Layer | Where | Role |
|-------|-------|------|
| TLS + reverse proxy | A VPS running Caddy (e.g. DigitalOcean) | Terminates HTTPS, proxies to the home server over the tailnet |
| Transport | Tailscale | Encrypted mesh between VPS and home server — no exposed ports on the home network |
| Application | Seafile stack on the home server | Bound to the home server's Tailscale interface only (`tailscale0`), unreachable from LAN or public internet |
| DNS | Your DNS provider (DNS-only, no proxy) | An A record points directly at the VPS public IP — traffic is end-to-end between user and VPS |

> This is the same public-edge pattern as the `documenso` stack — a VPS bastion fronting a Tailscale-bound home service. If you don't need public access, see [Ports](#ports) for simpler local bindings.

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `seafile` | `seafileltd/seafile-mc:13.0-latest` | All-in-one Seafile server (seaf-server, seahub, internal nginx) |
| `seafile-mysql` | `mariadb:10.11` | MySQL backend (ccnet, seafile, seahub schemas) |
| `seafile-redis` | `redis:7-alpine` | Cache server (replaces memcached as of Seafile 13) |

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Available for optional local reverse-proxy access (e.g. an internal-only NPM hostname); not used by the public access path |
| `seafile-net` | Internal bridge | App ↔ MariaDB ↔ Redis |

> The public access path does **not** use `proxy_net` — Caddy on the VPS reaches Seafile via the published host port over Tailscale. `proxy_net` membership is retained for optional local NPM access (e.g. an internal hostname like `seafile.lan.example.com`) if ever desired.

---

## Ports

| Service | Host binding | Container | Notes |
|---------|--------------|-----------|-------|
| `seafile` | `${TAILSCALE_IP}:8088` | `80` | Bound to the home server's Tailscale interface only. Not reachable from LAN, not reachable publicly, only reachable from tailnet members. Caddy on the VPS proxies to here. |

DB and Redis are not published.

> **Why bind to the Tailscale IP and not `127.0.0.1`?** Loopback is per-machine. The reverse proxy (Caddy) lives on a different machine (the VPS), so it can't reach `127.0.0.1` on the home server. Binding to the Tailscale IP exposes the port only to tailnet members — strictly tighter than `0.0.0.0` (which would also expose it on LAN) and reachable from the VPS, which is what we need.
>
> **Not using a VPS bastion?** This binding is specific to the Caddy-on-VPS-over-Tailscale pattern. For a plain local setup, edit the mapping in `compose.yaml`: use `8088:80` to publish on all interfaces (reachable on your LAN), or `127.0.0.1:8088:80` for loopback-only behind a local reverse proxy. Then drop `TAILSCALE_IP` from `.env` and skip the `net.ipv4.ip_nonlocal_bind` step below.

---

## Volumes

| Host path (`.env` var) | Container path | Mode | Purpose |
|------------------------|----------------|------|---------|
| `${SEAFILE_DATA_PATH}` | `/shared` | rw | All Seafile data: config, logs, and block storage (NFS to the NAS) |
| `${SEAFILE_MYSQL_PATH}` | `/var/lib/mysql` | rw | MariaDB data directory (local SSD) |

> Seafile's `/shared` lives entirely on NFS, including config and logs. This matches the official compose layout. If log latency becomes a concern under load, set `SEAFILE_LOG_TO_STDOUT=true` to route logs to Docker's logging driver instead of the NFS-mounted files.

---

## Environment Variables

### `.env` (Compose interpolation)

#### Server identity

| Variable | Example | Purpose |
|----------|---------|---------|
| `SEAFILE_SERVER_HOSTNAME` | `seafile.example.com` | Public hostname — Seafile builds absolute URLs from this; must match the public Caddy host on the VPS |
| `SEAFILE_SERVER_PROTOCOL` | `https` | Scheme Seafile uses when generating absolute URLs |
| `TIME_ZONE` | `America/New_York` | Container timezone (used in logs and scheduled tasks) |
| `TAILSCALE_IP` | `100.x.y.z` | The home server's Tailscale IP. Used by compose to bind the published port to the `tailscale0` interface. Get with `tailscale ip -4`. |

#### Secrets

| Variable | Purpose |
|----------|---------|
| `JWT_PRIVATE_KEY` | Internal service auth between seaf-server and seahub. ≥32 characters. Generate with `openssl rand -hex 32` |
| `INIT_SEAFILE_MYSQL_ROOT_PASSWORD` | MariaDB `root` password — only consumed on first boot |
| `SEAFILE_MYSQL_DB_PASSWORD` | Password for the `seafile` MySQL user (consumed on every boot) |
| `REDIS_PASSWORD` | Redis AUTH password |
| `INIT_SEAFILE_ADMIN_PASSWORD` | First admin account password — only consumed on first boot |

#### Initial admin (first boot only)

| Variable | Example | Purpose |
|----------|---------|---------|
| `INIT_SEAFILE_ADMIN_EMAIL` | `admin@example.com` | First admin account email |
| `INIT_SEAFILE_ADMIN_PASSWORD` | *(secret)* | First admin account password |

> The `INIT_*` variables only apply on first boot. Changing them later does nothing — rotate the admin password via the Seafile UI, or use `docker exec -it seafile /opt/seafile/seafile-server-latest/reset-admin.sh`.

#### Volumes

| Variable | Example | Purpose |
|----------|---------|---------|
| `SEAFILE_DATA_PATH` | `/path/to/nas-data/seafile` | Host path for `/shared` (NAS mount) |
| `SEAFILE_MYSQL_PATH` | `/path/to/seafile/mysql` | Host path for the MariaDB data directory (local SSD) |

---

## Public Access — VPS Bastion (Caddy)

Public HTTPS is served by Caddy on a dedicated VPS (the public edge). The VPS terminates TLS with Let's Encrypt and proxies to this Seafile stack over the tailnet.

### Caddy site block (on the VPS, `/etc/caddy/Caddyfile`)

```caddy
seafile.example.com {
    encode zstd gzip

    request_body {
        max_size 10GB
    }

    reverse_proxy <home-server-tailscale-ip>:8088 {
        header_up X-Real-IP {remote_host}
    }

    log {
        output file /var/log/caddy/seafile.log {
            roll_size 50mb
            roll_keep 10
        }
        format json
    }
}
```

### DNS

| Record | Type | Value | Proxy |
|--------|------|-------|-------|
| `seafile.example.com` | A | VPS public IP | **DNS only** (no proxy) |

> If your DNS provider offers a proxy/CDN (e.g. Cloudflare's orange cloud), keep it **off** for this hostname. Proxying would terminate TLS at the provider's edge, defeating the privacy goal of self-hosting. The VPS public IP being exposed is acceptable because the VPS is the designated public edge — hardened with a host firewall (only 80/443 + tailnet), key-only SSH, fail2ban, and unattended-upgrades.

### Tailscale interface binding requirement

Because Docker binds host ports at container start, `tailscale0` must exist when Docker starts the container — otherwise the bind fails after a reboot. Set on the home server:

```bash
echo 'net.ipv4.ip_nonlocal_bind=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

This allows Docker to bind to an IP that doesn't exist yet; when Tailscale comes up shortly after, the binding starts serving.

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing)
- `seafile-net` internal network is created automatically by compose
- NFS share for `${SEAFILE_DATA_PATH}` exists, is mounted, and is writable by root (Seafile runs as root with `NON_ROOT=false`)
- Local data directory exists (match your `${SEAFILE_MYSQL_PATH}`): `mkdir -p /path/to/seafile/mysql`
- Tailscale running on the home server; get the IP with `tailscale ip -4`
- `net.ipv4.ip_nonlocal_bind=1` applied (see above)
- Generated values for `JWT_PRIVATE_KEY`, `INIT_SEAFILE_MYSQL_ROOT_PASSWORD`, `SEAFILE_MYSQL_DB_PASSWORD`, `REDIS_PASSWORD`, `INIT_SEAFILE_ADMIN_PASSWORD`

### Bring up

```bash
cd /opt/stacks/seafile

# Copy the example env file
cp .env.example .env
chmod 600 .env

# Fill in the blank secrets (edits the values in place, no duplicate keys)
sed -i "s|JWT_PRIVATE_KEY=.*|JWT_PRIVATE_KEY=$(openssl rand -hex 32)|" .env
sed -i "s|INIT_SEAFILE_MYSQL_ROOT_PASSWORD=.*|INIT_SEAFILE_MYSQL_ROOT_PASSWORD=$(openssl rand -base64 32 | tr -d '\n')|" .env
sed -i "s|SEAFILE_MYSQL_DB_PASSWORD=.*|SEAFILE_MYSQL_DB_PASSWORD=$(openssl rand -base64 32 | tr -d '\n')|" .env
sed -i "s|REDIS_PASSWORD=.*|REDIS_PASSWORD=$(openssl rand -base64 32 | tr -d '\n')|" .env
sed -i "s|INIT_SEAFILE_ADMIN_PASSWORD=.*|INIT_SEAFILE_ADMIN_PASSWORD=$(openssl rand -base64 24 | tr -d '\n')|" .env

nano .env  # set TAILSCALE_IP, hostname, admin email, timezone, NAS paths

docker compose up -d
docker compose logs -f seafile
```

### First-run setup

1. Watch the seafile container logs for the full init sequence — takes 1–3 minutes. You want to see all four `Now creating ... database tables ...` blocks, followed by `Successfully created seafile admin` and `Seahub is started`.
2. Container goes healthy within ~60s of the init completing.
3. Verify the bind from the home server: `curl -I http://${TAILSCALE_IP}:8088` — expect a `302` redirect to the login page.
4. Verify the bind is reachable from the VPS: SSH into the VPS, run `curl -I http://<home-server-tailscale-ip>:8088` — same `302` expected.
5. Open `https://seafile.example.com` and log in with `INIT_SEAFILE_ADMIN_EMAIL` / `INIT_SEAFILE_ADMIN_PASSWORD`.

---

## Backup

Critical paths to include in Kopia / Borgmatic snapshots:

- `${SEAFILE_DATA_PATH}` — Seafile config, logs, **and block storage**. The blocks are the actual user file contents; without them the DB is just metadata pointing at nothing.
- `${SEAFILE_MYSQL_PATH}` — MariaDB data (users, libraries, file metadata, share links, history)
- `.env` — needed to bring the stack back up with the same DB and Redis passwords

> A DB-only backup is **not sufficient** — Seafile's content-addressed blocks under `${SEAFILE_DATA_PATH}/seafile/seafile-data/storage/` must be backed up alongside the DB or the restore will be useless.
>
> For MariaDB specifically, a periodic `mysqldump` is safer than relying solely on file-level snapshots:
>
> ```bash
> docker compose exec -T db sh -c \
>   'mysqldump -uroot -p"$MYSQL_ROOT_PASSWORD" --all-databases' \
>   | gzip > /path/to/seafile/dumps/seafile-$(date +%F).sql.gz
> ```

---

## Maintenance

### Garbage collection

Deleted file blocks aren't reclaimed automatically. Run periodically (safe with the server up — online GC mode in 13.x):

```bash
docker exec seafile /scripts/gc.sh
```

### Admin password reset

```bash
docker exec -it seafile /opt/seafile/seafile-server-latest/reset-admin.sh
```

### Enter the container

```bash
docker exec -it seafile /bin/bash
```

### Find logs

Inside the container: `/shared/logs/` (system) and `/shared/seafile/logs/` (Seafile services).
From the host: under `${SEAFILE_DATA_PATH}/logs/` and `${SEAFILE_DATA_PATH}/seafile/logs/`.

```bash
sudo tail -f $(find /path/to/nas-data/seafile/ -type f -name '*.log' 2>/dev/null)
```

---

## Notes & Gotchas

- **First-boot directory trap.** Seafile's init script checks for an existing `/shared/seafile/seafile-data` directory and skips initialization if found. If you bind-mount a subdirectory of `/shared` (e.g. trying to put just block storage on NAS while keeping config local), Docker pre-creates the parent path and trips this check — nginx starts, but `seaf-server` and `seahub` never do, and the container reports unhealthy with a 502. **Fix:** mount `/shared` as a single volume (current layout) rather than splitting subpaths.
- **Tailscale IP binding + boot order.** Without `net.ipv4.ip_nonlocal_bind=1`, Docker will fail to bind the published port on boot if it starts before Tailscale establishes `tailscale0`. The sysctl setting lets Docker bind to a not-yet-existing IP; the bind starts working as soon as Tailscale is up. (Not relevant if you switched to an all-interfaces / loopback binding — see [Ports](#ports).)
- **Tailscale IP changes.** Tailscale IPs are sticky but not guaranteed permanent (e.g. after node deletion + re-add). If the home server's Tailscale IP changes, update `TAILSCALE_IP` in `.env`, recreate the container, and update the `reverse_proxy` target in the VPS's Caddyfile.
- **NAS root squashing.** Seafile runs as root with `NON_ROOT=false`. If the NAS NFS export squashes root, init will fail with permission errors writing config files. Confirm the export is configured permissively (or use `NON_ROOT=true` with the uid/gid pre-setup steps from the [official docs](https://manual.seafile.com/latest/setup/run_seafile_as_non_root_user_inside_docker/)).
- **CSRF errors on first login.** If login returns a CSRF error, verify `${SEAFILE_DATA_PATH}/seafile/conf/seahub_settings.py` contains `CSRF_TRUSTED_ORIGINS = ['https://seafile.example.com']`. Setting `SEAFILE_SERVER_HOSTNAME` + `SEAFILE_SERVER_PROTOCOL=https` should populate this automatically on first boot — manual edit + container restart is the fallback.
- **Notification server, SeaDoc, AI, face recognition** are all disabled in this deployment. Enable later by flipping the corresponding `ENABLE_*` env var (some require adding additional services from the official `seafile-server.yml` — notification server in particular needs its own container).
- **NFS performance.** Block reads/writes go over NFS. For typical document/photo sync this is fine; for workloads with many small files, expect slower performance than local SSD. Logs also go to NFS — flip `SEAFILE_LOG_TO_STDOUT=true` if log volume becomes a bottleneck.
- **Major version upgrades.** WUD pins the major version (`13.0-latest`); point releases float. Seafile major version upgrades historically require manual schema migration — read the upgrade notes before bumping from 13.x to 14.x.

---

## References

- [Seafile CE deployment with Docker](https://manual.seafile.com/latest/setup/setup_ce_by_docker/)
- [Seafile environment variables reference](https://manual.seafile.com/latest/config/env/)
- [Seafile admin manual (GitHub)](https://github.com/haiwen/seafile-admin-docs)
- [Top-level README](../../README.md)
