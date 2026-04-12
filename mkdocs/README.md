# mkdocs

Documentation site stack using Material for MkDocs for authoring and Nginx for serving the built static site.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **MkDocs** | Material for MkDocs site publisher | `squidfunk/mkdocs-material` |
| **Nginx** | Lightweight webserver for serving the built site | `nginx:alpine` |

## Network Architecture

Both services connect to the external `proxy_net` network. The MkDocs dev server port is bound to `127.0.0.1` for reverse proxy access only.

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse proxy access |

## Ports

| Service | Host Port | Container Port | Notes |
|---------|-----------|----------------|-------|
| MkDocs | `${MKDOCS_PORT}` | 8000 | Dev server (localhost only) |

## Environment Variables

Create a `.env` file in the same directory as the compose file.

### Service Ports

| Variable | Description |
|----------|-------------|
| `MKDOCS_PORT` | MkDocs dev server host port |

### Volume Paths

| Variable | Description |
|----------|-------------|
| `MKDOCS_DOCS` | MkDocs source documentation directory |
| `MKDOCS_SITE` | Built static site directory (shared with Nginx) |

## Deployment

1. Create the external network if it doesn't exist:
   ```bash
   docker network create proxy_net
   ```

2. Create your `.env` file with all required variables.

3. Deploy:
   ```bash
   docker compose up -d
   ```

4. The MkDocs container watches the source docs and rebuilds on changes. The Nginx container serves the built output from the shared `MKDOCS_SITE` volume.

## Notes

- The MkDocs container runs with `stdin_open` and `tty` enabled for interactive use.
- Nginx mounts the built site directory as read-only (`:ro`).
- The MkDocs dev server is useful for live preview during editing; Nginx serves the production build.
- MkDocs WUD trigger is set to `enable=true` (auto-update enabled), unlike most other stacks.
