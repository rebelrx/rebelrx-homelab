# bentopdf

Self-hosted PDF tools (merge, split, compress, edit) with [BentoPDF](https://github.com/alam00000/bentopdf).

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `bentopdf` | `ghcr.io/alam00000/bentopdf:latest` | Static web app serving client-side PDF tools |

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse-proxy access via NPM |

---

## Ports

| Host port (`.env` var) | Container port | Bind | Purpose |
|------------------------|----------------|------|---------|
| `${BENTOPDF_PORT}` (`3000`) | `8080` | all interfaces | Static web app |

Published on all interfaces so the app is reachable on your LAN out of the box. To keep it behind a reverse proxy / loopback only, prefix the mapping in `compose.yaml` with `127.0.0.1:`.

> `3000` is a common port (used by other apps like documenso, Grafana) — change `BENTOPDF_PORT` if it clashes with another service on the host.

---

## Volumes

None. BentoPDF is stateless — all PDF processing happens client-side in the browser, and the container only serves static assets. Nothing to bind mount.

---

## Environment Variables

### `.env` (Compose interpolation)

| Variable | Example | Purpose |
|----------|---------|---------|
| `TIMEZONE` | `America/New_York` | Container TZ |
| `BENTOPDF_PORT` | `3000` | Host port published for the web UI (container listens on `8080`) |

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme |
|----------|---------------|---------------|--------|
| `bentopdf.example.com` | `bentopdf` | `8080` | `http` |

---

## Deployment

```bash
cd /opt/stacks/bentopdf
cp .env.example .env
nano .env

docker compose up -d
```

No first-run setup required — open the app (e.g. `https://bentopdf.example.com` via NPM, or `http://<host-ip>:3000` directly) and use it.

---

## Backup

Nothing to back up — BentoPDF stores no state on the host or in the container.

---

## References

- [BentoPDF on GitHub](https://github.com/alam00000/bentopdf)
- [Top-level README](../../README.md)
