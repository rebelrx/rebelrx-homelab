# Stack: backup

## Purpose
Runs a Kopia server (Web UI + API) to manage backups/snapshots of host data and store them in a Kopia repository.

This stack is the control plane for backups:
- creates and schedules snapshots
- retains/prunes per policy
- stores data in a repository mounted on the host/NAS
- provides a Web UI/API on port 51515 (mapped via `${PORT}`)

---

## Services

| Service | Image | Exposed | Purpose |
|--------|-------|---------|---------|
| kopia  | `kopia/kopia:latest` | `${PORT} -> 51515` | Kopia server (UI + API) |

---

## Prerequisites

1. Docker + Docker Compose v2
2. Required paths exist on the host (set via `.env`):
   - `${KOPIA_CONFIG_PATH}`
   - `${KOPIA_CACHE_PATH}`
   - `${KOPIA_LOG_PATH}`
   - `${KOPIA_LOCAL_PATH}` (read-only snapshot source)
   - `${KOPIA_REPOSITORY_PATH}` (repository storage)
   - `${KOPIA_TMP_PATH}` (mounted snapshot browsing)

---

## Environment Variables

Configured via `.env`.

Required:
```env
PORT=
TIMEZONE=America/New_York

# Kopia server auth (Web UI / API)
KOPIA_UI_USER=
KOPIA_UI_PASS=

# Kopia repository password (used by Kopia)
KOPIA_PASSWORD=

# Host paths
KOPIA_CONFIG_PATH=
KOPIA_CACHE_PATH=
KOPIA_LOG_PATH=
KOPIA_LOCAL_PATH=
KOPIA_REPOSITORY_PATH=
KOPIA_TMP_PATH=
