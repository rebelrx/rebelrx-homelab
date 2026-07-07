# backup

Two-tier backup strategy: [Kopia](https://kopia.io/) for off-host snapshots to the NAS, and [Borgmatic](https://torsion.org/borgmatic/) for fast local snapshots to a separate NVMe drive.

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `kopia` | `kopia/kopia:latest` | Off-host snapshots → NAS NFS repository |
| `borgmatic` | `ghcr.io/borgmatic-collective/borgmatic:latest` | On-host snapshots → secondary NVMe; cron-driven |

> The two services run **independently**. They don't talk to each other and don't share state — losing one doesn't affect the other.

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Kopia web UI access via NPM |
| `backup_default` | Auto-created bridge | Borgmatic only — isolated; Borg uses local FS, no networking required |

> Borgmatic doesn't declare a network, so compose places it on the stack's default bridge. That's intentional — Borgmatic only writes to a bind-mounted local repo and has no outbound needs.

---

## Ports

| Host port (`.env` var) | Container port | Bind | Purpose |
|------------------------|----------------|------|---------|
| `${KOPIA_UI_PORT}` (`51515`) | `51515` | all interfaces | Kopia web UI |

Only Kopia publishes a port; Borgmatic exposes nothing. Published on all interfaces so the UI is reachable on your LAN out of the box. To keep it behind a reverse proxy / loopback only, prefix the mapping in `compose.yaml` with `127.0.0.1:`.

---

## Volumes

### Kopia

| Host path (`.env` var) | Container path | Mode | Purpose |
|------------------------|----------------|------|---------|
| `${KOPIA_CONFIG_PATH}` | `/app/config` | rw | Kopia client config, repository connection |
| `${KOPIA_CACHE_PATH}` | `/app/cache` | rw | Local cache (chunk index, metadata) — slow to rebuild |
| `${KOPIA_LOG_PATH}` | `/app/logs` | rw | Operation logs |
| `${KOPIA_TMP_PATH}` | `/tmp` (`shared`) | shared | Temp working space; `shared` propagation for FUSE mounts during restore |
| `${KOPIA_LOCAL_PATH}` | `/data` | **ro** | Source data — mounted read-only |
| `${KOPIA_REPOSITORY_PATH}` | `/repository` | rw | Kopia repository on NAS NFS |

> The source tree is mounted read-only — Kopia can never modify it. Snapshot policies are defined inside Kopia's UI on subpaths like `/data/containers/<app>/config`. Bulk media on other NAS mounts is **not** included because those mounts live outside the source tree.

### Borgmatic

| Host path (`.env` var) | Container path | Mode | Purpose |
|------------------------|----------------|------|---------|
| `${BACKUP_SOURCE}` | `/mnt/source` | **ro** | Source tree mounted read-only |
| `${BACKUP_REPOSITORY}` | `/mnt/borg-repository` | rw | Local Borg repository on the secondary NVMe |
| `${BORGMATIC_D}` | `/etc/borgmatic.d` | rw | Borgmatic config files (`*.yaml` defining sources, retention, hooks) |
| `${BORGMATIC_CONFIG}` | `/root/.config/borg` | rw | Borg client config (despite the variable name; this is Borg's `BORG_CONFIG_DIR`, not borgmatic's config) |
| `${BORGMATIC_CACHE}` | `/root/.cache/borg` | rw | Borg chunk index — slow to rebuild, persist this |
| `${BORGMATIC_STATE}` | `/root/.local/state/borgmatic` | rw | Borgmatic run state (last-run timestamps, etc.) |
| `${BORGMATIC_SSH}` | `/root/.ssh` | rw | Reserved for future remote-repo use; can be empty for the current local-repo setup |

> `${BACKUP_REPOSITORY}` lives on a second NVMe drive on the host. **Add an fstab entry for that mount under `system/<hostname>/fstab.example` once stable** if you're mounting it manually.

---

## Environment Variables

### `.env` (Compose interpolation)

#### General

| Variable | Example | Purpose |
|----------|---------|---------|
| `TIMEZONE` | `America/New_York` | Container TZ |

#### Kopia

| Variable | Example | Purpose |
|----------|---------|---------|
| `KOPIA_UI_USER` | `admin` | Kopia web UI basic-auth username |
| `KOPIA_UI_PASS` | *(secret)* | Kopia web UI basic-auth password. Generate with `openssl rand -base64 24` |
| `KOPIA_PASSWORD` | *(secret)* | **Repository encryption password** — different from UI auth; this is what encrypts your snapshots. Without it, the repo is unrecoverable. Generate with `openssl rand -base64 32` |
| `KOPIA_HOSTNAME` | `docker-host` | Container hostname — Kopia labels snapshots by `user@hostname`; pin to a stable name |
| `KOPIA_UI_PORT` | `51515` | Host port published for the Kopia web UI |
| `KOPIA_CONFIG_PATH` | `/path/to/kopia/config` | Bind mount path (see Volumes) |
| `KOPIA_CACHE_PATH` | `/path/to/kopia/cache` | Bind mount path |
| `KOPIA_LOG_PATH` | `/path/to/kopia/logs` | Bind mount path |
| `KOPIA_LOCAL_PATH` | `/path/to/backup-source` | Source tree (mounted RO) |
| `KOPIA_REPOSITORY_PATH` | `/path/to/nas-backup/kopia-repo` | Repository on the NAS |
| `KOPIA_TMP_PATH` | `/path/to/kopia/tmp` | Temp/FUSE working dir |

#### Borgmatic

| Variable | Example | Purpose |
|----------|---------|---------|
| `BORG_PASSPHRASE` | *(secret)* | Borg repository passphrase. Without it, the repo is unrecoverable. Generate with `openssl rand -base64 32` |
| `BACKUP_SOURCE` | `/path/to/backup-source` | Source tree (mounted RO) |
| `BACKUP_REPOSITORY` | `/path/to/nvme-backup/borg-repo` | Local Borg repo |
| `BORGMATIC_D` | `/path/to/borgmatic/borgmatic.d` | Borgmatic YAML config dir |
| `BORGMATIC_CONFIG` | `/path/to/borgmatic/config` | Borg `BORG_CONFIG_DIR` |
| `BORGMATIC_CACHE` | `/path/to/borgmatic/cache` | Borg chunk cache |
| `BORGMATIC_STATE` | `/path/to/borgmatic/state` | Borgmatic state |
| `BORGMATIC_SSH` | `/path/to/borgmatic/ssh` | SSH dir (unused for local repo; reserved for remote backends) |

> `BACKUP_CRON=0 3 * * *` and `RUN_ON_STARTUP=false` are set directly in `compose.yaml`, not via `.env`. Edit them in compose if you want to change the schedule or run-on-start behavior.

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme |
|----------|---------------|---------------|--------|
| `kopia.example.com` | `kopia` | `51515` | `http` |

> Kopia runs with `--insecure` (no TLS at the container) and `--disable-csrf-token-checks`. TLS is provided by NPM in front; CSRF is disabled because NPM's request rewriting can interfere. The HTTP traffic between NPM and Kopia stays on `proxy_net`, which doesn't leave the host.

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing)
- Your NAS NFS share mounted on the host (e.g. `/mnt/nas`; see the top-level `system/<hostname>/fstab.example`)
- A secondary drive for the local Borg repo (e.g. `/mnt/nvme-backup`)
- Generated values for `KOPIA_UI_PASS`, `KOPIA_PASSWORD`, and `BORG_PASSPHRASE`

### Bring up

```bash
cd /opt/stacks/backup
cp .env.example .env

# Generate secrets
echo "KOPIA_UI_PASS=$(openssl rand -base64 24 | tr -d '\n')" >> .env
echo "KOPIA_PASSWORD=$(openssl rand -base64 32 | tr -d '\n')" >> .env
echo "BORG_PASSPHRASE=$(openssl rand -base64 32 | tr -d '\n')" >> .env

nano .env  # set KOPIA_UI_USER, KOPIA_HOSTNAME, review paths
chmod 600 .env

docker compose up -d
```

### First-run setup — Kopia

1. Open the app (e.g. `https://kopia.example.com` via NPM, or `http://<host-ip>:51515` directly) and log in with `KOPIA_UI_USER` / `KOPIA_UI_PASS`.
2. **Connect / create the repository** at `/repository` (filesystem repo type), using `KOPIA_PASSWORD` as the repository password.
3. Set up snapshot sources — typical paths under the mounted source tree:
   - `/data/containers/<app>/config` for each stack's config dir
   - `/data/compose` for the compose definitions themselves (if mirrored under the source tree)
4. Configure policies (retention, schedule, compression, encryption — defaults are sane).
5. Trigger an initial snapshot and verify it lands in `/repository`.

### First-run setup — Borgmatic

1. Drop one or more YAML files into `${BORGMATIC_D}`. Minimal example (`<BORGMATIC_D>/local.yaml`):

   ```yaml
   source_directories:
     - /mnt/source/containers
   repositories:
     - path: /mnt/borg-repository
       label: local
   keep_daily: 7
   keep_weekly: 4
   keep_monthly: 6
   compression: zstd,3
   encryption_passcommand: ""  # passphrase comes from $BORG_PASSPHRASE
   ```

2. Initialize the repo (one-time):

   ```bash
   docker compose exec borgmatic \
     borg init --encryption=repokey-blake2 /mnt/borg-repository
   ```

3. Run a manual backup to verify:

   ```bash
   docker compose exec borgmatic borgmatic --verbosity 1
   ```

4. Confirm the cron entry is active:

   ```bash
   docker compose logs borgmatic | grep -i cron
   ```

   It should show the `0 3 * * *` schedule.

---

## Backup

This **is** the backup stack — its own configuration is what gets backed up by other means.

Critical paths to externalize / record outside this host:

- `${KOPIA_CONFIG_PATH}` and `${BORGMATIC_D}` — repository connection details and YAML configs. If lost, you can rebuild from scratch but you lose source/retention definitions.
- `KOPIA_PASSWORD` and `BORG_PASSPHRASE` — **store these in a password manager outside the homelab.** Without them, the repositories are encrypted bricks.

> The `.env` file holds both passphrases. Keep it at `chmod 600`. Consider also keeping printed copies of both passphrases in physical secure storage — losing them at the same time as the host means losing every backup.

---

## References

- [Kopia docs](https://kopia.io/docs/)
- [Borgmatic docs](https://torsion.org/borgmatic/)
- [Borgmatic Docker image repo](https://github.com/borgmatic-collective/docker-borgmatic)
- [Top-level README](../../README.md)
