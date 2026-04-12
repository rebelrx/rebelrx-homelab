# hadockermon

Docker monitoring and control stack for Home Assistant integration, with a secure Docker socket proxy for API access.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **HA Dockermon** | REST API for Home Assistant to monitor/control Docker containers | `philhawthorne/ha-dockermon` |
| **Docker Socket Proxy** | Secure, filtered proxy for the Docker socket API | `tecnativa/docker-socket-proxy` |

## Ports

| Service | Host Port | Container Port | Notes |
|---------|-----------|----------------|-------|
| HA Dockermon | `${PORT_HADOCKERMON}` | 8126 | REST API |
| Docker Socket Proxy | `${PORT_DOCKERPROXY}` | 2375 | Docker API proxy |

## Environment Variables

Create a `.env` file in the same directory as the compose file.

| Variable | Description | Example |
|----------|-------------|---------|
| `PORT_HADOCKERMON` | HA Dockermon host port | `8126` |
| `PORT_DOCKERPROXY` | Docker Socket Proxy host port | `2375` |

### Docker Socket Proxy Capabilities

The following Docker API endpoints are enabled in the proxy:

| Capability | Enabled | Description |
|------------|---------|-------------|
| `CONTAINERS` | Yes | List and inspect containers |
| `INFO` | Yes | Docker system info |
| `IMAGES` | Yes | List images |
| `NETWORKS` | Yes | List networks |
| `VOLUMES` | Yes | List volumes |
| `SYSTEM` | Yes | System-level operations |
| `VERSION` | Yes | Docker version info |
| `STATS` | Yes | Container resource stats |

## Volumes

| Service | Container Path | Access | Purpose |
|---------|----------------|--------|---------|
| HA Dockermon | `/var/run/docker.sock` | Read-write | Docker socket access |
| Docker Socket Proxy | `/var/run/docker.sock` | **Read-only** | Filtered Docker socket access |

## Deployment

1. Create your `.env` file with all required variables.

2. Deploy:
   ```bash
   docker compose up -d
   ```

3. Configure Home Assistant to use the HA Dockermon API at `http://<host>:<PORT_HADOCKERMON>`.

4. Point any services needing Docker API access to the socket proxy at `http://<host>:<PORT_DOCKERPROXY>`.

## Notes

- HA Dockermon has not been actively maintained for some time. Consider evaluating alternatives for Home Assistant Docker integration.
- The Docker Socket Proxy mounts the socket as read-only (`:ro`). HA Dockermon mounts it read-write — add `:ro` if it only needs monitoring access.
- This stack does not use an external network.
