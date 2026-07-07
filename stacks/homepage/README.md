# homepage

Service dashboard with [Homepage](https://gethomepage.dev/) — a single-page launcher that pulls live status from your containers, integrations, and APIs.

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `homepage` | `ghcr.io/gethomepage/homepage:latest` | Dashboard, service status widgets, container discovery |

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse-proxy access via NPM |

---

## Ports

| Host port (`.env` var) | Container port | Bind | Purpose |
|------------------------|----------------|------|---------|
| `${HOMEPAGE_PORT}` (`3000`) | `3000` | all interfaces | Web UI |

Published on all interfaces so the UI is reachable on your LAN out of the box. To keep it behind a reverse proxy / loopback only, prefix the mapping in `compose.yaml` with `127.0.0.1:`. `3000` is a common port — change `HOMEPAGE_PORT` if it clashes on your host.

> **Direct-IP access needs an allowlist entry.** Homepage validates the request `Host:` header against `HOMEPAGE_ALLOWED_HOSTS`. If you reach it directly at `http://<host-ip>:3000`, add that IP to `HOMEPAGE_ALLOWED_HOSTS` or you'll get a blank page / 403 — see [Environment Variables](#environment-variables).

---

## Volumes

| Host path (`.env` var) | Container path | Mode | Purpose |
|------------------------|----------------|------|---------|
| `${HOMEPAGE_CONFIG}` | `/app/config` | rw | All YAML config files (services, widgets, bookmarks, settings, docker integration) |
| `${HOMEPAGE_ICONS}` | `/app/public/icons` | rw | Custom icons surfaced in service cards via `homepage.icon=<filename>` |
| `/var/run/docker.sock` | `/var/run/docker.sock` | **ro** | Docker daemon socket — read-only; Homepage only inspects container metadata, never writes |

---

## Environment Variables

### `.env` (Compose interpolation)

| Variable | Example | Purpose |
|----------|---------|---------|
| `TIMEZONE` | `America/New_York` | Container TZ |
| `HOMEPAGE_PORT` | `3000` | Host port published for the web UI |
| `HOMEPAGE_CONFIG` | `/path/to/homepage/config` | Host bind for YAML config files |
| `HOMEPAGE_ICONS` | `/path/to/homepage/icons` | Host bind for custom icons mounted at `/app/public/icons` |
| `HOMEPAGE_ALLOWED_HOSTS` | `homepage.example.com,localhost,127.0.0.1` | **Required.** Comma-separated list of `Host:` headers Homepage will accept. Anything else returns 403 / blank page. Add your host's LAN IP if accessing directly |
| `HOMEPAGE_VAR_*` | *(secrets)* | Widget credentials passed through to YAML configs via `{{HOMEPAGE_VAR_*}}` substitution. See [Widget secrets via env vars](#widget-secrets-via-env-vars) |

> The `compose.yaml` uses `env_file: .env` so **every** variable in `.env` is exported into the container's environment — not just the ones referenced under `environment:`. That's what makes the `HOMEPAGE_VAR_*` passthrough work without listing each one in compose.

---

## Configuration

Homepage's behavior is driven entirely by YAML files inside `${HOMEPAGE_CONFIG}`. The container auto-generates empty templates on first start if the directory is empty, but for a homelab you almost certainly want to author and version-control these yourself.

| File | Purpose |
|------|---------|
| `settings.yaml` | Global UI settings (theme, layout, language, providers) |
| `services.yaml` | Service cards grouped into rows/columns — your main dashboard |
| `widgets.yaml` | Info widgets at the top (search, weather, resources, datetime) |
| `bookmarks.yaml` | Bookmark groups and links |
| `docker.yaml` | Docker integration config (enables container-aware status, autodiscovery via labels) |
| `kubernetes.yaml` | K8s integration (unused on this homelab — leave empty) |
| `custom.css` / `custom.js` | Optional theming and behavior overrides |

> Full schema reference: [Homepage docs — configs](https://gethomepage.dev/configs/).

### Custom icons

Drop image files into `${HOMEPAGE_ICONS}` (PNG, SVG, etc.) and reference them by filename in a container label or `services.yaml` entry:

```yaml
- homepage.icon=my-custom-icon.png
```

Homepage already ships with the Dashboard Icons set built in, so this mount is only needed for icons that aren't already covered.

### Widget secrets via env vars

Widget definitions in `services.yaml` typically include credentials (API keys, usernames, passwords) for the upstream services they query — Proxmox, Portainer, Jellyfin, Immich, etc. Committing those secrets to YAML defeats the point of keeping the config in version control.

Homepage supports `{{HOMEPAGE_VAR_*}}` substitution: any env var prefixed with `HOMEPAGE_VAR_` is interpolated into YAML at load time. Define secrets in `.env`:

```bash
HOMEPAGE_VAR_PROXMOX_USERNAME=root@pam
HOMEPAGE_VAR_PROXMOX_PASSWORD=<password>
HOMEPAGE_VAR_PROXMOX_NODE=pve
```

Then reference them in `services.yaml`:

```yaml
- Proxmox:
    icon: proxmox.png
    href: https://proxmox.example.com
    widget:
      type: proxmox
      url: https://proxmox.example.com
      username: {{HOMEPAGE_VAR_PROXMOX_USERNAME}}
      password: {{HOMEPAGE_VAR_PROXMOX_PASSWORD}}
      node: {{HOMEPAGE_VAR_PROXMOX_NODE}}
```

The YAML stays safe to commit; the secrets live only in `.env`. See `.env.example` for the full set of widget variables currently defined for this homelab.

> The `compose.yaml` mounts `.env` via `env_file:` — meaning every variable in it is exported into the container automatically. No need to enumerate each `HOMEPAGE_VAR_*` under `environment:`.

### Container autodiscovery

With `docker.yaml` defining the local Docker socket as a "server", Homepage can auto-populate service cards from container labels — no need to hand-maintain `services.yaml` entries for every stack. Add labels like these to any container you want Homepage to surface:

```yaml
    labels:
      - homepage.group=Media
      - homepage.name=Jellyfin
      - homepage.icon=jellyfin.png
      - homepage.href=https://jellyfin.example.com
      - homepage.description=Media server
      - homepage.widget.type=jellyfin
      - homepage.widget.url=http://jellyfin:8096
      - homepage.widget.key={{HOMEPAGE_VAR_JELLYFIN_KEY}}
```

This is the recommended pattern over editing `services.yaml` by hand — the dashboard self-updates as you add or remove stacks. The `{{HOMEPAGE_VAR_*}}` substitution works inside container labels too.

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme |
|----------|---------------|---------------|--------|
| `homepage.example.com` | `homepage` | `3000` | `http` |

> Homepage validates the inbound `Host:` header against `HOMEPAGE_ALLOWED_HOSTS`. If NPM passes a hostname not in that list, Homepage returns a blank page or 403. Keep them in sync — if you add another hostname in NPM, also add it (comma-separated) to `HOMEPAGE_ALLOWED_HOSTS`.

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing)
- `${HOMEPAGE_CONFIG}` and `${HOMEPAGE_ICONS}` exist on the host (the container will populate the config dir with template files on first start)

### Bring up

```bash
cd /opt/stacks/homepage
cp .env.example .env
nano .env  # set HOMEPAGE_ALLOWED_HOSTS + any HOMEPAGE_VAR_* widget secrets

mkdir -p /path/to/homepage/config
mkdir -p /path/to/homepage/icons

chmod 600 .env

docker compose up -d
```

### First-run setup

1. On first start, Homepage writes default template YAML files into `${HOMEPAGE_CONFIG}` if missing. Edit them to taste.
2. Open the app (e.g. `https://homepage.example.com` via NPM, or `http://<host-ip>:3000` directly). If you see a blank page or 403, check `HOMEPAGE_ALLOWED_HOSTS` includes the hostname (or IP) in your browser's address bar.
3. Add container labels to existing stacks (see [Container autodiscovery](#container-autodiscovery) above) to populate service cards automatically.
4. For each widget you want live status from, populate the corresponding `HOMEPAGE_VAR_*` in `.env`, restart Homepage (`docker compose restart homepage`), and reference the var in your YAML or labels.

---

## Backup

Paths to include in Kopia / Borgmatic snapshots:

- `${HOMEPAGE_CONFIG}` — every YAML file and any custom CSS/JS
- `${HOMEPAGE_ICONS}` — custom icon files (small, but easy to forget)
- `.env` — now holds widget secrets (`HOMEPAGE_VAR_*`); losing it means re-pulling every API key from each upstream service

Homepage is otherwise stateless — wiping the container and restoring config + `.env` is a complete recovery.

> Consider keeping `${HOMEPAGE_CONFIG}` in your homelab git repo as `stacks/homepage/config/`, with secrets (API keys for widgets) substituted at deploy time via `{{HOMEPAGE_VAR_*}}`. The YAML is small, change-tracked, and survives a full host loss — and as long as `.env` stays out of git, no credentials leak. **If you do commit the config publicly, scrub real hostnames/URLs from the YAML too, and confirm every credential uses `{{HOMEPAGE_VAR_*}}` rather than a literal value.**

---

## Notes & Gotchas

- **`HOMEPAGE_ALLOWED_HOSTS` is mandatory.** Without it (or with a value that doesn't match the request `Host:` header), Homepage returns blank/403. This was added in 2024 to prevent open-proxy / DNS-rebinding abuse. Every additional hostname Homepage should serve (alternate domains, the LAN IP, etc.) needs to be listed comma-separated.
- **`HOMEPAGE_VAR_*` requires a restart.** Adding or changing a widget variable in `.env` won't be picked up until Homepage is restarted (`docker compose restart homepage`). Reloading the YAML config alone isn't enough — the env vars are read at process start.
- **`env_file` exposes everything.** Because `compose.yaml` uses `env_file: .env`, every variable in `.env` is exported to the container — not just the ones referenced under `environment:`. That's by design for `HOMEPAGE_VAR_*` passthrough, but it also means anything you put in `.env` (even unrelated comments-as-vars) ends up inside the container.
- **RO docker socket.** Homepage only reads container metadata; never give it RW. This is one of the stacks that's already correct per the homelab convention.
- **No healthcheck.** Homepage is a dashboard — nothing depends on it being healthy. Adding a healthcheck would be pure cosmetic noise.

---

## References

- [Homepage docs — Docker install](https://gethomepage.dev/installation/docker/)
- [Homepage docs — configuration](https://gethomepage.dev/configs/)
- [Homepage docs — Docker integration & labels](https://gethomepage.dev/configs/docker/)
- [Homepage docs — environment variable substitution](https://gethomepage.dev/configs/services/#using-environment-variables)
- [Top-level README](../../README.md)
