# npm

Reverse proxy and TLS termination with [Nginx Proxy Manager](https://nginxproxymanager.com/) — the front door for every other stack in the homelab.

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `nginx-proxy-manager` | `docker.io/jc21/nginx-proxy-manager:latest` | Web-based reverse proxy, TLS termination, certificate management |

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | The shared network most stacks join — NPM resolves their container names here |
| `nextcloud-aio` | External | Joined explicitly so NPM can resolve `nextcloud-aio-apache` and `nextcloud-aio-mastercontainer` |
| `nomad` (alias for `project-nomad_default`) | External | Joined so NPM can resolve NOMAD-managed app containers (`nomad_kiwix_server`, `nomad_kolibri`, `nomad_cyberchef`, `nomad_flatnotes`, etc.) spawned dynamically by the NOMAD Command Center |

> **NPM has to join every network it needs to resolve container names on.** Most stacks attach to `proxy_net`, so that single membership covers them. The exceptions are `nextcloud-aio` (Nextcloud uses its own external network) and `project-nomad_default` (NOMAD spawns managed app containers on its own network and won't attach them to `proxy_net`). If a future stack uses a different non-`proxy_net` network and needs reverse-proxy access, add its network to this compose's `networks:` block.

---

## Ports

This is the only stack that publishes ports to the host's public interfaces. Three port bindings, three different exposure levels:

| Host binding | Container | Purpose |
|--------------|-----------|---------|
| `0.0.0.0:80` | `80` | HTTP — redirects to HTTPS; Let's Encrypt HTTP-01 challenges land here |
| `0.0.0.0:443` | `443` | HTTPS — the user-facing entry point for every proxied host |
| `127.0.0.1:${NPM_ADMIN_PORT}:81` | `81` | Admin UI — loopback-bound; reached via the recursive NPM host entry below |

> `80` and `443` are the public entrypoints and must be on all interfaces. The admin UI (`81`) is **deliberately loopback-bound** — unlike the all-interfaces default used by most stacks, it's kept off the LAN because it controls the whole reverse proxy. Whatever network-level gating you use (host firewall, Tailscale ACLs) applies to the `0.0.0.0` bindings; the admin UI's loopback binding adds a second layer of isolation on top. To administer NPM remotely without the self-proxy below, use an SSH tunnel (see [Bootstrap loop](#notes--gotchas)) rather than exposing `81`.

---

## Volumes

| Host path (`.env` var) | Container path | Purpose |
|------------------------|----------------|---------|
| `${NPM_DATA}` | `/data` | NPM's SQLite DB (proxy hosts, redirection rules, users), Nginx config snippets, custom locations |
| `${NPM_LETSENCRYPT}` | `/etc/letsencrypt` | Let's Encrypt account keys, issued certificates, renewal history |

---

## Environment Variables

### `.env` (Compose interpolation)

| Variable | Example | Purpose |
|----------|---------|---------|
| `TIMEZONE` | `America/New_York` | Container TZ (affects cert renewal log timestamps) |
| `NPM_DATA` | `/path/to/npm/data` | Host bind for NPM state |
| `NPM_LETSENCRYPT` | `/path/to/npm/letsencrypt` | Host bind for Let's Encrypt data |
| `NPM_ADMIN_PORT` | `81` | Host port for the admin UI (loopback-bound) |

### Hardcoded in `compose.yaml`

| Variable | Value | Purpose |
|----------|-------|---------|
| `DISABLE_IPV6` | `true` | Skip IPv6 listeners (IPv4-only setup). Set to `false` if you enable IPv6 on the host |

---

## Reverse Proxy (NPM)

NPM proxies its own admin UI back to itself — a recursive setup that lets you reach the admin UI through the same TLS pipeline as every other stack:

| Hostname | Upstream host | Upstream port | Scheme |
|----------|---------------|---------------|--------|
| `npm.example.com` | `nginx-proxy-manager` | `81` | `http` |

> **Container-name upstream** — same pattern as every other stack. NPM resolves its own container name via Docker DNS on `proxy_net` and reaches its admin listener that way.
>
> **Bootstrap consideration:** because the admin host depends on NPM itself, if NPM is down you can't reach the admin UI to fix it. Fall back to an SSH tunnel: `ssh -L 8181:127.0.0.1:81 <host>`, then visit `http://127.0.0.1:8181` locally.

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing)
- `nextcloud-aio` external network (`docker network create nextcloud-aio` if missing)
- `project-nomad_default` external network — **created automatically when the `project-nomad` stack is brought up.** If NPM is brought up first, either bring `project-nomad` up first, pre-create the network with `docker network create project-nomad_default`, or temporarily comment out the `nomad` network block in this compose
- Ports `80` and `443` free on all host interfaces (no other web server running)

### Bring up

```bash
cd /opt/stacks/npm
cp .env.example .env
nano .env

# Pre-create local dirs (match your ${NPM_DATA} / ${NPM_LETSENCRYPT})
sudo mkdir -p /path/to/npm/{data,letsencrypt}

docker compose up -d
```

### First-run setup

1. Open the admin UI. **Three options** in increasing order of convenience:
   - **SSH tunnel** (always works, even before NPM proxies itself): `ssh -L 8181:127.0.0.1:81 <host>`, then `http://127.0.0.1:8181`
   - **Direct from host** (if you have a local session): `http://127.0.0.1:81`
   - **Through NPM** (after step 4 below): `https://npm.example.com`
2. Default credentials: `admin@example.com` / `changeme` — change immediately on first login.
3. Configure any reverse-proxy hosts you need. Start with the homelab essentials, then add stacks as they're brought up.
4. **Add the self-reverse-proxy entry for the admin UI** so future logins go through TLS:
   - **Proxy Hosts → Add Proxy Host**
   - Domain Names: `npm.example.com`
   - Forward Hostname: `nginx-proxy-manager`
   - Forward Port: `81`
   - Scheme: `http`
   - Enable WebSockets Support
   - SSL tab: request a Let's Encrypt cert (or use a wildcard cert if you have one)
5. Verify by opening `https://npm.example.com`.

---

## Backup

Paths to include in Kopia / Borgmatic snapshots:

- `${NPM_DATA}` — SQLite DB with every proxy host, redirection rule, and user account. **Losing this means manually rebuilding every NPM entry from scratch.**
- `${NPM_LETSENCRYPT}` — issued certificates and Let's Encrypt account keys. Losing certs forces re-issuance; losing account keys forces a fresh account.

> Both directories are small (typically <100 MB combined). High-frequency snapshots are cheap here and the recovery value is enormous — NPM rebuild without backups is hours of manual work.

---

## Notes & Gotchas

- **NPM is the homelab's keystone.** Every proxied URL depends on this container being healthy. When it goes down, nothing's reachable by hostname — only by direct port (if the stack publishes one). Treat reboots and upgrades with more care than usual.
- **Network membership rules.** NPM has to join the network of any container it proxies via container DNS. Today that's `proxy_net` (used by most stacks), `nextcloud-aio` (used by Nextcloud's spawned containers), and `project-nomad_default` (used by NOMAD-managed app containers). If you add a stack on a fourth network and want reverse-proxy access without exposing host ports, add that network to NPM's `networks:` block, then `docker compose up -d` to rejoin.
- **NOMAD-managed containers are dynamic.** Unlike most stacks where container names are fixed in compose, NOMAD spawns its managed app containers (Kiwix, Kolibri, CyberChef, FlatNotes, etc.) at runtime via Docker socket on the `project-nomad_default` network. NPM resolves them by name (`nomad_kiwix_server:8080`, `nomad_kolibri:8080`, etc.) the same way it resolves any other container — once they're created from the NOMAD Command Center, they're reachable via NPM proxy hosts. See the [project-nomad stack README](../project-nomad/README.md) for the per-app NPM configuration.
- **Bootstrap loop.** The admin UI is proxied through NPM itself (`npm.example.com` → `nginx-proxy-manager:81`). When NPM is healthy this is great — uniform TLS, accessed like any other stack. When NPM is broken, the admin UI is broken too. The escape hatch is the loopback host binding (`127.0.0.1:81`) plus an SSH tunnel.
- **WebSockets matter on the admin host.** NPM's admin UI uses WebSockets for live updates. Make sure "Websockets Support" is enabled on the `npm.example.com` proxy host.
- **Let's Encrypt and private/tailnet hostnames.** Hostnames that don't resolve publicly (e.g. a private LAN domain or a tailnet name like `*.<your-tailnet>.ts.net`) won't validate via Let's Encrypt's HTTP-01 challenge, since there's no public DNS pointing at the host. For valid certs on such hostnames, use DNS-01 against a real DNS zone you control, or fall back to [Tailscale-issued certs](https://tailscale.com/kb/1153/enabling-https) uploaded to NPM. Self-signed certs work inside a private network, but browsers will warn.
- **Port 80 still needed.** Some clients try HTTP first and follow the redirect, and Let's Encrypt HTTP-01 challenges need it. Keep `80:80` bound even when it's not the primary entry point.
- **`security_opt: no-new-privileges` is on.** NPM doesn't need any privileged operations beyond binding low ports (Docker handles that via the daemon), so the flag is safe and reduces blast radius.

---

## References

- [Nginx Proxy Manager — setup docs](https://nginxproxymanager.com/setup/)
- [Nginx Proxy Manager — proxy hosts](https://nginxproxymanager.com/guide/#proxy-hosts)
- [Tailscale HTTPS certs](https://tailscale.com/kb/1153/enabling-https)
- [Top-level README](../../README.md)
