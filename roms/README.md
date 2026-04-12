# roms

ROM library management with RomM, a game collection organizer with metadata scraping from multiple providers. Backed by MariaDB.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **RomM** | ROM management web application with metadata scraping | `rommapp/romm` |
| **MariaDB** | Database backend | `mariadb` |

## Network Architecture

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse proxy access and inter-service communication |

## Ports

| Service | Host Port | Container Port | Notes |
|---------|-----------|----------------|-------|
| RomM | `${ROMM_PORT}` | 8080 | Web UI |

## Environment Variables

Create a `.env` file in the same directory as the compose file.

### Global

| Variable | Description | Example |
|----------|-------------|---------|
| `TIMEZONE` | Timezone | `America/New_York` |

### RomM

| Variable | Description | Example |
|----------|-------------|---------|
| `ROMM_PORT` | RomM web UI host port | `8080` |
| `ROMM_AUTH_SECRET_KEY` | Auth secret (generate with `openssl rand -hex 32`) | *(secret)* |

### Database

| Variable | Description | Example |
|----------|-------------|---------|
| `MARIADB_ROOT_PASSWORD` | MariaDB root password | *(secret)* |
| `MARIADB_DATABASE` | Database name | `romm` |
| `MARIADB_USER` | Database username | `romm` |
| `MARIADB_PASSWD` | Database password | *(secret)* |

### Metadata Providers

| Variable | Description | Source |
|----------|-------------|--------|
| `IGDB_CLIENT_ID` | IGDB API client ID | [IGDB docs](https://docs.romm.app/latest/Getting-Started/Metadata-Providers/#igdb) |
| `IGDB_CLIENT_SECRET` | IGDB API client secret | IGDB |
| `SCREENSCRAPER_USER` | ScreenScraper username | [ScreenScraper docs](https://docs.romm.app/latest/Getting-Started/Metadata-Providers/#screenscraper) |
| `SCREENSCRAPER_PASSWORD` | ScreenScraper password | ScreenScraper |
| `MOBYGAMES_API_KEY` | MobyGames API key | [MobyGames docs](https://docs.romm.app/latest/Getting-Started/Metadata-Providers/#mobygames) |
| `RETROACHIEVEMENTS_API_KEY` | RetroAchievements API key | [RetroAchievements docs](https://docs.romm.app/latest/Getting-Started/Metadata-Providers/#retroachievements) |
| `STEAMGRIDDB_API_KEY` | SteamGridDB API key | [SteamGridDB docs](https://docs.romm.app/latest/Getting-Started/Metadata-Providers/#steamgriddb) |

### Volume Paths

| Variable | Description |
|----------|-------------|
| `ROMM_RESOURCES` | Metadata resources (covers, screenshots) |
| `ROMM_DATA` | Redis cache data for background tasks |
| `ROMM_LIBRARY` | Game ROM library |
| `ROMM_ASSETS` | Uploaded saves, states, etc. |
| `ROMM_CONFIG` | Config directory (config.yml) |
| `ROMM_MYSQL_DATA` | MariaDB data directory |

## Enabled APIs

The following metadata APIs are enabled by default in the compose file: Hasheous, PlayMatch, LaunchBox (with scheduled metadata updates at 4:00 AM), Flashpoint, and HowLongToBeat.

## Dependencies

```
MariaDB (romm-db) ──► RomM
```

RomM waits for MariaDB's healthcheck to pass before starting. The `restart: true` flag on the dependency means RomM will restart if MariaDB restarts.

## Deployment

1. Create the external network if it doesn't exist:
   ```bash
   docker network create proxy_net
   ```

2. Create your `.env` file with all required variables.

3. Set up your ROM library following the [RomM folder structure docs](https://docs.romm.app/latest/Getting-Started/Folder-Structure/).

4. Deploy:
   ```bash
   docker compose up -d
   ```

## Notes

- Filesystem change monitoring is enabled (`ENABLE_RESCAN_ON_FILESYSTEM_CHANGE`) along with scheduled rescans.
- LaunchBox metadata updates are scheduled via cron (`0 4 * * *` — daily at 4:00 AM).
