# portainer

Web UI for managing Docker containers, images, volumes, and networks with [Portainer Business Edition](https://www.portainer.io/) (free for personal use up to 3 nodes).

> **Coexists with Dockge in this homelab.** Dockge handles compose-stack management with a git-friendly UI; Portainer handles fine-grained container/image/volume operations. Both manage the same Docker daemon and are safe to run side-by-side.

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `portainer` | `portainer/portainer-ee:sts` | Container management UI (Business Edition, Short-Term Support release line) |

> Uses **Portainer EE** (Business Edition), not CE. EE has more features but needs a free license (see [First-run setup](#first-run-setup)). To skip the license entirely, swap the image to `portainer/portainer-ce:latest` — see the [Notes](#notes--gotchas).

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse-proxy access via NPM |

---

## Ports

| Host port (`.env` var) | Container port | Bind | Purpose |
|------------------------|----------------|------|---------|
| `${PORTAINER_PORT}` (`9443`) | `9443` | all interfaces | Web UI (HTTPS, self-signed) |

Published on all interfaces so the UI is reachable on your LAN out of the box (`https://<host-ip>:9443` — expect a self-signed cert warning). Portainer also listens internally on `9000` (HTTP); to publish that instead, map `${PORTAINER_PORT}:9000`. To keep it behind a reverse proxy / loopback only, prefix the mapping in `compose.yaml` with `127.0.0.1:`.

---

## Volumes

| Host path (`.env` var) | Container path | Mode | Purpose |
|------------------------|----------------|------|---------|
| `${PORTAINER_DATA}` | `/data` | rw | Portainer settings, users, API keys, environment definitions, license |
| `/var/run/docker.sock` | `/var/run/docker.sock` | rw | Docker daemon socket — required to manage containers, images, volumes, networks |

---

## Environment Variables

### `.env` (Compose interpolation)

| Variable | Example | Purpose |
|----------|---------|---------|
| `PORTAINER_PORT` | `9443` | Host port published for the HTTPS web UI |
| `PORTAINER_DATA` | `/path/to/portainer/data` | Host bind for Portainer state |

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme |
|----------|---------------|---------------|--------|
| `portainer.example.com` | `portainer` | `9443` | **`https`** |

> **HTTPS upstream** — Portainer terminates its own self-signed TLS on `9443`. NPM has to use `https` as the scheme, with "Custom certificate verification disabled" (or equivalent) ticked in the proxy host's advanced settings to accept the self-signed cert. This is the same pattern as the AIO mastercontainer (`nextcloud-admin.example.com`).
>
> Alternative: use `http://portainer:9000` if you'd rather have NPM be the only TLS terminator. Functionally equivalent; saves the cert-verification toggle.

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing)

### Bring up

```bash
cd /opt/stacks/portainer
cp .env.example .env
nano .env

# Pre-create local dir (match your ${PORTAINER_DATA})
sudo mkdir -p /path/to/portainer/data

docker compose up -d
```

### First-run setup

1. Open the app (e.g. `https://portainer.example.com` via NPM, or `https://<host-ip>:9443` directly).
2. Create the **first admin user** within 5 minutes — Portainer locks the initial-setup endpoint after that window for security. If you miss it, restart the container and try again.
3. Choose **Get Started** with the local Docker environment (the host this container is on).
4. **Business Edition license:** Portainer EE requires a free license for up to 3 nodes. Sign up at [portainer.io/take-3](https://www.portainer.io/take-3) and paste the license key under Settings → License. Without a license, the UI runs in read-only mode after the trial expires.
5. Optional: configure additional environments (other Docker hosts, Kubernetes, Edge agents) under **Environments → Add environment**.

---

## Backup

Paths to include in Kopia / Borgmatic snapshots:

- `${PORTAINER_DATA}` — admin accounts, API keys, environment definitions, stored stack templates, license

> Portainer also has a built-in backup feature (Settings → **Backup Portainer**) that produces a portable archive. Useful before major upgrades alongside file-level snapshots.

---

## Notes & Gotchas

- **RW Docker socket.** Portainer manages container lifecycle (create, start, stop, destroy, exec) and image/volume/network operations, so the socket mount is intentionally read-write. This is one of the stacks in the homelab with RW socket access (alongside `dockge`). Anything that compromises the Portainer container has full Docker access on the host.
- **`no-new-privileges` enabled.** Portainer doesn't need privilege escalation, so the flag reduces blast radius even with the RW socket.
- **5-minute setup window.** Portainer's initial admin creation endpoint times out after 5 minutes of inactivity on first boot. If you boot the container and don't immediately create the admin user, restart with `docker compose restart portainer` to reset the timer.
- **Business Edition free license.** The image is `portainer-ee` (Business Edition), not `portainer-ce` (Community Edition). EE has more features but requires a free license for up to 3 nodes — without it the UI degrades to read-only after the trial. CE is unrestricted but missing some features (RBAC, edge management, etc.). Switch to `portainer/portainer-ce:latest` in compose if you'd rather skip the license dance.
- **Coexistence with Dockge.** Both Portainer and Dockge manage Docker on this host. Generally fine — they use the same socket and don't fight over containers — but avoid editing the same stack from both UIs in quick succession to prevent stale-state confusion. Dockge is better for compose-stack workflow (file-based, git-friendly); Portainer is better for one-off container ops and detailed inspection.

---

## References

- [Portainer docs — Docker install](https://docs.portainer.io/start/install-ce/server/docker/linux)
- [Portainer Business Edition free license](https://www.portainer.io/take-3)
- [Top-level README](../../README.md)
