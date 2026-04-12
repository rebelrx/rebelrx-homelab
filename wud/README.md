# wud

Container update monitoring with What's Up Docker (WUD). Watches all containers for available image updates and sends email notifications via SMTP.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **What's Up Docker** | Container image update monitor with notifications | `getwud/wud` |

## Network Architecture

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse proxy access |

## Ports

| Service | Host Port | Container Port | Notes |
|---------|-----------|----------------|-------|
| WUD | `${WUD_PORT}` | 3000 | Web UI (localhost only) |

## Environment Variables

Create a `.env` file in the same directory as the compose file.

### General

| Variable | Description | Example |
|----------|-------------|---------|
| `WUD_PORT` | WUD web UI host port | `3000` |
| `TIMEZONE` | Timezone | `America/New_York` |

### SMTP Email Notifications

| Variable | Description | Example |
|----------|-------------|---------|
| `WUD_TRIGGER_SMTP_EMAIL_HOST` | SMTP server hostname | `smtp.example.com` |
| `WUD_TRIGGER_SMTP_EMAIL_PORT` | SMTP server port | `587` |
| `WUD_TRIGGER_SMTP_EMAIL_USER` | SMTP username | `user@example.com` |
| `WUD_TRIGGER_SMTP_EMAIL_PASS` | SMTP password | *(secret)* |
| `WUD_TRIGGER_SMTP_EMAIL_FROM` | Sender email address | `wud@example.com` |
| `WUD_TRIGGER_SMTP_EMAIL_TO` | Recipient email address | `admin@example.com` |

## Volumes

| Container Path | Purpose |
|----------------|---------|
| `/var/run/docker.sock` | Docker socket for monitoring running containers |

## Container Labels

WUD monitors containers based on labels set in their compose files. The standard labels used across all stacks:

| Label | Value | Description |
|-------|-------|-------------|
| `wud.watch` | `true` / `false` | Whether WUD should monitor this container |
| `wud.trigger.docker.enable` | `true` / `false` | Whether WUD should auto-update the container |

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

4. Access the WUD dashboard at the reverse proxy URL to see update status for all monitored containers.

## Notes

- Port is bound to `127.0.0.1` for reverse proxy access only.
- WUD is the only service with `wud.trigger.docker.enable=true` (auto-updates itself).
- TLS is disabled for SMTP (`WUD_TRIGGER_SMTP_EMAIL_TLS_ENABLED=false`). Enable if your SMTP server requires it.
- WUD monitors all containers with `wud.watch=true` labels across all Docker Compose stacks on the host.
