# pdf

PDF toolkit using BentoPDF for browser-based PDF manipulation and conversion.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **BentoPDF** | Web-based PDF tools (merge, split, convert, etc.) | `ghcr.io/alam00000/bentopdf` |

## Network Architecture

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse proxy access |

## Ports

| Service | Host Port | Container Port | Notes |
|---------|-----------|----------------|-------|
| BentoPDF | `${BENTOPDF_PORT}` | 8080 | Web UI (localhost only) |

## Environment Variables

Create a `.env` file in the same directory as the compose file.

| Variable | Description | Example |
|----------|-------------|---------|
| `BENTOPDF_PORT` | BentoPDF web UI host port | `8080` |
| `TIMEZONE` | Timezone | `America/New_York` |

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

## Notes

- Port is bound to `127.0.0.1` for reverse proxy access only.
- Stateless service — no persistent volumes required.
