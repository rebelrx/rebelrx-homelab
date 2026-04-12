# audiobooks

Audiobook and podcast media server powered by Audiobookshelf, a self-hosted solution for managing and streaming your audio library.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **Audiobookshelf** | Audiobook/podcast media server with built-in player and library management | `ghcr.io/advplyr/audiobookshelf` |

## Network Architecture

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse proxy access and inter-service communication |

## Ports

| Service | Host Port | Container Port | Notes |
|---------|-----------|----------------|-------|
| Audiobookshelf | `${AUDIOBOOKSHELF_PORT}` | 80 | Web UI |

## Environment Variables

Create a `.env` file in the same directory as the compose file.

### Global

| Variable | Description | Example |
|----------|-------------|---------|
| `TIMEZONE` | Timezone | `America/New_York` |

### Audiobookshelf

| Variable | Description | Example |
|----------|-------------|---------|
| `AUDIOBOOKSHELF_PORT` | Audiobookshelf web UI host port | `13378` |

### Volume Paths

| Variable | Description |
|----------|-------------|
| `AUDIOBOOKS` | Audiobook library directory |
| `PODCASTS` | Podcast library directory |
| `AUDIOBOOKSHELF_CONFIG` | Configuration directory |
| `AUDIOBOOKSHELF_METADATA` | Metadata directory (covers, author images, cache) |

## Dependencies

None — this is a single-service stack with no external service dependencies.

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

- Container update monitoring is enabled via [What's Up Docker](https://github.com/fmartinou/whats-up-docker) labels (`wud.watch=true`, `wud.trigger.docker.enable=true`).
- Audiobookshelf supports audiobooks (M4B, MP3, etc.) and podcasts with automatic RSS feed management.
- Refer to the [Audiobookshelf docs](https://www.audiobookshelf.org/docs/) for library setup and configuration details.
