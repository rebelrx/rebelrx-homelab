# speedtest

Network speed monitoring — scheduled WAN tests with [Speedtest Tracker](https://docs.speedtest-tracker.dev/), plus on-demand client-server speed tests via [Librespeed](https://librespeed.org/).

> **Two complementary tools.** Speedtest Tracker runs scheduled Ookla speedtests against the public internet to track ISP performance over time. Librespeed runs ad-hoc browser-driven tests against your homelab itself — useful for diagnosing LAN/WiFi throughput.

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `speedtest-tracker` | `lscr.io/linuxserver/speedtest-tracker:latest` | Scheduled Ookla speedtest results, history, graphs |
| `librespeed` | `lscr.io/linuxserver/librespeed:latest` | On-demand browser-driven speed test (point-to-point) |

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse-proxy access via NPM |

> Both services join `proxy_net` only. They don't communicate with each other — completely independent apps that happen to share a stack for grouping.

---

## Ports

| Host port (`.env` var) | Container port | Bind | Purpose |
|------------------------|----------------|------|---------|
| `${SPEEDTEST_HTTP_PORT}` (`8080`) | `80` | all interfaces | Speedtest Tracker web UI |
| `${LIBRESPEED_HTTP_PORT}` (`8081`) | `80` | all interfaces | Librespeed web UI |

Each service listens on container port `80`, so they use **distinct host ports**. Published on all interfaces so both UIs are reachable on your LAN out of the box. To keep either behind a reverse proxy / loopback only, prefix its mapping in `compose.yaml` with `127.0.0.1:`. `8080` is a common port — change the vars if they clash.

> Speedtest Tracker also listens internally on `443` (HTTPS, self-signed); it's not published. NPM connects to either app via `proxy_net` using container DNS, so the reverse proxy works regardless of the host-port mapping.

---

## Volumes

| Host path (`.env` var) | Container path | Used by | Purpose |
|------------------------|----------------|---------|---------|
| `${SPEEDTEST_CONFIG}` | `/config` | speedtest-tracker | SQLite DB (results history), config, user accounts |
| `${LIBRESPEED_CONFIG}` | `/config` | librespeed | SQLite DB (results), config |

Both apps use SQLite — no separate DB service.

---

## Environment Variables

### `.env` (Compose interpolation)

#### General

| Variable | Example | Purpose |
|----------|---------|---------|
| `PUID` | `1000` | UID inside containers (linuxserver convention) |
| `PGID` | `1000` | GID inside containers |
| `TIMEZONE` | `America/New_York` | Container TZ |
| `SPEEDTEST_HTTP_PORT` | `8080` | Host port for the Speedtest Tracker UI (container `80`) |
| `LIBRESPEED_HTTP_PORT` | `8081` | Host port for the Librespeed UI (container `80`) |

#### Speedtest Tracker

| Variable | Example | Purpose |
|----------|---------|---------|
| `APP_KEY` | *(secret)* | Laravel app key for session/cookie encryption. Generate with `openssl rand -base64 32` |
| `APP_URL` | `https://speedtest.example.com` | Public URL Speedtest Tracker advertises |
| `ASSET_URL` | `https://speedtest.example.com` | URL for serving CSS/JS assets — usually same as `APP_URL` |
| `DISPLAY_TIMEZONE` | `America/New_York` | TZ used for rendering result timestamps in the UI |
| `SPEEDTEST_SCHEDULE` | `0 0 * * *` | Cron schedule for automatic tests (default: midnight daily) |
| `SPEEDTEST_CONFIG` | `/path/to/speedtest-tracker/data` | Host bind for app data |

#### Librespeed

| Variable | Example | Purpose |
|----------|---------|---------|
| `LIBRESPEED_PASSWORD` | *(secret)* | Admin password for the Librespeed results-management interface |
| `LIBRESPEED_CONFIG` | `/path/to/librespeed/config` | Host bind for app data |

### Hardcoded in `compose.yaml`

| Variable | Value | Purpose |
|----------|-------|---------|
| `DB_CONNECTION` | `sqlite` | Speedtest Tracker uses SQLite |
| `DB_TYPE` | `sqlite` | Librespeed uses SQLite |
| `CUSTOM_RESULTS` | `false` | Librespeed: don't allow custom result-submission scripts |
| `PRUNE_RESULTS_OLDER_THAN` | `7` | Speedtest Tracker: auto-delete results older than 7 days |

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme |
|----------|---------------|---------------|--------|
| `speedtest.example.com` | `speedtest-tracker` | `80` | `http` |
| `librespeed.example.com` | `librespeed` | `80` | `http` |

> **`APP_URL` must match the NPM hostname exactly** for Speedtest Tracker. Wrong value breaks the UI's CSRF protection and result links.
>
> **`ASSET_URL` should match too.** Mismatched values cause the UI to load with broken CSS/JS because asset URLs point at the wrong origin.

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing)
- Generated values for `APP_KEY` and `LIBRESPEED_PASSWORD`

### Bring up

```bash
cd /opt/stacks/speedtest
cp .env.example .env

# Generate secrets (fills the blank values in place)
sed -i "s|APP_KEY=.*|APP_KEY=$(openssl rand -base64 32 | tr -d '\n')|" .env
sed -i "s|LIBRESPEED_PASSWORD=.*|LIBRESPEED_PASSWORD=$(openssl rand -base64 24 | tr -d '\n')|" .env

nano .env  # set APP_URL, ASSET_URL, DISPLAY_TIMEZONE
chmod 600 .env

# Pre-create local dirs (match your SPEEDTEST_CONFIG / LIBRESPEED_CONFIG)
sudo mkdir -p /path/to/speedtest-tracker/data
sudo mkdir -p /path/to/librespeed/config

docker compose up -d
```

### First-run setup

#### Speedtest Tracker

1. Open the app (e.g. `https://speedtest.example.com` via NPM, or `http://<host-ip>:8080` directly).
2. Default admin credentials: `admin@example.com` / `password` — **change immediately** via the admin profile.
3. Optionally configure result thresholds, notification channels (Slack, Discord, email), and notification webhooks under Settings.
4. Trigger a manual test to verify the GeoIP database is loaded and the scheduled cron is healthy.

#### Librespeed

1. Open the app (e.g. `https://librespeed.example.com` via NPM, or `http://<host-ip>:8081` directly).
2. The main page is the test interface — anyone with the URL can run a test.
3. Visit `/results/?op=login` and authenticate with `LIBRESPEED_PASSWORD` to view the results database.

---

## Backup

Paths to include in Kopia / Borgmatic snapshots:

- `${SPEEDTEST_CONFIG}` — SQLite DB with full test history (graphs and trend data live here)
- `${LIBRESPEED_CONFIG}` — SQLite DB with on-demand test results
- `.env` — contains `APP_KEY` and `LIBRESPEED_PASSWORD`

Both SQLite DBs are small (tens of MB even after months of history). Hot snapshots are generally fine; for safety, stop the containers briefly before snapshotting if you want guaranteed-consistent dumps:

```bash
docker compose stop && \
  rsync -a /path/to/speedtest-tracker/ /path/to/librespeed/ /tmp/speedtest-snapshot/ && \
  docker compose start
```

---

## References

- [Speedtest Tracker docs](https://docs.speedtest-tracker.dev/)
- [LinuxServer.io — docker-speedtest-tracker](https://hub.docker.com/r/linuxserver/speedtest-tracker)
- [LinuxServer.io — docker-librespeed](https://docs.linuxserver.io/images/docker-librespeed/)
- [Top-level README](../../README.md)
