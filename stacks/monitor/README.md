# monitor

Infrastructure monitoring with [Prometheus](https://prometheus.io/) (metrics collection), [Grafana](https://grafana.com/) (dashboards), and [Node Exporter](https://github.com/prometheus/node_exporter) (host-level metrics).

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `grafana` | `grafana/grafana:latest` | Dashboards, alerting UI, datasource integrations |
| `prometheus` | `prom/prometheus:latest` | Time-series metrics database and scrape engine |
| `node-exporter` | `prom/node-exporter:latest` | Exports host CPU, memory, disk, network, filesystem metrics |

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse-proxy access via NPM (Grafana only) |
| `monitor` | Internal bridge | Grafana ↔ Prometheus ↔ Node Exporter |

> Only `grafana` joins `proxy_net`. Prometheus and node-exporter are reachable only on the internal `monitor` network — Prometheus exposes admin endpoints you don't want public, and node-exporter is a raw metrics scrape target.

---

## Ports

| Host port (`.env` var) | Container port | Bind | Purpose |
|------------------------|----------------|------|---------|
| `${GRAFANA_PORT}` (`3000`) | `3000` | all interfaces | Grafana UI |

Only Grafana publishes a port. **Prometheus (`9090`) and node-exporter (`9100`) are intentionally not published** — Prometheus has admin endpoints you don't want on the LAN, and node-exporter is a raw scrape target. Both are reached only via internal DNS over the `monitor` network (Grafana scrapes Prometheus; Prometheus scrapes node-exporter).

Grafana is published on all interfaces so the UI is reachable on your LAN out of the box. To keep it behind a reverse proxy / loopback only, prefix the mapping in `compose.yaml` with `127.0.0.1:`. `3000` is a common port — change `GRAFANA_PORT` if it clashes.

> To query Prometheus directly, attach a temporary container to the `monitor` network or use Grafana's "Explore" view. To expose it anyway, add a `ports:` mapping to the `prometheus` service.

---

## Volumes

### Grafana

| Host path (`.env` var) | Container path | Purpose |
|------------------------|----------------|---------|
| `${GRAFANA_DATA}` | `/var/lib/grafana` | Grafana SQLite DB, users, sessions, alert state, plugins |
| `${GRAFANA_PROVISIONING}` | `/etc/grafana/provisioning` | Declarative provisioning — YAML files defining datasources, dashboards, alert rules |
| `${GRAFANA_DASHBOARDS}` | `/var/lib/grafana/dashboards` | JSON dashboard files referenced by provisioning |

### Prometheus

| Host path (`.env` var) | Container path | Purpose |
|------------------------|----------------|---------|
| `${PROMETHEUS_CONFIG}` | `/etc/prometheus` | `prometheus.yml` and any rule files |
| `${PROMETHEUS_DATA}` | `/prometheus` | Time-series database (TSDB) — all historical metrics |

### Node Exporter

| Host path | Container path | Mode | Purpose |
|-----------|----------------|------|---------|
| `/proc` | `/host/proc` | `ro` | Host process tree (read-only) |
| `/sys` | `/host/sys` | `ro` | Host sysfs (read-only) |
| `/` | `/host` | `ro,rslave` | Host root for filesystem metrics (read-only, slave mount propagation) |

> **Bind-mount ownership.** Grafana and Prometheus run as `${PUID}:${PGID}` (1000:1000 by default). The host directories `${GRAFANA_DATA}` and `${PROMETHEUS_DATA}` must be writable by that UID/GID, or the containers will crash on startup with permission errors. Pre-create them:
>
> ```bash
> sudo mkdir -p /path/to/grafana/{data,provisioning,dashboards}
> sudo mkdir -p /path/to/prometheus/{config,data}
> sudo chown -R 1000:1000 /path/to/grafana /path/to/prometheus
> ```

---

## Environment Variables

### `.env` (Compose interpolation)

#### General

| Variable | Example | Purpose |
|----------|---------|---------|
| `PUID` | `1000` | UID Grafana and Prometheus run as inside their containers |
| `PGID` | `1000` | GID Grafana and Prometheus run as inside their containers |
| `TIMEZONE` | `America/New_York` | Container TZ |
| `GRAFANA_PORT` | `3000` | Host port published for the Grafana UI |

#### Volumes

| Variable | Example | Purpose |
|----------|---------|---------|
| `GRAFANA_DATA` | `/path/to/grafana/data` | Grafana DB / state |
| `GRAFANA_PROVISIONING` | `/path/to/grafana/provisioning` | Provisioning YAML |
| `GRAFANA_DASHBOARDS` | `/path/to/grafana/dashboards` | Dashboard JSON |
| `PROMETHEUS_CONFIG` | `/path/to/prometheus/config` | `prometheus.yml` + rules |
| `PROMETHEUS_DATA` | `/path/to/prometheus/data` | TSDB |

#### Reference only

| Variable | Value | Purpose |
|----------|-------|---------|
| `PROMETHEUS_PORT` | `9090` | Reference only — Prometheus is intentionally not published (internal scrape/admin) |
| `NODE_EXPORTER_PORT` | `9100` | Reference only — node-exporter is intentionally not published (internal scrape target) |

---

## `prometheus.yml`

Prometheus requires `prometheus.yml` at `${PROMETHEUS_CONFIG}/prometheus.yml` — the container exits at startup if it's missing. This stack ships a template at [`prometheus.yml.example`](prometheus.yml.example); copy it into the config dir before first start:

```bash
cp prometheus.yml.example ${PROMETHEUS_CONFIG}/prometheus.yml
```

The shipped template scrapes Prometheus itself and node-exporter:

```yaml
global:
  scrape_interval: 15s
  scrape_timeout: 10s
  evaluation_interval: 15s
alerting:
  alertmanagers:
    - static_configs:
      - targets: []
      scheme: http
      timeout: 10s
      api_version: v2
scrape_configs:
  - job_name: prometheus
    honor_timestamps: true
    scrape_interval: 15s
    scrape_timeout: 10s
    metrics_path: /metrics
    scheme: http
    static_configs:
      - targets:
        - prometheus:9090
  - job_name: node
    static_configs:
      - targets:
        - node-exporter:9100
```

Targets use container DNS names (`prometheus`, `node-exporter`) resolved over the `monitor` network — the hostnames match the container names. Add each additional scrape target (cAdvisor, blackbox-exporter, app-specific exporters) as another entry under `scrape_configs`.

Full schema: [Prometheus config docs](https://prometheus.io/docs/prometheus/latest/configuration/configuration/).

> The Grafana provisioning YAML below could be committed to this stack directory the same way (as `*.example` templates) if you want it version-controlled — it's small and secret-free.

---

## Grafana Provisioning

`${GRAFANA_PROVISIONING}` enables file-based provisioning so datasources and dashboards survive container resets and are version-controllable.

Typical layout under `${GRAFANA_PROVISIONING}`:

```
provisioning/
├── datasources/
│   └── prometheus.yml        # Defines the Prometheus datasource pointing at http://prometheus:9090
├── dashboards/
│   └── default.yml           # Tells Grafana to load dashboards from /var/lib/grafana/dashboards
└── alerting/                 # Optional: declarative alert rules and contact points
```

Example `datasources/prometheus.yml`:

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
```

Example `dashboards/default.yml`:

```yaml
apiVersion: 1
providers:
  - name: 'default'
    orgId: 1
    folder: ''
    type: file
    options:
      path: /var/lib/grafana/dashboards
```

Then drop dashboard JSON files into `${GRAFANA_DASHBOARDS}` and Grafana picks them up on startup.

Full schema: [Grafana provisioning docs](https://grafana.com/docs/grafana/latest/administration/provisioning/).

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme |
|----------|---------------|---------------|--------|
| `grafana.example.com` | `grafana` | `3000` | `http` |

> **Enable WebSocket support in NPM** — Grafana's live alerting and panel auto-refresh use WebSockets.

Prometheus is **not** exposed via NPM by design. To query Prometheus directly, attach a temporary container to the `monitor` network or use Grafana's "Explore" view as a passthrough.

---

## Node Exporter — Security Model

Node exporter is more privileged than the average container:

- `pid: host` — shares the host's PID namespace (required to enumerate host processes)
- Mounts `/proc`, `/sys`, and `/` from the host **read-only**
- `rslave` mount propagation on `/` prevents the container from injecting mounts back into the host

`no-new-privileges` is still active. The read-only bind mounts are the key safety mechanism — node-exporter can observe everything on the host but can't modify anything. This is the canonical pattern from the upstream node-exporter docs; don't drop the `:ro` flags.

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing)
- `monitor` internal network created automatically by compose
- Host directories created with correct ownership (see [Volumes](#volumes))
- `prometheus.yml` present at `${PROMETHEUS_CONFIG}/prometheus.yml` — copy the shipped [`prometheus.yml.example`](prometheus.yml.example) into place
- Optional: Grafana provisioning YAML files at `${GRAFANA_PROVISIONING}`

### Bring up

```bash
cd /opt/stacks/monitor
cp .env.example .env
nano .env

# Pre-create host dirs with correct ownership (match your ${GRAFANA_*} / ${PROMETHEUS_*} paths)
sudo mkdir -p /path/to/grafana/{data,provisioning,dashboards}
sudo mkdir -p /path/to/prometheus/{config,data}
sudo chown -R 1000:1000 /path/to/grafana /path/to/prometheus

# Copy prometheus.yml into place before first start (required — Prometheus won't start without it)
cp prometheus.yml.example /path/to/prometheus/config/prometheus.yml

docker compose up -d
```

### First-run setup

1. Open the app (e.g. `https://grafana.example.com` via NPM, or `http://<host-ip>:3000` directly).
2. Default credentials: `admin` / `admin` — Grafana prompts to change on first login.
3. If you used provisioning, the Prometheus datasource and dashboards appear automatically. Otherwise:
   - **Connections → Data Sources → Add data source → Prometheus**
   - URL: `http://prometheus:9090`
   - Save & test
4. Import the [Node Exporter Full](https://grafana.com/grafana/dashboards/1860-node-exporter-full/) dashboard (ID `1860`) for a comprehensive host-metrics view.

---

## Backup

Paths to include in Kopia / Borgmatic snapshots:

- `${GRAFANA_DATA}` — users, sessions, alert state, installed plugins
- `${GRAFANA_PROVISIONING}` and `${GRAFANA_DASHBOARDS}` — small, change-tracked; consider version-controlling these in the homelab repo
- `${PROMETHEUS_CONFIG}` — `prometheus.yml` and rule files; same — git-track
- `${PROMETHEUS_DATA}` — TSDB; can grow large. Optional in backups since historical metrics are rebuildable (the universe regenerates them every 15s)

---

## References

- [Prometheus docs — installation](https://prometheus.io/docs/prometheus/latest/installation/)
- [Prometheus docs — configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)
- [Grafana docs — Docker install](https://grafana.com/docs/grafana/latest/setup-grafana/installation/docker/)
- [Grafana docs — provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/)
- [Node Exporter on Docker Hub](https://hub.docker.com/r/prom/node-exporter)
- [Top-level README](../../README.md)
