# dns

DNS filtering and ad blocking with AdGuard Home. Provides network-wide ad/tracker blocking, query logging, and a web-based admin UI.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **AdGuard Home** | DNS filter with ad/tracker blocking and query logging | `adguard/adguardhome` |

## Network Architecture

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse proxy access to admin UI |

## Ports

| Service | Host Port | Container Port | Notes |
|---------|-----------|----------------|-------|
| AdGuard Home | `53` | 53 (TCP/UDP) | DNS — bound to all interfaces for network-wide access |
| AdGuard Home | `${ADGUARD_WEBUI_PORT}` | 3000 | Initial setup wizard |
| AdGuard Home | `${ADGUARD_ADMIN_PORT}` | 80 | Admin web UI |
| AdGuard Home | `${ADGUARD_TLS_PORT}` | 853 | DNS-over-TLS |
| AdGuard Home | `${ADGUARD_HTTPS_PORT}` | 443 | DNS-over-HTTPS |

## Environment Variables

Create a `.env` file in the same directory as the compose file.

### Ports

| Variable | Description | Example |
|----------|-------------|---------|
| `ADGUARD_WEBUI_PORT` | Setup wizard host port | `3000` |
| `ADGUARD_ADMIN_PORT` | Admin panel host port | `8080` |
| `ADGUARD_TLS_PORT` | DNS-over-TLS host port | `853` |
| `ADGUARD_HTTPS_PORT` | DNS-over-HTTPS host port | `4443` |

### Volume Paths

| Variable | Description |
|----------|-------------|
| `ADGUARD_WORK` | AdGuard working data (stats, query log) |
| `ADGUARD_CONF` | AdGuard configuration |

## Deployment

1. Create the external network if it doesn't exist:
   ```bash
   docker network create proxy_net
   ```

2. Verify port 53 is not already in use:
   ```bash
   sudo ss -tulnp | grep :53
   ```

3. If `systemd-resolved` occupies port 53, disable its stub listener:
   ```bash
   sudo sed -i 's/#DNSStubListener=yes/DNSStubListener=no/' /etc/systemd/resolved.conf
   sudo systemctl restart systemd-resolved
   ```

4. Create your `.env` file with all required variables.

5. Deploy:
   ```bash
   docker compose up -d
   ```

6. Access the setup wizard at `http://<host-ip>:${ADGUARD_WEBUI_PORT}` to configure admin credentials, upstream DNS servers, and blocklists.

## Notes

- Port 53 is bound to all interfaces (not localhost) so DNS queries can reach the server from other devices and containers.
- The setup wizard port (3000) is only needed for initial configuration and can be removed from the compose file after setup.
- Tailscale can be configured to use AdGuard Home as the DNS server for all tailnet devices via the Tailscale admin console under DNS → Custom nameservers with "Override local DNS" enabled.
- After initial setup, the admin panel is available at `http://<host-ip>:${ADGUARD_ADMIN_PORT}`.
