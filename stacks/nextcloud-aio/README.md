# nextcloud-aio

Self-hosted file sync, sharing, calendar, contacts, and collaboration with [Nextcloud AIO](https://github.com/nextcloud/all-in-one) — the official "all-in-one" deployment pattern that uses a mastercontainer to orchestrate every Nextcloud component (web, DB, cache, talk, office, etc.) as separate containers.

---

## Services

This stack defines a **single service** — the AIO mastercontainer — which then spawns and manages a fleet of additional Nextcloud containers via the Docker socket.

| Service | Image | Purpose |
|---------|-------|---------|
| `nextcloud-aio-mastercontainer` | `ghcr.io/nextcloud-releases/all-in-one:latest` | Setup UI + orchestrator; launches and manages all other Nextcloud containers |

### Containers spawned by AIO (visible only after first-run setup)

| Container | Purpose |
|-----------|---------|
| `nextcloud-aio-apache` | Reverse-proxies user traffic to the Nextcloud app (listens on `${APACHE_PORT}`) |
| `nextcloud-aio-nextcloud` | The Nextcloud PHP application server |
| `nextcloud-aio-database` | PostgreSQL database |
| `nextcloud-aio-redis` | Redis cache |
| `nextcloud-aio-talk` | Nextcloud Talk signaling server (if Talk is enabled) |
| `nextcloud-aio-talk-recording` | Talk call recording (if enabled) |
| `nextcloud-aio-collabora` | Collabora Online for in-browser document editing (if enabled) |
| `nextcloud-aio-imaginary` | Image preview generator |
| `nextcloud-aio-fulltextsearch` | Search engine (if enabled) |
| `nextcloud-aio-notify-push` | Push notification service |
| `nextcloud-aio-clamav` | Antivirus scanning (if enabled) |
| `nextcloud-aio-onlyoffice` | OnlyOffice document editing (if enabled) |

> The exact set depends on which optional features you enable in the AIO setup UI.

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `nextcloud-aio` | External | Shared with spawned containers and with `npm` — used for container-to-container DNS |

> **`nextcloud-aio` must be external** so the spawned containers can join the same network as the mastercontainer, and so NPM (declared on this network in its own stack) can resolve `nextcloud-aio-apache` and `nextcloud-aio-mastercontainer` as hostnames.

> The `nextcloud-aio` stack does **not** join `proxy_net`. Instead, NPM joins `nextcloud-aio` to reach Apache and the AIO mastercontainer directly via container DNS (see [Reverse Proxy](#reverse-proxy-npm)).

---

## Ports

| Host binding | Container | Purpose |
|--------------|-----------|---------|
| `127.0.0.1:${AIO_MANAGEMENT_PORT}:8080` | Mastercontainer | Setup UI fallback access — loopback-bound for SSH-tunneled direct access if NPM is unavailable |
| `${APACHE_PORT}` (e.g. `11000`) | Spawned `nextcloud-aio-apache` | Published by AIO when it spawns Apache. Not actually required for NPM routing in this setup (NPM uses container DNS) but AIO publishes it by default |

> **The management port is deliberately loopback-bound**, unlike the all-interfaces default used by most stacks in this repo. It's a powerful admin interface (it controls the whole Nextcloud fleet), so it's kept off the LAN by design — day-to-day admin flows through NPM, and direct access uses an SSH tunnel. If you're running standalone without NPM and need LAN access to the setup wizard, either tunnel (`ssh -L 8080:127.0.0.1:${AIO_MANAGEMENT_PORT} <host>`) or, accepting the exposure, drop the `127.0.0.1:` prefix in `compose.yaml`.

---

## Volumes

| Host path / volume | Container path | Mode | Purpose |
|--------------------|----------------|------|---------|
| `nextcloud_aio_mastercontainer` (named Docker volume) | `/mnt/docker-aio-config` | rw | AIO config, secrets, spawn state — **do not delete** |
| `${NEXTCLOUD_DATADIR}` | (via `NEXTCLOUD_DATADIR` env) | rw | User data root (files, attachments) — passed to the spawned Nextcloud container by AIO |
| `/var/run/docker.sock` | `/var/run/docker.sock` | **ro** | Required: AIO spawns and manages sibling containers via the Docker API. `:ro` is the upstream-recommended default — on a Unix socket the flag only affects filesystem permissions on the socket file, not the protocol writes that go through it |
| `/mnt` | `/mnt` | rw | Host `/mnt` tree exposed to AIO so Nextcloud's external storage can mount user-visible paths inside spawned containers |

> **`nextcloud_aio_mastercontainer` is a named Docker volume**, not a bind mount. Lives under `/var/lib/docker/volumes/` on the host. Restoring this volume on a new host gets you back the mastercontainer's state (which containers exist, their configs, etc.) without needing to re-run setup.

> **The `/mnt` bind is wide.** Anything mounted under `/mnt` on the host (including your NAS mounts) is visible to AIO. This is intentional — it lets you add NAS paths as external storage in Nextcloud — but it means AIO can write anywhere under `/mnt`. Treat the AIO admin password as host-level access.

---

## Environment Variables

### `.env` (Compose interpolation)

| Variable | Example | Purpose |
|----------|---------|---------|
| `AIO_MANAGEMENT_PORT` | `8080` | Host port for the AIO setup UI (loopback-bound) |
| `APACHE_PORT` | `11000` | Port AIO publishes for the Nextcloud Apache container |
| `APACHE_IP_BINDING` | `0.0.0.0` | Interface AIO binds Apache to — `0.0.0.0` means all interfaces (required for NPM to reach it across Docker networks) |
| `NEXTCLOUD_DATADIR` | `/path/to/nas-data/nextcloud-aio/ncdata` | Host path AIO passes to the Nextcloud container as the user-data root |
| `SKIP_DOMAIN_VALIDATION` | `true` | Skip AIO's check that the public domain resolves to the host. Set `true` for Tailscale-only / split-DNS deployments where the domain isn't publicly resolvable by design |

### Hardcoded in `compose.yaml`

| Variable | Value | Purpose |
|----------|-------|---------|
| `NEXTCLOUD_MOUNT` | `/mnt` | Tells AIO that `/mnt` (visible inside the mastercontainer) is the root path for external-storage mounts |
| `NEXTCLOUD_ENABLE_NVIDIA_GPU` | `true` | Allows AIO to attach the host's NVIDIA GPU to spawned containers that benefit (e.g. for transcoding via Memories or other apps) |

---

## Reverse Proxy (NPM)

Two NPM hosts proxy this stack — one for users, one for AIO admin:

| Hostname | Upstream host | Upstream port | Scheme | Purpose |
|----------|---------------|---------------|--------|---------|
| `nextcloud.example.com` | `nextcloud-aio-apache` | `11000` | `http` | User-facing Nextcloud app |
| `nextcloud-admin.example.com` | `nextcloud-aio-mastercontainer` | `8080` | **`https`** | AIO management / setup UI |

> **The admin upstream is HTTPS, not HTTP.** The AIO mastercontainer terminates its own TLS (self-signed) on port `8080`. NPM has to use `https` as the scheme; you'll need to allow the self-signed cert by ticking "Custom certificate verification disabled" (or equivalent) in NPM's proxy host config for `nextcloud-admin`. The Apache container, by contrast, serves plain HTTP on `11000` — TLS for end users is added by NPM.

> **NPM joins the `nextcloud-aio` external network** (declared in the `npm` stack's compose) so it can resolve `nextcloud-aio-apache` and `nextcloud-aio-mastercontainer` as container hostnames. No host-IP routing involved.

> **Bump `client_max_body_size` in NPM** on the Nextcloud host to match Nextcloud's upload limit — `10G` is a sensible upper bound for personal use.
>
> **Enable WebSocket support in NPM** on both hosts — Nextcloud uses WebSockets for notifications and Talk; the AIO admin UI uses WebSockets for live container-status updates.
>
> **Forward client IP** in NPM's "Custom Nginx Configuration" on the Nextcloud host so Nextcloud sees real client IPs (required for rate limiting and audit logs):
>
> ```nginx
> proxy_set_header X-Real-IP $remote_addr;
> proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
> proxy_set_header X-Forwarded-Proto $scheme;
> ```

---

## Deployment

### Prerequisites

- `nextcloud-aio` external network (`docker network create nextcloud-aio` if missing)
- Your NAS NFS share mounted on the host (e.g. under `/mnt`) if you're pointing `NEXTCLOUD_DATADIR` at NAS-backed storage
- `${NEXTCLOUD_DATADIR}` exists on the host (AIO creates the inner structure; just ensure the parent dir is reachable)
- A generated value for the AIO setup password — created on first visit to the management UI

### Bring up

```bash
cd /opt/stacks/nextcloud-aio
cp .env.example .env
nano .env  # adjust paths and ports as needed

# Pre-create the datadir parent (match your ${NEXTCLOUD_DATADIR})
sudo mkdir -p /path/to/nas-data/nextcloud-aio/ncdata

docker compose up -d
```

### First-run setup

1. Open the AIO management UI at `https://nextcloud-admin.example.com` (via NPM). The setup wizard displays an initial password.

   > If NPM isn't configured yet, you can also reach the management UI directly at `http://localhost:${AIO_MANAGEMENT_PORT}` from the host (loopback-bound; use an SSH tunnel from another machine if needed).

2. **Copy the password immediately and store it in a password manager** — AIO doesn't show it again. Losing it locks you out of the mastercontainer.
3. Log in to the AIO setup UI with the displayed password.
4. **Set the domain:** e.g. `nextcloud.example.com`. AIO would normally validate that the domain resolves to this host — `SKIP_DOMAIN_VALIDATION=true` bypasses that check (necessary for Tailscale-only / split-DNS deployments).
5. **Select optional features:** Talk, Collabora/OnlyOffice, ClamAV, Imaginary, Full Text Search, etc. — enable what you'll use; each adds another spawned container.
6. Click **Start containers**. AIO downloads and starts all the sibling containers. This takes 2-5 minutes on first run.
7. Once status shows all green, open `https://nextcloud.example.com` (through NPM) and complete Nextcloud's first-time login as `admin`.
8. Change the admin password and create user accounts as needed.

### Day-to-day administration

- Use the AIO management UI (`https://nextcloud-admin.example.com`) for: enabling/disabling features, triggering AIO-managed backups, and performing major Nextcloud version upgrades.
- Use Nextcloud's own admin interface (`https://nextcloud.example.com/settings/admin`) for: user management, app installs, OCC commands via web, day-to-day settings.

---

## Backup

AIO has its own built-in backup system using BorgBackup — **use it instead of file-level snapshots**.

1. In the AIO management UI: **Backup → Set backup location** (e.g. `/mnt/nas/backup/nextcloud-aio` or a local `/path/to/backup/nextcloud-aio`).
2. **Generate and save the backup password** AIO shows you — same as the install password rules. Without it, the backup archive is unrecoverable.
3. Run **Create backup** to seed the first archive, then enable the scheduled daily run.

Kopia / Borgmatic external backups:

- The AIO Borg archive (whatever path you chose in step 1) is the **only** path you need to include in external backups. AIO's archive is internally consistent — it stops containers, snapshots the data, and restarts them.
- **Do not** snapshot `${NEXTCLOUD_DATADIR}`, the named volume `nextcloud_aio_mastercontainer`, or the spawned-container internals directly with running containers. File-level snapshots of live Nextcloud data can produce inconsistent / partially-corrupted archives.

> The AIO mastercontainer's backup password is independent of any user passwords or the AIO admin password. Treat it as a separate critical secret.

---

## Notes & Gotchas

- **The mastercontainer pattern is different from every other stack.** Compose only defines one service. The mastercontainer talks to the Docker daemon and spawns ~8-12 sibling containers (`nextcloud-aio-apache`, `nextcloud-aio-nextcloud`, `nextcloud-aio-database`, `nextcloud-aio-redis`, and more depending on enabled features). They appear in `docker ps` but **not** in this compose file, and they're managed by AIO — not by Docker Compose or by Dockge/Portainer's stack view. Don't try to `docker compose up` them individually or recreate them outside the AIO UI; let AIO orchestrate.
- **AIO's update model is decoupled from WUD.** This compose tracks only the mastercontainer image. The Nextcloud version, PHP version, Postgres version, etc. used by spawned containers are dictated by the mastercontainer — when you upgrade Nextcloud via the AIO UI, AIO pulls and recreates the sibling containers using its own logic. WUD notifications on this stack only signal "the mastercontainer itself has a new release"; upgrading the mastercontainer is what enables a new Nextcloud version to be installed via the AIO UI.
- **NPM has two hosts proxying this stack**, both reaching the upstream via container DNS over the `nextcloud-aio` network (which NPM joins as an external network). The Apache and mastercontainer endpoints are inside the docker network, not via host-published ports — the loopback host binding on the management port is just an SSH-tunnel fallback for direct access if NPM is down.
- **Wide `/mnt` mount.** AIO has RW access to anything under `/mnt`. This is the upstream-recommended pattern (it lets Nextcloud expose NAS paths as external storage), but it means losing the AIO admin password is functionally equivalent to losing root on the NAS-visible portion of the host. Store the AIO password and the backup password separately and securely.
- **Docker socket mounted read-only.** The mount uses `:ro`, matching the upstream AIO `compose.yaml` default. On a Unix socket the `:ro` flag affects filesystem permissions on the socket file, not the API writes that flow through it — so AIO can still call `docker run`, `docker start`, etc. via the socket. Upstream notes the flag may need to be dropped on macOS, Windows, or rootless Docker. If AIO ever fails to start with `Cannot connect to the docker socket`, that's a different problem (group permissions on the host socket) — dropping `:ro` won't help.
- **Tailscale-only / split-DNS deployments require `SKIP_DOMAIN_VALIDATION=true`.** Without it, AIO refuses to start because it can't validate that your Nextcloud domain is reachable from the public internet — which it isn't, by design, on a private network.

---

## References

- [Nextcloud AIO repo](https://github.com/nextcloud/all-in-one)
- [Nextcloud AIO — reverse proxy guide](https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md)
- [Nextcloud AIO — backup and restore](https://github.com/nextcloud/all-in-one#how-to-use-the-built-in-backup-feature)
- [Top-level README](../../README.md)
