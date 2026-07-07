# searxng

Self-hosted privacy-respecting metasearch engine with [SearXNG](https://docs.searxng.org/) — aggregates results from dozens of search engines without tracking, suitable as a personal default search provider.

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `searxng` | `searxng/searxng:latest` | Web UI, search aggregator, results rendering |
| `searxng-valkey` (service key `valkey`) | `valkey/valkey:9-alpine` | Redis-protocol cache (Valkey fork); rate limiter and bot-detection state |

> Valkey is Redis Inc.'s open-source fork of Redis. SearXNG supports both; this homelab uses Valkey for consistency with the rest of the stacks that need a Redis-protocol cache (Immich also uses Valkey).

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse-proxy access via NPM (SearXNG only) |
| `searxng` | Internal bridge | SearXNG ↔ Valkey |

> Only `searxng` joins `proxy_net`. Valkey is reachable only on the internal `searxng` network.

---

## Ports

| Host port (`.env` var) | Container port | Bind | Purpose |
|------------------------|----------------|------|---------|
| `${SEARXNG_PORT}` (`8080`) | `8080` | all interfaces | Web UI |

Only `searxng` publishes a port; Valkey stays internal. Published on all interfaces so it's reachable on your LAN out of the box. To keep it behind a reverse proxy / loopback only, prefix the mapping in `compose.yaml` with `127.0.0.1:`. `8080` is a common port — change `SEARXNG_PORT` if it clashes.

---

## Volumes

| Host path (`.env` var) | Container path | Used by | Purpose |
|------------------------|----------------|---------|---------|
| `${SEARXNG_CONFIG}` | `/etc/searxng` | searxng | `settings.yml`, engine configs, custom UI overrides |
| `${SEARXNG_DATA}` | `/var/cache/searxng` | searxng | Search cache and runtime state |
| `${SEARXNG_DB_DATA}` | `/data` | valkey | Valkey persistence (`save 30 1` — write to disk after 30s if at least 1 change) |

---

## Environment Variables

### `.env` (Compose interpolation)

| Variable | Example | Purpose |
|----------|---------|---------|
| `SEARXNG_BASE_URL` | `https://searxng.example.com` | Public URL SearXNG advertises — used for CSRF, redirects, results links. Must match NPM hostname |
| `SEARXNG_PORT` | `8080` | Host port published for the web UI (container listens on `8080`) |
| `SEARXNG_SECRET_KEY` | *(secret)* | Session-signing key. Generate with `openssl rand -base64 32`. Blank in the template; **you must set it** |
| `SEARXNG_CONFIG` | `/path/to/searxng/core-config` | Host bind for SearXNG config |
| `SEARXNG_DATA` | `/path/to/searxng/data` | Host bind for SearXNG cache |
| `SEARXNG_DB_DATA` | `/path/to/searxng/valkey-data` | Host bind for Valkey persistence |

### Hardcoded in `compose.yaml`

| Variable | Value | Purpose |
|----------|-------|---------|
| `SEARXNG_VALKEY_URL` | `valkey://valkey:6379/0` | Internal Valkey URL — resolves over `searxng` network |
| `FORCE_OWNERSHIP` | `true` | SearXNG-image flag that re-chowns config/data dirs on startup. Useful when bind mounts have wrong ownership |

---

## Configuration

SearXNG generates a default `${SEARXNG_CONFIG}/settings.yml` on first start. Customize it to enable/disable engines, change UI defaults, and configure the rate limiter.

A minimal `settings.yml` example (relevant keys only):

```yaml
use_default_settings: true

server:
  secret_key: '${SEARXNG_SECRET_KEY}'  # injected from environment
  base_url: '${SEARXNG_BASE_URL}'      # injected from environment
  limiter: true                         # enable bot/abuse rate limiter
  public_instance: false                # private homelab instance
  image_proxy: true                     # proxy image thumbnails through SearXNG

ui:
  default_theme: simple
  default_locale: en
  query_in_title: true

search:
  safe_search: 0
  autocomplete: 'duckduckgo'
  formats:
    - html
    - json    # required for app integrations
```

> Setting `limiter: true` requires Valkey to be reachable — that's what the dependency on the `valkey` service guarantees.
>
> `use_default_settings: true` means SearXNG layers your overrides on top of the upstream defaults — you only need to specify what differs. The defaults change between SearXNG releases, so this keeps your config minimal.

Full schema: [SearXNG admin docs — settings.yml](https://docs.searxng.org/admin/settings/index.html).

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme |
|----------|---------------|---------------|--------|
| `searxng.example.com` | `searxng` | `8080` | `http` |

> **`SEARXNG_BASE_URL` must match this hostname exactly.** Mismatched values cause result links to point at the wrong URL and break the limiter's CSRF logic.
>
> **Forward client IP for rate limiting.** SearXNG's limiter applies per source IP — without `X-Forwarded-For` headers from NPM, every request appears to come from NPM's bridge IP and gets rate-limited as a single user. Add to NPM's "Custom Nginx Configuration" on this host:
>
> ```nginx
> proxy_set_header X-Real-IP $remote_addr;
> proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
> proxy_set_header X-Forwarded-Proto $scheme;
> ```

---

## Security Posture

This is the most-hardened stack in the homelab — useful as a reference for what minimal-cap container deployment looks like:

| Service | `cap_drop` | `cap_add` | Why |
|---------|------------|-----------|-----|
| `searxng` | `ALL` | `CHOWN`, `SETGID`, `SETUID` | Needs to chown bind mounts (`FORCE_OWNERSHIP`) and drop to non-root user inside the container |
| `valkey` | `ALL` | `CHOWN`, `SETGID`, `SETUID`, `DAC_OVERRIDE` | Same as above plus `DAC_OVERRIDE` for Valkey's persistence write to `/data` regardless of bind-mount UID quirks |

Both services also have `no-new-privileges` enabled. Removing any of the capabilities here may cause the containers to fail at startup with cryptic permission errors — only modify if you understand which Linux syscalls each cap covers.

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing)
- `searxng` internal network created automatically by compose
- Generated value for `SEARXNG_SECRET_KEY`

### Bring up

```bash
cd /opt/stacks/searxng
cp .env.example .env

# Generate secret
sed -i "s|SEARXNG_SECRET_KEY=.*|SEARXNG_SECRET_KEY=$(openssl rand -base64 32 | tr -d '\n')|" .env

nano .env  # set SEARXNG_BASE_URL
chmod 600 .env

# Pre-create local dirs (match your SEARXNG_* paths)
sudo mkdir -p /path/to/searxng/{core-config,data,valkey-data}

docker compose up -d
```

### First-run setup

1. Wait for SearXNG to write its default `settings.yml` to `${SEARXNG_CONFIG}` and reach the healthy state (~30s).
2. Open the app (e.g. `https://searxng.example.com` via NPM, or `http://<host-ip>:8080` directly) — basic search should already work.
3. Customize `${SEARXNG_CONFIG}/settings.yml`:
   - Pick a default theme (Settings UI → Preferences, or directly in the file)
   - Enable/disable specific engines (some are slow or low-quality)
   - Confirm `limiter: true` if you want abuse protection
4. Restart SearXNG to pick up settings changes: `docker compose restart searxng`.
5. Set SearXNG as your default search engine in your browser. For Firefox, the OpenSearch metadata is auto-detected on first visit; for Chrome-family browsers, add it manually under search-engine settings.

---

## Backup

Paths to include in Kopia / Borgmatic snapshots:

- `${SEARXNG_CONFIG}` — `settings.yml` and any UI customizations
- `.env` — contains `SEARXNG_SECRET_KEY`

Skip:
- `${SEARXNG_DATA}` — cache, regenerates on demand
- `${SEARXNG_DB_DATA}` — Valkey limiter state, ephemeral; loses an hour of "this IP was hammering" state on restore, otherwise no impact

---

## References

- [SearXNG docs — Docker install](https://docs.searxng.org/admin/installation-docker.html)
- [SearXNG docs — settings.yml schema](https://docs.searxng.org/admin/settings/index.html)
- [SearXNG public instances](https://searx.space/) — useful for comparison and engine selection
- [Top-level README](../../README.md)
