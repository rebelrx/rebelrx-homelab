# search

Self-hosted metasearch engine with SearXNG, backed by a Valkey cache for improved performance. SearXNG aggregates results from multiple search engines without tracking users.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **SearXNG** | Privacy-respecting metasearch engine | `docker.io/searxng/searxng` |
| **Valkey** | In-memory data store used as SearXNG cache | `docker.io/valkey/valkey:9-alpine` |

## Network Architecture

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse proxy access to SearXNG |
| `searxng_net` | Internal (bridge) | Private network between SearXNG and Valkey |

## Ports

| Service | Host Port | Container Port | Notes |
|---------|-----------|----------------|-------|
| SearXNG | `${SEARXNG_PORT}` | 8080 | Web UI (localhost only) |
| Valkey | — | 6379 | Internal network only, not exposed |

## Environment Variables

Create a `.env` file in the same directory as the compose file.

### General

| Variable | Description | Example |
|----------|-------------|---------|
| `SEARXNG_PORT` | SearXNG web UI host port | `8080` |

### SearXNG

| Variable | Description | Example |
|----------|-------------|---------|
| `SEARXNG_BASE_URL` | Public-facing base URL for SearXNG | `https://search.example.com/` |
| `SEARXNG_SECRET_KEY` | Secret key used by SearXNG (generate with `openssl rand -hex 32`) | *(secret)* |
| `SEARXNG_CONFIG` | Host path for SearXNG configuration | `/srv/searxng/config` |
| `SEARXNG_DATA` | Host path for SearXNG cache data | `/srv/searxng/data` |
| `SEARXNG_DB_DATA` | Host path for Valkey persistence data | `/srv/searxng/valkey` |

## Volumes

| Service | Container Path | Purpose |
|---------|----------------|---------|
| SearXNG | `/etc/searxng` | Configuration files (`settings.yml`, etc.) |
| SearXNG | `/var/cache/searxng` | Search cache data |
| Valkey | `/data` | Valkey persistence (snapshot saved every 30s if at least 1 key changed) |

## Security

Both containers drop all Linux capabilities by default and add back only what's required:

| Service | Capabilities Added |
|---------|--------------------|
| SearXNG | `CHOWN`, `SETGID`, `SETUID` |
| Valkey | `CHOWN`, `SETGID`, `SETUID`, `DAC_OVERRIDE` |

## Container Labels

To enable update monitoring via the [wud](../wud/) stack, add the following to each service:

```yaml
labels:
  - wud.watch=true
```

## Deployment

1. Create the external network if it doesn't exist:
   ```bash
   docker network create proxy_net
   ```

2. Create host directories for the bind-mounted volumes and ensure correct ownership:
   ```bash
   mkdir -p /srv/searxng/{config,data,valkey}
   ```

3. Create your `.env` file with all required variables. Generate a secret key with:
   ```bash
   openssl rand -hex 32
   ```

4. Deploy:
   ```bash
   docker compose up -d
   ```

5. On first start, SearXNG will populate `${SEARXNG_CONFIG}` with a default `settings.yml`. Edit it to customize enabled engines, UI options, and the `secret_key` if not set via environment.

6. Access SearXNG at the reverse proxy URL configured in `SEARXNG_BASE_URL`.

## Notes

- SearXNG's port is bound to `127.0.0.1` for reverse proxy access only.
- Valkey is not exposed to the host; it is reachable only by SearXNG over the internal `searxng_net` bridge.
- `FORCE_OWNERSHIP=true` causes SearXNG to fix ownership of its config/data directories on startup — required for the reduced capability set to work with bind mounts.
- Valkey is a BSD-licensed fork of Redis and is a drop-in replacement; SearXNG connects via the `valkey://` URL scheme.
- The `valkey-server --save 30 1` flag configures a snapshot every 30 seconds if at least 1 key has changed.
