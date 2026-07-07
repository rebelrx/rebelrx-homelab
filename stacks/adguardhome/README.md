# adguardhome

Network-wide DNS filtering and ad blocking with [AdGuard Home](https://adguard.com/en/adguard-home/overview.html).

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `adguardhome` | `adguard/adguardhome:latest` | DNS resolver, filter engine, and admin UI |

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse-proxy access via NPM |

---

## Ports

All ports are published on **all interfaces** so LAN clients can reach the resolver out of the box.

| Host binding | Container | Purpose |
|--------------|-----------|---------|
| `0.0.0.0:53/tcp` | `53/tcp` | DNS — must be reachable by every LAN client |
| `0.0.0.0:53/udp` | `53/udp` | DNS — must be reachable by every LAN client |
| `0.0.0.0:${ADGUARD_WEBUI_PORT}` (`3000`) | `3000` | Admin web UI + setup wizard |
| `0.0.0.0:${ADGUARD_TLS_PORT}` (`853`) | `853` | DNS-over-TLS (DoT) |
| `0.0.0.0:${ADGUARD_HTTPS_PORT}` (`443`) | `443` | DNS-over-HTTPS (DoH) |

The admin UI is published directly **and** reachable via NPM (`adguardhome:3000` over `proxy_net`). To restrict any published port to loopback only (e.g. keep the admin UI behind the reverse proxy), prefix its mapping in `compose.yaml` with `127.0.0.1:`.

> **DoH port conflict.** `${ADGUARD_HTTPS_PORT}` defaults to `443`, which collides
> with the `npm` stack (Nginx Proxy Manager binds `0.0.0.0:443`). If you run both
> on the same host, set `ADGUARD_HTTPS_PORT` to a free port (e.g. `8443`) or drop
> the DoH mapping from `compose.yaml` if you don't need it.

---

## Volumes

| Host path (`.env` var) | Container path | Purpose |
|------------------------|----------------|---------|
| `${ADGUARD_WORK}` | `/opt/adguardhome/work` | Runtime state — query log, statistics, filter list cache |
| `${ADGUARD_CONF}` | `/opt/adguardhome/conf` | Persistent config (`AdGuardHome.yaml`), users, custom rules |

---

## Environment Variables

### `.env` (Compose interpolation)

| Variable | Example | Purpose |
|----------|---------|---------|
| `ADGUARD_WORK` | `/path/to/adguardhome/work` | Host bind mount for runtime state |
| `ADGUARD_CONF` | `/path/to/adguardhome/conf` | Host bind mount for configuration |
| `ADGUARD_WEBUI_PORT` | `3000` | Host port for the admin UI; also the healthcheck + NPM upstream target |
| `ADGUARD_TLS_PORT` | `853` | Host port for DNS-over-TLS (DoT) |
| `ADGUARD_HTTPS_PORT` | `443` | Host port for DNS-over-HTTPS (DoH) — see conflict note above |
| `ADGUARD_ADMIN_PORT` | `8080` | Reference only — not consumed by compose; optional alternate admin port set in `AdGuardHome.yaml` |

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme |
|----------|---------------|---------------|--------|
| `adguardhome.example.com` | `adguardhome` | `3000` | `http` |

---

## Deployment

```bash
cd /opt/stacks/adguardhome
cp .env.example .env
nano .env

docker compose up -d
```

### First-run setup

1. Open the app (e.g. `https://adguardhome.example.com` via NPM, or `http://<host-ip>:3000` directly) — the initial setup wizard runs on port `3000`.
2. In the wizard, set the **Admin Web Interface** to listen on `0.0.0.0:3000` (or whatever you've set for `${ADGUARD_WEBUI_PORT}`). This must match NPM's upstream port.
3. Set the **DNS server** to listen on `0.0.0.0:53`.
4. Create the admin account.
5. Point LAN clients (router, DHCP) at the host's IP for DNS.

After setup completes, AdGuard keeps responding on the admin port for the healthcheck and NPM upstream — no re-configuration needed if you stick with `3000`.

---

## Backup

Paths to include in Kopia / Borgmatic snapshots:

- `${ADGUARD_CONF}` — config, users, custom rules (essential for restore)
- `${ADGUARD_WORK}` — query log and stats (optional; will rebuild from filter sources on restore)

`AdGuardHome.yaml` in the conf directory is the single source of truth for restoring on a new host.

---

## Notes & Gotchas

- **Port 53 must be free on the host.** On Debian, this typically means disabling `systemd-resolved` (or stopping it from binding `127.0.0.53:53`) before bringing the stack up. `ss -ulpn | grep ':53'` will show what's holding it.
- **DoH (443) collides with NPM.** See the conflict note under Ports — change `ADGUARD_HTTPS_PORT` or drop the DoH mapping if you run NPM on the same host.
- **Admin port hardcoded to 3000 inside the container.** The compose healthcheck and NPM upstream both target `3000`. If you change AdGuard's admin port in the wizard, update the healthcheck and NPM together.
- **DoT (`853`) and DoH (`443`) are published** for upstream/encrypted DNS. If you don't use them, you can safely remove those two lines from `compose.yaml`.

---

## References

- [Upstream Docker Hub page](https://hub.docker.com/r/adguard/adguardhome)
- [Top-level README](../../README.md)
