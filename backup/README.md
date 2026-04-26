# Stack: backup

## Purpose
Provides two complementary backup tools:

- **Kopia** — a Kopia server (Web UI + API) for managing backups/snapshots of host data into a Kopia repository (typically NAS / remote storage).
- **Borgmatic** — a scheduled Borg backup runner for managing local Borg backups via cron.

Together this stack acts as the control plane for backups:
- creates and schedules snapshots (Kopia UI/API + Borgmatic cron)
- retains/prunes per policy
- stores data in repositories mounted on the host/NAS
- exposes Kopia's Web UI/API on port 51515 (mapped via `${PORT}`)

---

## Services

| Service   | Image | Exposed | Purpose |
|-----------|-------|---------|---------|
| kopia     | `kopia/kopia:latest` | `${PORT} -> 51515` | Kopia server (UI + API) — NAS / remote backups |
| borgmatic | `ghcr.io/borgmatic-collective/borgmatic:latest` | — | Scheduled Borg local backups via cron |

---

## Prerequisites

1. Docker + Docker Compose v2
2. External network `proxy_net` exists (used by `kopia`)
3. Required host paths exist (set via `.env`):

   **Kopia:**
   - `${KOPIA_CONFIG_PATH}`
   - `${KOPIA_CACHE_PATH}`
   - `${KOPIA_LOG_PATH}`
   - `${KOPIA_LOCAL_PATH}` (read-only snapshot source)
   - `${KOPIA_REPOSITORY_PATH}` (repository storage)
   - `${KOPIA_TMP_PATH}` (mounted snapshot browsing)

   **Borgmatic:**
   - `${BACKUP_SOURCE}` (read-only backup source)
   - `${BACKUP_REPOSITORY}` (Borg repository storage)
   - `${BORGMATIC_D}` (borgmatic config directory)
   - `${BORGMATIC_CONFIG}` (Borg config directory)
   - `${BORGMATIC_CACHE}` (Borg cache directory)
   - `${BORGMATIC_STATE}` (Borgmatic state directory)
   - `${BORGMATIC_SSH}` (SSH keys for remote repos, if used)

---

## Environment Variables

Configured via `.env`.

Required:
```env
PORT=
TIMEZONE=America/New_York

# --- Kopia ---
# Kopia server auth (Web UI / API)
KOPIA_UI_USER=
KOPIA_UI_PASS=

# Kopia repository password (used by Kopia)
KOPIA_PASSWORD=

# Kopia host paths
KOPIA_CONFIG_PATH=
KOPIA_CACHE_PATH=
KOPIA_LOG_PATH=
KOPIA_LOCAL_PATH=
KOPIA_REPOSITORY_PATH=
KOPIA_TMP_PATH=

# --- Borgmatic ---
# Borg repository passphrase
BORG_PASSPHRASE=

# Borgmatic host paths
BACKUP_SOURCE=
BACKUP_REPOSITORY=
BORGMATIC_D=
BORGMATIC_CONFIG=
BORGMATIC_CACHE=
BORGMATIC_STATE=
BORGMATIC_SSH=
```

---

## Notes

- **Borgmatic schedule:** runs daily at 03:00 (`BACKUP_CRON=0 3 * * *`), with `RUN_ON_STARTUP=false` so backups don't fire on container start.
- **Borgmatic config:** drop one or more `*.yaml` config files into `${BORGMATIC_D}` to define source paths, retention, and repository targets.
- **WUD labels:** both services are watched by WUD. Kopia has `wud.trigger.docker.enable=false` (manual updates), Borgmatic has `wud.trigger.docker.enable=true` (auto-update on new image).
