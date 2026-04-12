# monitor

Infrastructure monitoring stack with Grafana dashboards, Prometheus metrics collection, and Node Exporter for host-level metrics.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **Grafana** | Dashboard and visualization platform | `grafana/grafana` |
| **Prometheus** | Time-series metrics database and alerting | `prom/prometheus` |
| **Node Exporter** | Host hardware and OS metrics exporter | `prom/node-exporter` |

## Network Architecture

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse proxy access |
| `monitor` | Bridge | Internal communication between monitoring services |

## Ports

| Service | Host Port | Container Port | Notes |
|---------|-----------|----------------|-------|
| Grafana | `${GRAFANA_PORT}` | 3000 | Web UI (localhost only) |
| Prometheus | `${PROMETHEUS_PORT}` | 9090 | Web UI and API |
| Node Exporter | `${NODE_EXPORTER_PORT}` | 9100 | Metrics endpoint |

## Environment Variables

Create a `.env` file in the same directory as the compose file.

| Variable | Description | Example |
|----------|-------------|---------|
| `PUID` | User ID for Grafana and Prometheus | `1000` |
| `PGID` | Group ID for Grafana and Prometheus | `1000` |
| `TIMEZONE` | Timezone | `America/New_York` |
| `GRAFANA_PORT` | Grafana web UI host port | `3000` |
| `PROMETHEUS_PORT` | Prometheus host port | `9090` |
| `NODE_EXPORTER_PORT` | Node Exporter host port | `9100` |
| `GRAFANA_DATA` | Path to Grafana data directory | `/opt/grafana/data` |
| `GRAFANA_PROVISIONING` | Path to Grafana provisioning configs | `/opt/grafana/provisioning` |
| `GRAFANA_DASHBOARDS` | Path to Grafana dashboard JSON files | `/opt/grafana/dashboards` |
| `PROMETHEUS_CONFIG` | Path to Prometheus config directory | `/opt/prometheus/config` |
| `PROMETHEUS_DATA` | Path to Prometheus data directory | `/opt/prometheus/data` |

## Deployment

1. Create the external network if it doesn't exist:
   ```bash
   docker network create proxy_net
   ```

2. Create your `.env` file with all required variables.

3. Create a `prometheus.yml` config file in your `PROMETHEUS_CONFIG` directory. At minimum, configure scrape targets for Prometheus itself and Node Exporter:
   ```yaml
   scrape_configs:
     - job_name: 'prometheus'
       static_configs:
         - targets: ['localhost:9090']
     - job_name: 'node'
       static_configs:
         - targets: ['node-exporter:9100']
   ```

4. Deploy:
   ```bash
   docker compose up -d
   ```

5. Access Grafana at the reverse proxy URL, add Prometheus as a data source (`http://prometheus:9090`), and import or create dashboards.

## Notes

- Grafana and Prometheus run as `${PUID}:${PGID}` via the `user` directive. Ensure the data directories are owned by this user.
- Node Exporter runs with `pid: host` and mounts `/proc`, `/sys`, and `/` read-only to collect host-level metrics.
- Grafana's port is bound to `127.0.0.1` for reverse proxy access. Prometheus and Node Exporter are not localhost-bound — restrict access via firewall if needed.
- Grafana provisioning allows you to pre-configure data sources and dashboards via YAML files.
