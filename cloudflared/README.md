# cloudflared

Cloudflare Tunnel client for securely exposing services to the internet without opening inbound ports.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **Cloudflared** | Cloudflare Tunnel daemon | `cloudflare/cloudflared:latest` |

## Network Architecture

The service connects to the external `proxy_net` network to reach proxied services.

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Access to proxied services |

## Ports

No ports are exposed. Cloudflared establishes an outbound-only connection to Cloudflare's edge network.

## Environment Variables

Create a `.env` file in the same directory as the compose file.

### Tunnel Configuration

| Variable | Description |
|----------|-------------|
| `TUNNEL_TOKEN` | Cloudflare Tunnel token from the Zero Trust dashboard |

## Deployment

1. Create the external network if it doesn't exist:
   ```bash
   docker network create proxy_net
   ```

2. Create a tunnel in the [Cloudflare Zero Trust dashboard](https://one.dash.cloudflare.com/) and copy the tunnel token.

3. Create your `.env` file with the tunnel token.

4. Deploy:
   ```bash
   docker compose up -d
   ```

## Notes

- The tunnel token is passed via environment variable and interpolated by Docker Compose.
- Cloudflared requires no inbound ports — all connections are initiated outbound to Cloudflare's edge.
- Public hostnames and routing rules are configured in the Cloudflare Zero Trust dashboard, not in the compose file.
