# open-webui

Web front-end for [Ollama](https://ollama.com/) running on a separate LLM workstation (or any reachable Ollama host). Provides chat, RAG, automations, and a user/session store for everyone connecting to local models.

Upstream: [Open WebUI docs](https://docs.openwebui.com/) — [github.com/open-webui/open-webui](https://github.com/open-webui/open-webui)

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `open-webui` | `ghcr.io/open-webui/open-webui:latest` | Chat UI, user/session store, RAG vector store, scheduler for automations, embedding model cache |

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Shared reverse-proxy network — NPM resolves `open-webui` by container name here |

> Outbound traffic to Ollama on the LLM workstation routes through the host's default route. If your Ollama host is on a tailnet, setting `OLLAMA_BASE_URL` to its Tailscale IP means no host networking or extra network membership is needed.
>
> The compose declares `extra_hosts: - host.docker.internal:host-gateway`, which makes `host.docker.internal` resolve to the host's gateway IP from inside the container. Not required for the default IP-based `OLLAMA_BASE_URL`, but kept available as a fallback — e.g. switching `OLLAMA_BASE_URL` to `http://host.docker.internal:11434` if Ollama is bound directly to a host interface, or for reaching other host-bound services without leaving the bridge network.

---

## Ports

| Host port (`.env` var) | Container port | Bind | Purpose |
|------------------------|----------------|------|---------|
| `${OPENWEBUI_PORT}` (`8080`) | `8080` | all interfaces | Web UI + API |

Published on all interfaces so it's reachable on your LAN out of the box. To keep it behind a reverse proxy / loopback only, prefix the mapping in `compose.yaml` with `127.0.0.1:`.

> `8080` is a common host port (also the default for `filebrowser` and `calibre` in this repo). Change `OPENWEBUI_PORT` if it clashes on your host.

---

## Volumes

| Host path (`.env` var) | Container path | Purpose |
|------------------------|----------------|---------|
| `${OPENWEBUI_DATA_PATH}` | `/app/backend/data` | SQLite DB (users, chats, settings, automations, RAG knowledge), uploaded files, vector store, embedding/reranker model cache, and the auto-generated signing key if `WEBUI_SECRET_KEY` is unset |

---

## Environment Variables

### `.env` (Compose interpolation)

| Variable | Example | Purpose |
|----------|---------|---------|
| `TIMEZONE` | `America/New_York` | Container TZ — log timestamps and UI-rendered dates |
| `OPENWEBUI_PORT` | `8080` | Host port published for the web UI |
| `OLLAMA_BASE_URL` | `http://100.x.y.z:11434` | Ollama API endpoint on your LLM host. Use its Tailscale IP (if on a tailnet) to sidestep MagicDNS resolution from inside the container |
| `CORS_ALLOW_ORIGIN` | `*` (or `https://openwebui.example.com`) | Restrict CORS to the NPM-fronted hostname. Defaults to `*` and logs a loud boot warning |
| `ENABLE_SIGNUP` | `true` for first boot, then `false` | Account registration toggle. Boot once with `true` to create the admin, then flip and recreate the container |
| `USER_AGENT` | `open-webui/homelab` | User-Agent for outbound LangChain web fetches. Cosmetic — silences a boot warning |
| `WEBUI_SECRET_KEY` | _(secret — `openssl rand -base64 32`)_ | Session signing key. Pinning here makes sessions survive a data-volume wipe |
| `HF_TOKEN` | _(secret, optional)_ | HuggingFace Hub read-only token. Only needed if swapping embedding/reranker models often enough to hit anonymous rate limits |
| `OPENWEBUI_DATA_PATH` | `/path/to/open-webui/data` | Host bind for persistent data |

### Hardcoded in `compose.yaml`

_None._ All variables are sourced from `.env`.

---

## Healthchecks

| Service | Test | Interval | Timeout | Retries | Start period |
|---------|------|----------|---------|---------|--------------|
| `open-webui` | `curl -fsS http://localhost:8080/health` | 30s | 5s | 3 | 90s |

> `/health` returns `{"status":true}` once alembic migrations finish and uvicorn is bound. The generous start period covers the one-shot embedding model download (~90MB) and the full migration chain on first boot.

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme | WebSockets |
|----------|---------------|---------------|--------|------------|
| `openwebui.example.com` | `open-webui` | `8080` | `http` | **Required** |

> **WebSockets are not optional.** Open WebUI streams responses over WS. Without "Websockets Support" toggled on in the NPM proxy host, chats hang mid-reply or fall back to broken long-poll. If chats appear to take forever and only show up after a long delay, this is the first thing to check.

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing)
- Ollama listening at `OLLAMA_BASE_URL` and reachable from the host. Verify from the host:
  ```bash
  curl http://<ollama-host>:11434/api/tags
  ```

### Bring up

```bash
cd /opt/stacks/open-webui
cp .env.example .env
nano .env
# Generate WEBUI_SECRET_KEY: openssl rand -base64 32

sudo mkdir -p /path/to/open-webui/data

docker compose up -d
docker logs -f open-webui
```

Wait for `INFO: Started server process` and the scheduler worker line. Alembic runs the full migration chain on first boot; on subsequent boots it's a no-op.

### First-run setup

1. Visit `https://openwebui.example.com` after the NPM proxy host is configured (see above), or `http://<host-ip>:8080` directly.
2. Register the first account through the UI. This account becomes the admin.
3. Edit `.env` and set `ENABLE_SIGNUP=false`.
4. Recreate the container to pick up the change:
   ```bash
   docker compose up -d
   ```
   An `.env` edit alone does not take effect — Open WebUI reads `ENABLE_SIGNUP` only at startup.
5. In the admin UI under **Settings → Connections**, confirm the Ollama endpoint is healthy and the model list populates. If empty, check `OLLAMA_BASE_URL` and that Ollama on the workstation is bound to `0.0.0.0:11434`.

---

## Backup

Paths to include in Kopia / Borgmatic snapshots:

- `${OPENWEBUI_DATA_PATH}` — **the entire stack's state.** SQLite DB (users, chats, settings, automations, RAG knowledge), uploaded files, vector store, signing keys. Losing this loses everything except the model weights themselves (those live on the LLM host).

> The subdirectory `cache/embedding/` holds HuggingFace models that are regeneratable on demand. Excluding it shrinks backup size noticeably — worth doing if backup storage is tight, not worth bothering with otherwise.

---

## Upgrades

Standard WUD-driven flow. WUD notifies on a new `:latest` tag; review the upstream changelog, then apply manually:

```bash
cd /opt/stacks/open-webui
docker compose pull
docker compose up -d
docker logs -f open-webui
```

Open WebUI ships frequent releases (often weekly) with schema migrations. Watch the log on first boot post-upgrade — alembic runs any new migrations, and the embedding cache may re-pull if the default model has changed upstream.

> **Major-version jumps** (e.g., `0.x` → `1.0`): always read release notes. Env-var names and DB schema have shifted between minor versions historically.

---

## Notes & Gotchas

- **The image was not built for capability hardening.** Open WebUI's Dockerfile defaults `UID=0`/`GID=0` and explicitly notes that non-root configurations are untested. Adding `cap_drop: [ALL]` causes SQLite open failures during alembic migrations because the entrypoint expects the default Docker capability set. `security_opt: no-new-privileges:true` is fine and stays on (it's set in compose); capability dropping is not.
- **`ENABLE_SIGNUP` is read at startup only.** Toggling it in `.env` does nothing until the container is recreated. Workflow: boot once with `true`, register admin, flip to `false`, `docker compose up -d` to apply.
- **WebSockets in NPM are mandatory.** Streaming responses fail silently without them. See the Reverse Proxy section above.
- **`WEBUI_SECRET_KEY` defense in depth.** If unset, Open WebUI auto-generates and stores it inside the data volume. Pinning it in `.env` means sessions survive a data wipe and the secret isn't sitting inside the backed-up data directory.
- **Reaching Ollama over Tailscale.** The container's outbound traffic routes through the host, which knows the Tailscale route. Using the Tailscale IP in `OLLAMA_BASE_URL` (rather than a `*.ts.net` MagicDNS name) sidesteps any dependency on the container resolver being able to follow the DNS chain out to Tailscale.
- **`host.docker.internal` available as fallback.** The compose declares `extra_hosts: - host.docker.internal:host-gateway`, so `host.docker.internal` resolves to the host's bridge gateway from inside the container. Not used in the default configuration but available if `OLLAMA_BASE_URL` ever needs to point at a host-interface-bound service.
- **Embedding model first-boot download.** Open WebUI pulls `sentence-transformers/all-MiniLM-L6-v2` (~90MB) from HuggingFace on first boot for RAG embedding. Cached at `/app/backend/data/cache/embedding/` and never re-fetched unless the model is changed in the admin UI. `HF_TOKEN` only matters if you swap models often enough to hit anonymous rate limits.
- **`CORS_ALLOW_ORIGIN=*` warning.** Open WebUI defaults to `*` and logs a loud warning at boot. Behind a reverse proxy on a private network the practical risk is low, but tightening to the actual hostname silences the warning and is good hygiene.
- **Benign weight-loading note.** First boot logs `embeddings.position_ids | UNEXPECTED` from the BERT loader. This is an artifact of loading older checkpoints into newer transformers and is safe to ignore — does not affect embedding quality.

---

## References

- [Open WebUI docs](https://docs.openwebui.com/)
- [Open WebUI environment variables](https://docs.openwebui.com/getting-started/env-configuration/)
- [Open WebUI Dockerfile (UID/GID defaults)](https://github.com/open-webui/open-webui/blob/main/Dockerfile)
- [Ollama API reference](https://github.com/ollama/ollama/blob/main/docs/api.md)
- [Top-level README](../../README.md)
