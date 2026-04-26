# portainer

Self-hosted Docker management UI. Portainer Business Edition (free for up to 3 nodes) provides a web interface for managing Docker containers, images, networks, and volumes.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **Portainer** | Docker container management UI | `portainer/portainer-ee:sts` |

## Network Architecture

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse proxy access to Portainer |

## Ports

| Service | Host Port | Container Port | Notes |
|---------|-----------|----------------|-------|
| Portainer | `${PORTAINER_PORT}` | 9443 | Web UI (HTTPS, localhost only) |

## Environment Variables

Create a `.env` file in the same directory as the compose file.

### General

| Variable | Description | Example |
|----------|-------------|---------|
| `PORTAINER_PORT` | Portainer web UI host port | `9443` |

### Portainer

| Variable | Description | Example |
|----------|-------------|---------|
| `PORTAINER_DATA` | Host path for Portainer persistent data | `/srv/portainer/data` |

## Volumes

| Service | Container Path | Purpose |
|---------|----------------|---------|
| Portainer | `/var/run/docker.sock` | Docker socket for managing the host's Docker daemon |
| Portainer | `/data` | Portainer configuration, users, settings, and edge agent data |

## Security

Portainer requires access to the Docker socket (`/var/run/docker.sock`) to manage containers, which effectively grants it root-equivalent privileges on the host. Mitigations in this stack:

- Web UI is bound to `127.0.0.1` only, accessible exclusively via reverse proxy.
- Uses Portainer's built-in HTTPS on port 9443 (self-signed certificate by default).
- Strong admin password and 2FA should be configured on first login.

## Container Labels

To enable update monitoring via the [wud](../wud/) stack, add the following to the service:

```yaml
labels:
  - wud.watch=true
  - wud.tag.include=^sts$
```

The `sts` tag tracks the Short-Term Support release channel; the `wud.tag.include` filter prevents WUD from suggesting unrelated tags.

## Deployment

1. Create the external network if it doesn't exist:
   ```bash
   docker network create proxy_net
   ```

2. Create the host directory for the bind-mounted volume:
   ```bash
   mkdir -p /srv/portainer/data
   ```

3. Create your `.env` file with all required variables.

4. Deploy:
   ```bash
   docker compose up -d
   ```

5. Access Portainer at the reverse proxy URL and complete the initial admin setup within 5 minutes of first start (Portainer locks itself if the admin account isn't created in time).

## Notes

- Portainer's port is bound to `127.0.0.1` for reverse proxy access only.
- The `portainer-ee:sts` image is the Business Edition on the Short-Term Support channel; it is free for environments with 3 or fewer nodes and requires a free license key entered on first login. For the fully open-source build, use `portainer/portainer-ce:sts` instead.
- Mounting the Docker socket grants the container full control over the Docker daemon — treat Portainer access as equivalent to root access on the host.
- To manage additional Docker hosts, deploy the Portainer Edge Agent on remote nodes rather than exposing their Docker sockets directly.
