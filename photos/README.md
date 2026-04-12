# photos

Photo and video management with Immich, including AI-powered search, face recognition, and NVIDIA GPU-accelerated machine learning and transcoding.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **Immich Server** | Core API and web interface | `ghcr.io/immich-app/immich-server` |
| **Immich Machine Learning** | AI/ML for facial recognition, search, and tagging (CUDA) | `ghcr.io/immich-app/immich-machine-learning` |
| **Valkey (Redis)** | Cache and job queue | `docker.io/valkey/valkey` |
| **PostgreSQL + pgvecto.rs** | Database with vector search for AI features | `ghcr.io/immich-app/postgres` |

## Network Architecture

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse proxy access and inter-service communication |

## Ports

| Service | Host Port | Container Port | Notes |
|---------|-----------|----------------|-------|
| Immich Server | `${IMMICH_PORT:-2283}` | 2283 | Web UI (localhost only) |

All other services are internal only.

## Environment Variables

Immich uses a `.env` file referenced via `env_file`. See [Immich environment docs](https://immich.app/docs/install/environment-variables) for the full list.

### Key Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `IMMICH_VERSION` | Immich version tag (defaults to `release`) | `release` |
| `IMMICH_PORT` | Immich web UI host port (defaults to `2283`) | `2283` |
| `UPLOAD_LOCATION` | Path to photo/video upload storage | `/mnt/photos` |
| `IMMICH_CACHE` | Path to ML model cache | `/opt/immich/cache` |
| `DB_PASSWORD` | PostgreSQL password | *(secret)* |
| `DB_USERNAME` | PostgreSQL username | `postgres` |
| `DB_DATABASE_NAME` | PostgreSQL database name | `immich` |
| `DB_DATA_LOCATION` | Path to PostgreSQL data directory | `/opt/immich/pgdata` |

## GPU Acceleration

### Transcoding (Immich Server)

Hardware-accelerated video transcoding via NVIDIA NVENC. Configuration is extended from `hwaccel.transcoding.yml`.

### Machine Learning (Immich ML)

CUDA-accelerated inference for face detection, CLIP search, and tagging. Uses the `-cuda` image variant and extends from `hwaccel.ml.yml`.

### Prerequisites

- NVIDIA GPU with drivers installed on the host
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html) installed
- `hwaccel.transcoding.yml` and `hwaccel.ml.yml` files present (from Immich docs)

## Dependencies

```
Valkey (redis) ────┐
PostgreSQL (db) ───┼──► Immich Server
                   │
                   └──► Immich Machine Learning (independent)
```

Immich Server waits for both Valkey and PostgreSQL healthchecks to pass before starting.

## Deployment

1. Create the external network if it doesn't exist:
   ```bash
   docker network create proxy_net
   ```

2. Create your `.env` file with all required variables. Download the hardware acceleration YAML files from the [Immich docs](https://immich.app/docs/install/docker-compose/).

3. Deploy:
   ```bash
   docker compose up -d
   ```

4. Access Immich at the reverse proxy URL and create your admin account.

## Notes

- Port is bound to `127.0.0.1` for reverse proxy access only.
- Valkey and PostgreSQL images are pinned with `@sha256:` digests for reproducibility.
- PostgreSQL uses `shm_size: 128mb` for shared memory.
- The `/etc/localtime` mount on the server ensures correct timestamps on uploaded media.
- The ML service can take several minutes to start on first launch while it downloads models.
