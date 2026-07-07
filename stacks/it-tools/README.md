# it-tools

Self-hosted collection of handy client-side developer/IT utilities — cryptography, encoders/decoders, format converters, network and math tools — all running in-browser with nothing sent to a backend.

Deployed from the **[sharevb fork](https://github.com/sharevb/it-tools)**, not the original [CorentinTh/it-tools](https://github.com/CorentinTh/it-tools). The original is on pause (last release Oct 2024); the fork is actively maintained with ~190 additional tools. See [Notes & Gotchas](#notes--gotchas).

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `it-tools` | `ghcr.io/sharevb/it-tools:v2026.1.4` | Static site (unprivileged nginx) serving the it-tools SPA |

Single-container stack. No database, no sidecars, no persistent state.

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reverse-proxy access via NPM |

> `it-tools` joins `proxy_net` only. NPM reaches the container by name over this network.

---

## Ports

| Host port (`.env` var) | Container port | Bind | Purpose |
|------------------------|----------------|------|---------|
| `${IT_TOOLS_HTTP_PORT}` (`8080`) | `8080` | all interfaces | Static site (unprivileged nginx) |

Published on all interfaces so it's reachable on your LAN out of the box. To keep it behind a reverse proxy / loopback only, prefix the mapping in `compose.yaml` with `127.0.0.1:`.

> **Some tools need HTTPS (a secure context).** PGP, anything using the WebCrypto API, and PWA install require a secure context and **silently fail over plain `http://`**. Reaching the app directly at `http://<host-ip>:8080` works for most tools but breaks those — put it behind an `https://` reverse proxy (NPM) for full functionality.
>
> The sharevb image listens on **`8080`** (unprivileged nginx), unlike the original corentinth image which listens on `80`. The NPM upstream port must be `8080`.

---

## Volumes

**None required.** it-tools is stateless — it ships the entire app as static files baked into the image and holds no runtime data.

Two *optional* customization files can be mounted (see the commented volume block in `compose.yaml`):

| Host path (`.env` var) | Container path | Purpose |
|------------------------|----------------|---------|
| `${IT_TOOLS_CONFIG}/tools-filter.json` | `/usr/share/nginx/html/tools-filter.json` | Include/exclude tools or categories via regex |
| `${IT_TOOLS_CONFIG}/home.custom.md` | `/usr/share/nginx/html/home.custom.md` | Custom markdown block on the home page |

> `tools-filter.json` accepts `includeCategoryFilterRegex`, `excludeCategoryFilterRegex`, `includeToolsFilterRegex`, `excludeToolsFilterRegex`. Category matches on English category names; tools match on tool path/url.

---

## Environment Variables

### `.env` (Compose interpolation)

| Variable | Example | Purpose |
|----------|---------|---------|
| `IT_TOOLS_HTTP_PORT` | `8080` | Host port published for the web UI (container listens on `8080`) |
| `TIMEZONE` | `America/New_York` | Cosmetic (nginx log timestamps). **Not consumed by the stock minimal compose** — add `TZ=${TIMEZONE}` to a compose `environment:` block to apply it |
| `IT_TOOLS_CONFIG` | `/path/to/it-tools/config` | Host dir for the optional customization files above. Unused unless the volume block is enabled |

> The image tag is pinned directly in `compose.yaml` (`:v2026.1.4`) rather than `.env`, so WUD can watch for newer tags without pulling them. See the WUD note below.

---

## Reverse Proxy (NPM)

| Hostname | Upstream host | Upstream port | Scheme |
|----------|---------------|---------------|--------|
| `it-tools.example.com` | `it-tools` | `8080` | `http` |

> TLS terminates at NPM; the container serves plain HTTP internally. **Serve it over HTTPS** — some tools (PGP, and anything using the WebCrypto API or PWA install) require a secure context and silently fail over plain `http://`. Reaching the container directly by IP on `http://` will break those tools; go through the NPM `https://` hostname for full functionality.

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing)
- An NPM proxy host pointing your chosen hostname (e.g. `it-tools.example.com`) → `it-tools:8080` (http)

No secrets to generate, no data directories to pre-create.

### Bring up

```bash
cd /opt/stacks/it-tools
cp .env.example .env
chmod 600 .env

docker compose up -d
```

### Verify

1. `docker compose ps` shows `it-tools` running (no crash-loop — see the `cap_drop` note below).
2. Open the app (e.g. `https://it-tools.example.com` via NPM) — the tool grid loads.
3. Spot-check a WebCrypto tool (e.g. **PGP encryption**) to confirm the secure-context tools work through NPM's HTTPS.

---

## Backup

**Nothing runtime to back up.** The container is disposable and holds no state — a lost container is recreated from `compose.yaml` alone.

Include in Kopia / Borgmatic snapshots only the stack definition:

- `compose.yaml`, `.env`, `.env.example`
- `${IT_TOOLS_CONFIG}/` — only if you use the optional customization files

Everything else is reproducible from the pinned image.

---

## Notes & Gotchas

- **Running the fork on purpose.** The original [CorentinTh/it-tools](https://github.com/CorentinTh/it-tools) is paused — last real release was `v2024.10.22`, and the maintainer confirmed on record ([issue #1635](https://github.com/CorentinTh/it-tools/issues/1635)) that the project is on hold and pointed users to the sharevb fork. The fork ships versioned images (`v2026.1.4`), rebuilds on every push, and carries the larger toolset. The original image still *works* (it's static), it's just frozen.
- **Single-maintainer supply chain — pin the tag.** The fork is essentially one person. The image is pinned to a specific version (`v2026.1.4`); let WUD notify on new tags, and review before bumping. **Do not** track `:latest` or `:nightly` — that pulls unreviewed changes from a single-maintainer repo on every `docker compose pull`.
- **WUD is notify-only.** Consistent with the rest of the homelab (`wud.trigger.docker.enable=false`). WUD flags newer tags; upgrades are manual.
- **Listens on `8080`, not `80`.** The sharevb image uses unprivileged nginx on `8080`. This is why the NPM upstream port and the published port mapping are `8080`, unlike the original image's `80`.
- **`cap_drop: ALL` — verify on first start.** Expected to be fine here because the unprivileged-nginx base runs entirely as non-root and does no root→worker setuid drop, so it needs none of the caps that trip up gosu/entrypoint images. But confirm the container doesn't crash-loop on first boot; if it does, `cap_drop: ALL` is the first thing to relax (add back `CHOWN`, `SETUID`, `SETGID`, `DAC_OVERRIDE`, or remove the drop).
- **No healthcheck defined.** Recon available binaries first (`docker exec it-tools sh -c 'which wget curl nc'`) before adding one — the nginx base may ship none of them. Given it's a trivial static server, omitting the healthcheck is reasonable; NPM's own upstream check covers reachability.
- **Stateless — customization is via mounts, not the UI.** There are no persisted settings. Filtering the tool list or customizing the home page requires the mounted config files above; those changes live in `${IT_TOOLS_CONFIG}` on the host, not inside the container.

---

## References

- [sharevb/it-tools (fork — deployed)](https://github.com/sharevb/it-tools)
- [sharevb/it-tools container package](https://github.com/sharevb/it-tools/pkgs/container/it-tools)
- [CorentinTh/it-tools (upstream — paused)](https://github.com/CorentinTh/it-tools)
- [Top-level README](../../README.md)
