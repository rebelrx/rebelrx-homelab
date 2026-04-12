# books

Book and reading management stack with Calibre for library management, Calibre-Web for browser-based reading, and Kavita as a comic/book reader.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **Calibre** | Desktop-class library management with web access | `lscr.io/linuxserver/calibre` |
| **Calibre-Web** | Web-based reader interface for Calibre libraries | `lscr.io/linuxserver/calibre-web` |
| **Kavita** | Modern comic and book reader with OPDS support | `lscr.io/linuxserver/kavita` |

## Network Architecture

All services connect to the external `proxy_net` network. All ports are bound to `127.0.0.1` for reverse proxy access only.

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse proxy access |

## Ports

| Service | Host Port | Container Port | Notes |
|---------|-----------|----------------|-------|
| Calibre HTTP | `${CALIBRE_PORT_HTTP}` | 8080 | Desktop GUI (localhost only) |
| Calibre HTTPS | `${CALIBRE_PORT_HTTPS}` | 8181 | Desktop GUI HTTPS (localhost only) |
| Calibre Webserver | `${CALIBRE_PORT_WEBSERVER}` | 8081 | Webserver GUI (localhost only) |
| Calibre-Web | `${CALIBREWEB_PORT}` | 8083 | Web UI (localhost only) |
| Kavita | `${KAVITA_PORT}` | 5000 | Web UI (localhost only) |

## Environment Variables

Create a `.env` file in the same directory as the compose file.

### Global

| Variable | Description | Example |
|----------|-------------|---------|
| `PUID` | User ID for file permissions | `1000` |
| `PGID` | Group ID for file permissions | `1000` |
| `TIMEZONE` | Timezone | `America/New_York` |

### Service Ports

| Variable | Description |
|----------|-------------|
| `CALIBRE_PORT_HTTP` | Calibre HTTP host port |
| `CALIBRE_PORT_HTTPS` | Calibre HTTPS host port |
| `CALIBRE_PORT_WEBSERVER` | Calibre webserver host port |
| `CALIBREWEB_PORT` | Calibre-Web host port |
| `KAVITA_PORT` | Kavita host port |

### Volume Paths

| Variable | Description |
|----------|-------------|
| `CALIBRE_CONFIG` | Calibre config directory |
| `CALIBRE_LIBRARY` | Calibre book library (shared with Calibre-Web) |
| `CALIBRE_PLUGINS` | Calibre plugins directory |
| `CALIBREWEB_CONFIG` | Calibre-Web config directory |
| `KAVITA_CONFIG` | Kavita config directory |
| `KAVITA_LIBRARY` | Kavita library directory |

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

4. Calibre-Web shares the same `CALIBRE_LIBRARY` volume as Calibre, so books managed in Calibre are automatically available in Calibre-Web.

## Notes

- Calibre's webserver GUI (port 8081) must be enabled in the Calibre desktop GUI settings before it will respond.
- Calibre uses `shm_size: 1gb` for its desktop rendering environment.
- Calibre-Web includes the `universal-calibre` Docker mod for ebook conversion support.
