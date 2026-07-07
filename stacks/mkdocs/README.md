# mkdocs

Documentation site stack using [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) for authoring and Nginx for serving the built static site.

Two containers: `mkdocs` runs the live-preview dev server (auto-rebuilds as you edit), and `mkdocs-web` serves the built static output via Nginx. The stack builds a small custom image (see [Custom image](#custom-image)) to add the i18n plugin.

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `mkdocs` | `squidfunk/mkdocs-material` (built locally — see `Dockerfile`) | Material for MkDocs live-preview dev server |
| `mkdocs-web` | `nginx:alpine` | Serves the built static site read-only |

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse-proxy access via NPM |

> Both services join `proxy_net`. NPM can reach either by container name (`mkdocs:8000` for live preview, `mkdocs-web:80` for the built site).

---

## Ports

| Service | Host port (`.env` var) | Container port | Bind | Purpose |
|---------|------------------------|----------------|------|---------|
| `mkdocs` | `${MKDOCS_PORT}` (`8005`) | `8000` | all interfaces | Live-preview dev server |
| `mkdocs-web` | `${MKDOCS_WEB_PORT}` (`8006`) | `80` | all interfaces | Built static site (Nginx) |

Both are published on all interfaces so they're reachable on your LAN out of the box. To keep either behind a reverse proxy / loopback only, prefix its mapping in `compose.yaml` with `127.0.0.1:`.

> Typically you use **one or the other**: the `mkdocs` dev server while authoring (live reload), or `mkdocs-web` for the built production site. Both are published for flexibility.

---

## Volumes

| Host path (`.env` var) | Container path | Mode | Purpose |
|------------------------|----------------|------|---------|
| `${MKDOCS_PROJECT_PATH}` | `/docs` | rw | MkDocs project root — `mkdocs.yml`, the `docs/` source tree, and the built `site/` output |
| `${MKDOCS_SITE}` | `/usr/share/nginx/html` | **ro** | Built static site, served read-only by Nginx |

---

## Environment Variables

### `.env` (Compose interpolation)

| Variable | Example | Purpose |
|----------|---------|---------|
| `MKDOCS_PORT` | `8005` | Host port for the live-preview dev server (container `8000`) |
| `MKDOCS_WEB_PORT` | `8006` | Host port for the Nginx-served built site (container `80`) |
| `MKDOCS_PROJECT_PATH` | `/path/to/mkdocs/project` | MkDocs project root, mounted at `/docs` |
| `MKDOCS_SITE` | `/path/to/mkdocs/project/site` | Built `site/` output, served by Nginx |

---

## Custom image

The `mkdocs` service builds a local image (`build: .`) from the committed `Dockerfile`, which extends the upstream image with an extra plugin:

```dockerfile
FROM squidfunk/mkdocs-material

RUN pip install \
    mkdocs-static-i18n
```

Add more MkDocs plugins to the `pip install` line and rebuild (`docker compose build mkdocs`) as needed.

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing)
- An existing MkDocs project at `${MKDOCS_PROJECT_PATH}` (containing `mkdocs.yml` and a `docs/` folder) — or create one with `mkdocs new .` inside that dir

### Bring up

```bash
cd /opt/stacks/mkdocs
cp .env.example .env
nano .env   # set MKDOCS_PROJECT_PATH and MKDOCS_SITE
chmod 600 .env

docker compose build
docker compose up -d
```

### Usage

1. **Author with live preview:** open the dev server (e.g. `http://<host-ip>:8005`, or via NPM). Edits to files under `${MKDOCS_PROJECT_PATH}/docs` reload automatically.
2. **Build the static site:** `docker compose exec mkdocs mkdocs build` writes the output to `${MKDOCS_PROJECT_PATH}/site` (which is `${MKDOCS_SITE}`).
3. **Serve the built site:** Nginx (`mkdocs-web`) serves whatever is in `${MKDOCS_SITE}` — reachable at `http://<host-ip>:8006` or via NPM.

---

## Backup

Include in Kopia / Borgmatic snapshots:

- `${MKDOCS_PROJECT_PATH}` — your docs source and `mkdocs.yml` (the only thing worth keeping; the built `site/` is regenerable)
- `compose.yaml`, `Dockerfile`, `.env`, `.env.example`

The built site and the container images are fully reproducible from the source + `Dockerfile`.

---

## Notes & Gotchas

- **WUD is notify-only here (`wud.trigger.docker.enable=false`).** This was changed from auto-update: because `mkdocs` uses `build: .`, an auto-pull would replace the locally-built image (with the i18n plugin) with the stock upstream image, dropping your custom build. WUD still *notifies* when the base image updates so you can rebuild manually (`docker compose build mkdocs`).
- **`mkdocs` runs with `stdin_open` + `tty`.** Enables interactive use (e.g. `docker compose exec mkdocs mkdocs ...`).
- **Nginx mounts the site read-only (`:ro`).** It only serves; it never writes. The build output is produced by the `mkdocs` container.
- **Live-preview vs production.** The dev server (`mkdocs`) is for editing (live reload, slower); Nginx (`mkdocs-web`) serves the built static output (fast, production). Point NPM at whichever role you want public.

---

## References

- [Material for MkDocs — getting started](https://squidfunk.github.io/mkdocs-material/getting-started/)
- [MkDocs docs](https://www.mkdocs.org/)
- [mkdocs-static-i18n plugin](https://github.com/ultrabug/mkdocs-static-i18n)
- [Top-level README](../../README.md)
