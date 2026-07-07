# guest

Tailscale sidecar containers that expose specific homelab services to guests via dedicated tailnet identities. One sidecar per guest-exposed app — currently RomM only, but the stack is designed to grow as more services are shared out.

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `tailscale-romm` | `tailscale/tailscale:stable` | Tailscale sidecar exposing RomM as `romm-guest` on the tailnet |

> Future sidecars (e.g. `tailscale-jellyfin`, `tailscale-immich-shared`) follow the same pattern: each gets its own Tailscale identity, hostname, state dir, and `serve.json`.

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `guest_net` | External | Shared with the `romm` stack — sidecar reaches `http://romm:<port>` here |

> Each Tailscale sidecar joins `guest_net` (and only `guest_net`). No `proxy_net` membership — these sidecars don't go through NPM. They expose services directly to the tailnet using Tailscale's built-in `serve` feature.

---

## Ports

**No host ports.** The Tailscale sidecar runs its own listener inside the container and registers it as a tailnet endpoint — there's nothing to publish on the host. Guests reach the service at `romm-guest.<tailnet>.ts.net`, where `<tailnet>` is your tailnet's name (find it in the Tailscale admin console — it looks like `tailXXXXXX.ts.net`, or a custom domain if you've set one). MagicDNS handles routing.

---

## Volumes

| Host path (`.env` var) | Container path | Purpose |
|------------------------|----------------|---------|
| `${TAILSCALE_ROMM_STATE}` | `/var/lib/tailscale` | Tailscale node identity, machine key, preferences — persistent across restarts |
| `${SERVE_CONFIG}` | `/config/serve.json` | Declarative `tailscale serve` config — defines what HTTP service this node proxies |

> The state dir is **identity-bearing**. Wiping it requires a fresh authkey to re-register the node on the tailnet (and the node will appear as a new device under a different machine key).

---

## Environment Variables

### `.env` (Compose interpolation)

| Variable | Example | Purpose |
|----------|---------|---------|
| `TS_AUTHKEY` | *(secret)* | Tailscale auth key — used **once** on first start to register the node. Subsequent restarts use the saved identity in `${TAILSCALE_ROMM_STATE}` |
| `TAILSCALE_ROMM_STATE` | `/path/to/tailscale/romm` | Host bind for Tailscale state |
| `SERVE_CONFIG` | `/path/to/tailscale/config/serve-romm.json` | Host path to the `serve.json` for this sidecar |

> `TS_STATE_DIR=/var/lib/tailscale` and `TS_SERVE_CONFIG=/config/serve.json` are set directly in `compose.yaml` (these are container-internal paths, not host paths).

> **The authkey is single-use after the node is registered.** You can leave `TS_AUTHKEY` in `.env` — the container only uses it when the state dir doesn't yet contain an identity. For tighter ops, generate an ephemeral or single-use authkey in the Tailscale admin console and remove the value from `.env` after first start.

---

## Reverse Proxy

Not applicable. These sidecars are reached directly via Tailscale's tailnet, not through NPM.

| Service | Public URL |
|---------|------------|
| `tailscale-romm` | `https://romm-guest.<tailnet>.ts.net` |

Tailscale terminates TLS using its own auto-issued certificate (HTTPS provided by Tailscale's serve feature when `HTTPSProxy: true` is set in `serve.json`).

---

## `serve.json` format

The `serve.json` file declares what each Tailscale node proxies. For `romm-guest`, a minimal config looks like this (replace `<tailnet>` with your actual tailnet name and `<port>` with RomM's internal listen port, e.g. `8080`):

```json
{
  "TCP": {
    "443": { "HTTPSProxy": true }
  },
  "Web": {
    "romm-guest.<tailnet>.ts.net:443": {
      "Handlers": {
        "/": { "Proxy": "http://romm:<port>" }
      }
    }
  }
}
```

The `http://romm:...` URL resolves over `guest_net` to the RomM container.

For the full schema, see Tailscale's [`serve` config reference](https://tailscale.com/kb/1242/tailscale-serve).

---

## Adding a new guest sidecar

To expose another app (e.g. Jellyfin) to a separate tailnet identity:

1. **Attach the target app to `guest_net`** in its own stack's `compose.yaml`:

   ```yaml
   networks:
     - <existing networks>
     - guest_net
   ```

   And declare it externally at the bottom:

   ```yaml
   networks:
     guest_net:
       external: true
   ```

2. **Add a sidecar service** to this stack's `compose.yaml`:

   ```yaml
   tailscale-jellyfin:
     image: tailscale/tailscale:stable
     container_name: tailscale-jellyfin
     hostname: jellyfin-guest
     restart: unless-stopped
     cap_add: [net_admin, net_raw]
     environment:
       - TS_AUTHKEY=${TS_AUTHKEY_JELLYFIN}
       - TS_STATE_DIR=/var/lib/tailscale
       - TS_SERVE_CONFIG=/config/serve.json
     volumes:
       - ${TAILSCALE_JELLYFIN_STATE}:/var/lib/tailscale
       - ${SERVE_CONFIG_JELLYFIN}:/config/serve.json
     networks:
       - guest_net
     security_opt:
       - no-new-privileges:true
     labels:
       - wud.watch=true
       - wud.trigger.docker.enable=false
   ```

3. **Create the matching `serve.json`** at `${SERVE_CONFIG_JELLYFIN}` with a `Proxy` URL pointing at `http://jellyfin:8096` (or whatever port).

4. **Add corresponding `.env` vars** (`TS_AUTHKEY_JELLYFIN`, `TAILSCALE_JELLYFIN_STATE`, `SERVE_CONFIG_JELLYFIN`).

5. Grant guest access via Tailscale ACLs or device sharing — see [Tailscale docs on sharing nodes](https://tailscale.com/kb/1084/sharing).

---

## Deployment

### Prerequisites

- `guest_net` external network (`docker network create guest_net` if missing)
- Target app (RomM) already running and attached to `guest_net`
- A Tailscale auth key — generate from the [Tailscale admin console](https://login.tailscale.com/admin/settings/keys). Use a **reusable** key only if you plan to rebuild the state dir; otherwise a single-use ephemeral key is fine.
- A valid `serve.json` at `${SERVE_CONFIG}` before first start

### Bring up

```bash
cd /opt/stacks/guest
cp .env.example .env

# Generate / paste authkey
nano .env  # set TS_AUTHKEY

# Pre-create state dir and serve.json (match your ${TAILSCALE_ROMM_STATE} / ${SERVE_CONFIG})
mkdir -p /path/to/tailscale/romm
mkdir -p /path/to/tailscale/config
nano /path/to/tailscale/config/serve-romm.json  # see schema above

chmod 600 .env

docker compose up -d
```

### First-run setup

1. Watch logs to confirm registration:

   ```bash
   docker compose logs -f tailscale-romm
   ```

   Expect to see a "Success." line followed by the node's tailnet name.

2. Confirm the node appears in the [Tailscale admin console](https://login.tailscale.com/admin/machines) as `romm-guest`.

3. From any tailnet device, hit `https://romm-guest.<tailnet>.ts.net` — RomM should load.

4. **Share the node with guests** via the admin console (Machine → … → Share). Recipients accept the share via a link and can then resolve `romm-guest.<tailnet>.ts.net` from their own tailnets.

5. Optional: clear `TS_AUTHKEY` from `.env` once registration is complete (the state dir now holds the identity; the key is no longer needed).

---

## Backup

Paths to include in Kopia / Borgmatic snapshots:

- `${TAILSCALE_ROMM_STATE}` — node identity. Restoring this on a fresh host avoids re-registration and keeps the same machine key (so existing shares with guests continue to work)
- `${SERVE_CONFIG}` — the `serve.json` itself (small; easy to lose)

`.env` is also worth backing up so a restored host can fall back to the authkey path if state restore fails.

---

## Notes & Gotchas

- **Pattern, not a one-off.** This stack is the homelab's pattern for exposing services to tailnet guests via per-app Tailscale identities. Each sidecar gets its own state, its own `serve.json`, and its own tailnet hostname — so guests granted access to RomM can't laterally browse Jellyfin or anything else, even if both have sidecars.
- **One identity per sidecar.** Don't try to multiplex apps onto a single Tailscale node — `serve.json` could technically host multiple handlers under the same hostname, but doing so breaks the access-isolation model that makes this stack useful.
- **Authkey lifecycle.** Tailscale authkeys can be single-use, reusable, or ephemeral (auto-expiring devices). For long-lived guest sidecars, generate a non-ephemeral single-use key, register the node, then revoke the key. The state dir keeps the identity working after revocation.
- **Cap requirements.** `net_admin` and `net_raw` are required by the Tailscale userspace networking layer. Removing them breaks the tunnel. `no-new-privileges` is still active.

---

## References

- [Tailscale Docker docs](https://tailscale.com/docs/features/containers/docker)
- [`tailscale serve` reference](https://tailscale.com/kb/1242/tailscale-serve)
- [Sharing tailnet nodes with users](https://tailscale.com/kb/1084/sharing)
- [Top-level README](../../README.md)
