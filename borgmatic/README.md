# borgmatic

Local backup stack powered by [Borgmatic](https://torsion.org/borgmatic/), a wrapper around [BorgBackup](https://www.borgbackup.org/) that handles scheduled, deduplicated, encrypted backups via a declarative configuration.

## Services

| Service | Description | Image |
|---------|-------------|-------|
| **Borgmatic** | Scheduled Borg backup runner with cron-based execution and declarative config | `ghcr.io/borgmatic-collective/borgmatic` |

## Network Architecture

| Network | Type | Purpose |
|---------|------|---------|
| _none_ | — | Backups run locally; no networking required |

## Ports

None — Borgmatic does not expose any ports.

## Environment Variables

Create a `.env` file in the same directory as the compose file.

### Global

| Variable | Description | Example |
|----------|-------------|---------|
| `TIMEZONE` | Timezone (used for cron scheduling) | `America/New_York` |

### Borgmatic

| Variable | Description | Example |
|----------|-------------|---------|
| `BORG_PASSPHRASE` | Passphrase used to encrypt the Borg repository | `<strong-secret>` |

> **Note:** `BACKUP_CRON` (default `0 3 * * *` — daily at 03:00) and `RUN_ON_STARTUP` (default `false`) are hard-coded in the compose file. Adjust them there if a different schedule is needed.

### Volume Paths

| Variable | Description | Container Path | Mode |
|----------|-------------|----------------|------|
| `BACKUP_SOURCE` | Source data to back up | `/mnt/source` | `ro` |
| `BACKUP_REPOSITORY` | Borg repository destination (where archives live) | `/mnt/borg-repository` | `rw` |
| `BORGMATIC_D` | Borgmatic configuration directory (`config.yaml` lives here) | `/etc/borgmatic.d` | `rw` |
| `BORGMATIC_CONFIG` | Borg client config (known repos, security info) | `/root/.config/borg` | `rw` |
| `BORGMATIC_CACHE` | Borg chunk cache | `/root/.cache/borg` | `rw` |
| `BORGMATIC_STATE` | Borgmatic state directory (last-run database, etc.) | `/root/.local/state/borgmatic` | `rw` |
| `BORGMATIC_SSH` | SSH keys/config (for remote repositories over SSH, if used) | `/root/.ssh` | `rw` |

## Dependencies

None — this is a single-service stack with no external service dependencies. Note that `BACKUP_SOURCE` should point at data produced or managed by other stacks/services on the host.

## Deployment

1. Create your `.env` file with all required variables.

2. Place a `config.yaml` (and any included files) in the directory referenced by `BORGMATIC_D`. See the [Borgmatic configuration reference](https://torsion.org/borgmatic/docs/reference/configuration/) for available options.

3. Initialize the Borg repository before the first run (one-time):
   ```bash
   docker compose run --rm borgmatic borg init --encryption=repokey /mnt/borg-repository
   ```

4. Deploy:
   ```bash
   docker compose up -d
   ```

5. (Optional) Trigger a manual backup to verify the configuration:
   ```bash
   docker compose exec borgmatic borgmatic --verbosity 1
   ```

## Notes

- Container update monitoring is enabled via [What's Up Docker](https://github.com/fmartinou/whats-up-docker) labels (`wud.watch=true`, `wud.trigger.docker.enable=true`).
- The source volume is mounted read-only to guarantee the backup process cannot modify the data it's protecting.
- `BORG_PASSPHRASE` is required to read or restore from the repository — store it somewhere safe and separate from the backup destination, otherwise the backups are unrecoverable.
- Borg deduplicates and compresses at the chunk level, so subsequent backups of large, mostly-static datasets are fast and space-efficient.
- For restore operations, run `docker compose exec borgmatic borg list /mnt/borg-repository` to enumerate archives, then `borg extract` to recover files.
- Refer to the [docker-borgmatic README](https://github.com/borgmatic-collective/docker-borgmatic) for image-specific environment variables and behavior (e.g., `BACKUP_CRON`, `RUN_ON_STARTUP`, healthcheck options).
