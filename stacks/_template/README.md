# template

Boilerplate Docker Compose stack for quickly spinning up new services with consistent conventions.

## Usage

1. Copy this directory for your new stack.
   ```bash
   cp -r _template <new-stack-name>
   cd <new-stack-name>
   ```

2. Update `docker-compose.yml`:
   - Replace the service name, image, container name, and ports
   - Update the comment header with the app name, description, and docs URL
   - Add any required environment variables, volumes, and networks
   - Adjust or remove `proxy_net` depending on whether the service needs reverse proxy access

3. Create a `.env` file with your variables.

4. Deploy:
   ```bash
   docker compose up -d
   ```

## Conventions

This template follows the conventions used across all stacks:

- **Comment header**: `# App Name (Description)` + docs URL above each service key
- **Key ordering**: `image → container_name → restart → network_mode → depends_on → ports → environment → volumes → networks → deploy → shm_size → healthcheck → labels`
- **Environment syntax**: List/array form (`- KEY=value`)
- **Port mappings**: Quoted (`"127.0.0.1:${PORT}:8080"`)
- **WUD labels**: Always include `wud.watch=true` and `wud.trigger.docker.enable=false`
- **No inline comments** or commented-out code
- **No trailing slashes** on volume paths
