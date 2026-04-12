# filebrowser

Web-based file manager using Filebrowser Quantum for browsing and managing files on the host.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **Filebrowser Quantum** | Modern web-based file manager | `gtstef/filebrowser` |

## Network Architecture

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse proxy access |

## Ports

| Service | Host Port | Container Port | Notes |
|---------|-----------|----------------|-------|
| Filebrowser | `${FILEBROWSER_PORT}` | 80 | Web UI (localhost only) |

## Environment Variables

Create a `.env` file in the same directory as the compose file.

| Variable | Description | Example |
|----------|-------------|---------|
| `FILEBROWSER_PORT` | Filebrowser web UI host port | `80` |
| `TIMEZONE` | Timezone | `America/New_York` |
| `FILEBROWSER_FOLDER_PATH` | Host path to browse | `/mnt/data` |
| `FILEBROWSER_DATA_PATH` | Filebrowser application data | `/opt/filebrowser/data` |
| `FILEBROWSER_TMP_PATH` | Filebrowser temp directory | `/opt/filebrowser/tmp` |

## Volumes

| Container Path | Purpose |
|----------------|---------|
| `/folder` | Host directory to browse and manage |
| `/home/filebrowser/data` | Application data and config |
| `/home/filebrowser/tmp` | Temporary files |

## Deployment

1. Create the external network if it doesn't exist:
   ```bash
   docker network create proxy_net
   ```

2. Create your `.env` file with all required variables.

3. Create a `config.yaml` in your `FILEBROWSER_DATA_PATH` directory (see [Filebrowser docs](https://github.com/gtsteffaniak/filebrowser/wiki/Getting-Started)).

4. Deploy:
   ```bash
   docker compose up -d
   ```

## Notes

- The `FILEBROWSER_CONFIG` environment variable points to `data/config.yaml` inside the container, which maps to your `FILEBROWSER_DATA_PATH`.
- Port is bound to `127.0.0.1` for reverse proxy access only.
