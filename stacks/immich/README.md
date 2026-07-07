# immich

Self-hosted photo and video management with [Immich](https://immich.app/) — Google Photos alternative with AI-powered search, face recognition, and mobile sync. GPU-accelerated transcoding and ML inference via the host's NVIDIA GPU.

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `immich-server` | `ghcr.io/immich-app/immich-server:${IMMICH_VERSION:-release}` | Web UI, API, background workers, microservices |
| `immich-machine-learning` | `ghcr.io/immich-app/immich-machine-learning:${IMMICH_VERSION:-release}-cuda` | CUDA-accelerated ML inference (face recognition, smart search, object detection) |
| `immich-redis` (service key `redis`) | `valkey/valkey:9@sha256:3eeb09…` | Redis-protocol cache (Valkey fork); job queue, session cache |
| `immich-postgres` (service key `database`) | `immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0@sha256:bcf633…` | PostgreSQL 14 with `pgvecto.rs` + `vectorchord` extensions for vector similarity search |

> The Postgres image is **Immich-specific** — vanilla `postgres:14` will not work. Immich requires `pgvecto.rs` (or its successor extensions) for the AI search features.

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse-proxy access via NPM (server only) |
| `immich` | Internal bridge | Server ↔ ML ↔ Valkey ↔ PostgreSQL |

> Only `immich-server` joins `proxy_net`. Everything else is internal-only.

---

## Ports

| Host port (`.env` var) | Container port | Bind | Purpose |
|------------------------|----------------|------|---------|
| `${IMMICH_HOST_PORT}` (`2283`) | `2283` | all interfaces | Web UI + API |

Only `immich-server` publishes a port; ML, Valkey, and Postgres stay internal. Published on all interfaces so it's reachable on your LAN out of the box. To keep it behind a reverse proxy / loopback only, prefix the mapping in `compose.yaml` with `127.0.0.1:`.

> **Why `IMMICH_HOST_PORT`, not `IMMICH_PORT`?** This stack mounts `.env` into the containers via `env_file:`, and Immich reads `IMMICH_PORT` to set its **internal** listen port. A var named `IMMICH_PORT` would therefore change the port Immich listens on inside the container and break the `:2283` mapping. `IMMICH_HOST_PORT` is ignored by Immich and used only for the host side.

---

## Hardware Acceleration

Immich uses two side-loaded compose files to enable GPU acceleration:

| File | Used by | Backend | Purpose |
|------|---------|---------|---------|
| `hwaccel.transcoding.yml` | `immich-server` (`extends: nvenc`) | NVENC | GPU-accelerated H.264/H.265 transcoding of uploaded videos |
| `hwaccel.ml.yml` | `immich-machine-learning` (`extends: cuda`) | CUDA | GPU-accelerated ML inference for face detection, embeddings, smart search |

Both files live alongside `compose.yaml` in `/opt/stacks/immich/` and are upstream-provided templates. The `extends:` directive in compose pulls in the `deploy.resources.reservations.devices` block that exposes the NVIDIA GPU to the container.

> **Prerequisites for GPU acceleration:**
> - NVIDIA proprietary driver + DKMS installed on the host
> - NVIDIA Container Toolkit installed and Docker daemon configured with the `nvidia` runtime
> - The Immich ML image tag must end in `-cuda` (it does — `${IMMICH_VERSION:-release}-cuda` in compose)
>
> Configure all three on your host before deploying. If you're documenting host-level NVIDIA setup in this repo, `system/<hostname>/README.md` is a sensible place for it.

---

## Volumes

| Host path (`.env` var) | Container path | Used by | Mode | Purpose |
|------------------------|----------------|---------|------|---------|
| `${UPLOAD_LOCATION}` | `/data` | server | rw | Photo and video library on NAS NFS — originals, derivatives, thumbnails, encoded videos |
| `${DB_DATA_LOCATION}` | `/var/lib/postgresql/data` | database | rw | PostgreSQL data — **must be local disk**, network shares are unsupported and will corrupt the DB |
| `${IMMICH_CACHE}` | `/cache` | ML | rw | ML model cache — downloaded on first inference, ~GB-scale |
| `/etc/localtime` | `/etc/localtime` | server | ro | Host timezone for EXIF date handling |

> **Library on NAS, DB on local disk.** This is intentional: photos are bulk media (large, slow OK) and benefit from NAS redundancy; the Postgres data dir needs fast local disk and would break on NFS.

---

## Environment Variables

### `.env` (Compose interpolation + container env via `env_file`)

Both `immich-server` and `immich-machine-learning` mount `.env` directly via `env_file:` — so every variable in `.env` is also available as a container env var. (This is why the host-port var is named `IMMICH_HOST_PORT` rather than `IMMICH_PORT` — see [Ports](#ports).)

#### General

| Variable | Example | Purpose |
|----------|---------|---------|
| `TZ` | `America/New_York` | Container TZ |
| `IMMICH_VERSION` | `release` | Image tag suffix. `release` = latest stable. Pin to a specific version like `v1.122.3` to freeze upgrades |
| `IMMICH_HOST_PORT` | `2283` | Host port published for the web UI / API (see [Ports](#ports)) |

#### Volumes

| Variable | Example | Purpose |
|----------|---------|---------|
| `UPLOAD_LOCATION` | `/path/to/nas-media/immich` | Photo/video library (NAS NFS is fine here) |
| `DB_DATA_LOCATION` | `/path/to/immich/postgres` | Postgres data dir (local disk only) |
| `IMMICH_CACHE` | `/path/to/immich/ml-cache` | ML model cache (local disk) |

#### Database

| Variable | Example | Purpose |
|----------|---------|---------|
| `DB_USERNAME` | `postgres` | Postgres username — kept as default per Immich docs |
| `DB_DATABASE_NAME` | `immich` | Postgres database name |
| `DB_PASSWORD` | *(secret)* | Postgres password — the upstream template ships `postgres`; **you must replace it**. Use `A-Za-z0-9` only (special chars break the connection string Immich builds). Generate with `openssl rand -hex 32` |

> Other Immich settings (SMTP, OAuth, external libraries, etc.) are configured **inside the Immich web UI**, not via env vars. See the [Admin Settings docs](https://immich.app/docs/administration/server-commands).

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme |
|----------|---------------|---------------|--------|
| `immich.example.com` | `immich-server` | `2283` | `http` |

> **Bump `client_max_body_size` in NPM** — Immich uploads can be hundreds of MB per video, and the mobile app uploads original-quality files. Default 1 MB is far too small. Set to `50G` or higher on this host.
>
> **Enable WebSocket support in NPM** — Immich uses WebSockets for real-time upload progress and notifications.

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing)
- `immich` internal network created automatically by compose
- Your NAS NFS share mounted on the host (for `${UPLOAD_LOCATION}`, e.g. `/mnt/nas`)
- NVIDIA Container Toolkit installed and Docker daemon configured with `nvidia` runtime
- `hwaccel.transcoding.yml` and `hwaccel.ml.yml` present in `/opt/stacks/immich/` alongside `compose.yaml`
- Generated value for `DB_PASSWORD` (using `A-Za-z0-9` only)

### Bring up

```bash
cd /opt/stacks/immich

# Copy template
cp .env.example .env

# Set DB password (alphanumeric only, no special chars)
sed -i "s|DB_PASSWORD=.*|DB_PASSWORD=$(openssl rand -hex 32)|" .env

# Confirm hwaccel files are present (these come from Immich's release)
ls hwaccel.transcoding.yml hwaccel.ml.yml

# Pre-create local dirs (DB and ML cache) — match your ${DB_DATA_LOCATION} / ${IMMICH_CACHE}
sudo mkdir -p /path/to/immich/postgres /path/to/immich/ml-cache

chmod 600 .env

docker compose up -d
```

### First-run setup

1. Wait ~60s for Postgres to initialize the `pgvecto.rs` and `vectorchord` extensions, then for Immich to run migrations. `docker compose logs immich-server` will show "Immich Server is listening on port 2283" when ready.
2. Open the app (e.g. `https://immich.example.com` via NPM, or `http://<host-ip>:2283` directly).
3. Create the **first admin user**.
4. Configure additional users from the admin panel as needed.
5. Install the mobile app (iOS/Android) and point it at your public URL (e.g. `https://immich.example.com`) — enable automatic upload from the camera roll.
6. Confirm GPU is being used:
   - **ML:** Upload a few photos, then check `nvidia-smi` on the host — `immich_ml` should appear in the process list during face detection / embedding.
   - **Transcoding:** Upload a video, then in Immich's admin settings under **Video Transcoding** verify NVENC is selected and active.

---

## Backup

Critical paths to include in Kopia / Borgmatic snapshots:

- `${DB_DATA_LOCATION}` — PostgreSQL data (users, albums, metadata, ML embeddings, all DB state)
- `${UPLOAD_LOCATION}` — the actual photo library on the NAS (already covered by NAS-level snapshots, but list it here for completeness)
- `${IMMICH_CACHE}` — **optional**, ML models will re-download if lost (~GB scale, costs network time only)
- `.env` — contains `DB_PASSWORD`

> Immich strongly recommends `pg_dump` for the DB on top of file-level snapshots, especially before major Immich upgrades. There's a [dedicated docs page](https://immich.app/docs/administration/backup-and-restore) covering exactly which paths and dumps you need:
>
> ```bash
> docker compose exec -T database \
>   pg_dumpall --clean --if-exists --username="$DB_USERNAME" \
>   | gzip > /path/to/immich/dumps/immich-$(date +%F).sql.gz
> ```

---

## Notes & Gotchas

- **Pinned image digests on `redis` and `database`.** Both images use `@sha256:...` rather than floating tags — this is Immich's recommended approach because they ship the specific Valkey/Postgres versions they've tested against. **Don't update these by hand.** When Immich bumps the digests in their release compose, update yours to match (their upgrade notes call this out). WUD will notify, but only when digest tags Immich publishes change; treat those events as "read Immich's release notes before applying".
- **`POSTGRES_INITDB_ARGS=--data-checksums`** is set on the database to enable page-level integrity checks. This adds a small write overhead but catches silent corruption — worthwhile for a long-lived photo DB.
- **Library on NAS, DB locally — non-negotiable.** Immich explicitly states the Postgres data dir must be on local storage. Network shares (including NFS) will corrupt the DB under load. Don't move `${DB_DATA_LOCATION}` onto a NAS.
- **`DB_PASSWORD` must be `A-Za-z0-9` only.** Special characters break the URL-encoded connection string Immich constructs internally. If migrating an existing password with `!` or `@`, change it.
- **ML container has no `no-new-privileges`.** Intentionally — the CUDA stack can conflict with the flag on certain driver versions. Server, Valkey, and Postgres still have it.
- **Mobile app must point at the public URL.** Use your public hostname (e.g. `https://immich.example.com`) or `http://<host-ip>:${IMMICH_HOST_PORT}`, not `http://immich:2283`. The internal hostname only works for container-to-container traffic.

---

## References

- [Immich docs — Docker Compose install](https://immich.app/docs/install/docker-compose/)
- [Immich docs — environment variables](https://immich.app/docs/install/environment-variables)
- [Immich docs — ML hardware acceleration](https://immich.app/docs/features/ml-hardware-acceleration)
- [Immich docs — hardware transcoding](https://immich.app/docs/features/hardware-transcoding)
- [Immich docs — backup and restore](https://immich.app/docs/administration/backup-and-restore)
- [Top-level README](../../README.md)
