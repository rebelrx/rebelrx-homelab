# drawio

Self-hosted diagramming with [draw.io](https://www.drawio.com/) (diagrams.net) — open-source Visio/Lucidchart alternative.

Accessible via NPM (e.g. `https://drawio.example.com`) or directly on the published port.

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `drawio` | `jgraph/drawio:latest` | draw.io web app (Tomcat/Java) |

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | NPM reverse-proxy plane |

> draw.io has no database or cache sidecar — no internal network required.

---

## Ports

| Host port (`.env` var) | Container port | Bind | Purpose |
|------------------------|----------------|------|---------|
| `${DRAWIO_PORT}` (`8080`) | `8080` | all interfaces | Web UI (Tomcat) |

Published on all interfaces so the app is reachable on your LAN out of the box. To keep it behind a reverse proxy / loopback only, prefix the mapping in `compose.yaml` with `127.0.0.1:`. The container port `8080` is hardcoded in the jgraph/drawio image and not configurable.

---

## Volumes

draw.io is stateless — all diagram data lives in the browser or in the user's chosen cloud storage (Google Drive, OneDrive, etc.). No bind mounts or persistent volumes are required.

---

## Environment Variables

### `.env` (Compose interpolation)

| Variable | Example | Purpose |
|----------|---------|---------|
| `DRAWIO_BASE_URL` | `https://drawio.example.com` | Public base URL — used by draw.io to construct asset paths and export/share links. Must exactly match the URL the app is accessed from, including scheme. Mismatches break PNG/SVG exports and embedded diagram links. |
| `DRAWIO_PORT` | `8080` | Host port published for the web UI |

---

## Healthcheck

```
curl -fsS http://localhost:8080
interval: 1m30s | timeout: 10s | retries: 5 | start_period: 30s
```

> `start_period` is set to `30s` — draw.io runs on Tomcat (JVM), which typically takes 20–30s to fully initialise on first start. The healthcheck only counts failures after `start_period` elapses.

---

## Reverse Proxy

NPM proxy host:

| Field | Value |
|-------|-------|
| Forward hostname | `drawio` |
| Forward port | `8080` |
| WebSockets | On |

> Enable WebSockets — required for the real-time collaboration features in some draw.io modes.

---

## Backup

draw.io is stateless. There is nothing to back up in this stack.

If users are saving diagrams to the host filesystem via a custom storage backend, those paths would need to be added as bind mounts and included in Kopia / Borgmatic snapshots. The default configuration does not do this.

---

## Upgrades

```bash
cd /opt/stacks/drawio
docker compose pull
docker compose up -d
docker compose logs -f drawio
```

WUD (`wud.watch=true`) monitors the image and notifies on new releases. Updates are manual (`wud.trigger.docker.enable=false`).

---

## Notes & Gotchas

- **`DRAWIO_BASE_URL` must be exact.** The app uses it server-side to construct asset URLs. If it doesn't match the browser's address bar (e.g. scheme mismatch, trailing slash), exports and some UI assets will fail to load. If you access the app directly on `${DRAWIO_PORT}` rather than through a reverse proxy, set this to `http://<host-ip>:${DRAWIO_PORT}` accordingly.
- **Stateless by design.** draw.io does not persist diagrams server-side unless you configure an explicit storage backend (e.g. a self-hosted WebDAV or Nextcloud endpoint). Out of the box, users save `.drawio` / `.xml` files locally via browser download.
- **Cloud storage dialogs.** On first launch, draw.io presents integration options for Google Drive and OneDrive. These work as-is; they are OAuth flows that run entirely in the browser. To disable them and force local-only mode, a custom `drawio-config.json` can be mounted into the container — see the upstream docs for the configuration reference.
- **Image is `:latest`.** draw.io publishes versioned tags (e.g. `jgraph/drawio:26`) if you want to pin to a major version and avoid surprise upgrades. Swap to a version tag and update `wud.tag.include` label accordingly.

---

## References

- [draw.io GitHub (docker-drawio)](https://github.com/jgraph/docker-drawio)
- [draw.io configuration reference](https://www.drawio.com/doc/faq/configure-diagram-editor)
- [Top-level README](../../README.md)
