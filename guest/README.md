# guest

Tailscale sidecar containers for exposing select services to guest users on a separate tailnet. Currently provides guest access to RomM.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **Tailscale (RomM)** | VPN sidecar providing guest tailnet access to RomM | `tailscale/tailscale` |

## Network Architecture

| Network | Type | Purpose |
|---------|------|---------|
| `guest_net` | External | Connects Tailscale sidecar to RomM |

## Ports

No ports are exposed directly. Traffic is routed through the Tailscale tunnel and served via `TS_SERVE_CONFIG`.

## Environment Variables

Create a `.env` file in the same directory as the compose file.

### Tailscale

| Variable | Description | Example |
|----------|-------------|---------|
| `TS_AUTHKEY` | Tailscale auth key for the guest tailnet | *(secret)* |

### Volume Paths

| Variable | Description |
|----------|-------------|
| `TAILSCALE_ROMM_STATE` | Tailscale state data (persists node identity across restarts) |
| `SERVE_CONFIG` | Path to Tailscale Serve configuration file (`serve.json`) |

## Capabilities

| Capability | Reason |
|------------|--------|
| `NET_ADMIN` | Required for Tailscale to manage network interfaces |
| `NET_RAW` | Required for Tailscale to send/receive raw packets |

## Deployment

1. Create the external network if it doesn't exist:
   ```bash
   docker network create guest_net
   ```

2. Generate an auth key from the guest tailnet in the [Tailscale admin console](https://login.tailscale.com/admin/settings/keys).

3. Create a `serve.json` for the RomM proxy target (adjust container name and port to match your RomM setup):
   ```json
   {
     "TCP": {
       "443": {
         "HTTPS": true
       }
     },
     "Web": {
       "${TS_CERT_DOMAIN}:443": {
         "Handlers": {
           "/": {
             "Proxy": "http://romm:8080"
           }
         }
       }
     }
   }
   ```

4. Create your `.env` file with all required variables.

5. Deploy:
   ```bash
   docker compose up -d
   ```

## Notes

- The `TS_AUTHKEY` should be generated from the guest tailnet (not the primary homelab tailnet) to maintain network separation.
- RomM must also be attached to the `guest_net` network for the sidecar to reach it.
- Tailscale state is persisted so the node survives container restarts without needing a new auth key.
- Additional guest services can be added by creating more Tailscale sidecar containers in this stack with their own state directories and serve configs.
