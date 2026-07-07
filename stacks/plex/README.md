# plex

Self-hosted media server with [Plex](https://www.plex.tv/) — proprietary alternative to Jellyfin. NVIDIA GPU-accelerated transcoding via the host's NVIDIA GPU.

> **Coexists with Jellyfin in this homelab.** Plex handles DLNA, smart-TV discovery, and the heavy-feature commercial workflows; Jellyfin handles the open-source / app-flexible workflows. Both serve the same NAS media library.

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `plex` | `lscr.io/linuxserver/plex:latest` | Media server, web UI, transcoding engine, DLNA discovery |

---

## Networks

**None.** Plex uses `network_mode: host`, so the container shares the host's network namespace directly. It does not join `proxy_net` or `media_net` — host networking overrides any network membership.

> Host networking is required for DLNA, SSDP discovery (smart TVs finding the server), GDM (Plex's own discovery protocol), and Plex's port-mapping behavior. Bridge networking breaks all of these.

---

## Ports

**Host networking — every port Plex listens on is bound to all host interfaces directly:**

| Port | Purpose |
|------|---------|
| `32400/tcp` | Plex web UI and API (the primary access port) |
| `1900/udp` | DLNA / SSDP discovery |
| `5353/udp` | Bonjour / Avahi discovery |
| `8324/tcp` | Plex Companion (Roku, etc.) |
| `32410-32414/udp` | GDM network discovery |
| `32469/tcp` | DLNA media server |

Because this runs in host-network mode, there is **no Docker `ports:` mapping** — the container binds the host sockets directly on all interfaces. You can't scope it via compose the way the bridge-networked stacks do; the boundary is the host firewall. This deployment gates reachability with UFW, allowing `32400/tcp` inbound only on the Tailscale interface:

```bash
sudo ufw allow in on tailscale0 to any port 32400 proto tcp comment 'Plex via Tailscale'
```

> Adjust the rule to your access model — e.g. `sudo ufw allow from 192.168.1.0/24 to any port 32400 proto tcp` for LAN access. The pattern is reusable for any host-networked or `0.0.0.0`-bound service you want to scope.

---

## Volumes

| Host path (`.env` var) | Container path | Purpose |
|------------------------|----------------|---------|
| `${PLEX_CONFIG}` | `/config` | Plex Media Server config, library DB, metadata cache, plugins |
| `${PLEX_TRANSCODE}` | `/transcode` | Transcode scratch space — **must be local NVMe**, never NFS |
| `${PLEX_MOVIES}` | `/movies` | Movies library |
| `${PLEX_TV}` | `/tv` | TV library |
| `${PLEX_MUSIC}` | `/music` | Music library |

> Each media library is mounted explicitly at its own container path. To add another library (audiobooks, comics, etc.), add a new explicit mount and a matching `.env` variable rather than reaching into a catch-all root — keeps Plex's library boundaries cleanly aligned with what you actually want indexed.

---

## Environment Variables

### `.env` (Compose interpolation)

| Variable | Example | Purpose |
|----------|---------|---------|
| `PUID` | `1000` | UID inside container (linuxserver convention) |
| `PGID` | `1000` | GID inside container |
| `TIMEZONE` | `America/New_York` | Container TZ |
| `PLEX_CLAIM` | *(one-time token)* | **Claim token for first-run binding** — fetched from [plex.tv/claim](https://www.plex.tv/claim), valid for ~4 minutes. After first launch the server is bound to your Plex account; this value becomes meaningless and can be cleared |
| `PLEX_CONFIG` | `/path/to/plex/library` | Host bind for config |
| `PLEX_TRANSCODE` | `/path/to/plex/transcode` | Host bind for transcode scratch (local NVMe) |
| `PLEX_MOVIES` | `/path/to/nas-media/movies` | Movies library |
| `PLEX_TV` | `/path/to/nas-media/tv` | TV library |
| `PLEX_MUSIC` | `/path/to/nas-media/music` | Music library |

### Hardcoded in `compose.yaml`

| Variable | Value | Purpose |
|----------|-------|---------|
| `VERSION` | `docker` | linuxserver image tag — `docker` = stable, `latest` = beta, `public` = standard Plex |
| `NVIDIA_VISIBLE_DEVICES` | `all` | Expose all GPUs to the container |
| `NVIDIA_DRIVER_CAPABILITIES` | `all` | Enable all driver capability classes (broader than necessary; minimum is `video,utility`) |

---

## GPU Acceleration

Plex uses NVENC/NVDEC for hardware transcoding. Two mechanisms are declared:

1. **`runtime: nvidia`** (legacy nvidia-docker v1 approach)
2. **`deploy.resources.reservations.devices`** (modern NVIDIA Container Toolkit approach)

With both present, the modern `deploy:` block is what actually grants GPU access. The legacy `runtime: nvidia` line is redundant but harmless.

**Prerequisites:**
- NVIDIA proprietary driver + DKMS on the host
- NVIDIA Container Toolkit installed; Docker daemon configured with the `nvidia` runtime
- A **Plex Pass** subscription (hardware transcoding is a paid feature)

**Enable in Plex:**

1. Settings → **Transcoder** → Use hardware acceleration when available
2. Use hardware-accelerated video encoding (Plex Pass only)

**Verify:**

```bash
# Force a transcode in a Plex client (e.g. play a 4K file on a 1080p target)
nvidia-smi
# Look for the Plex Transcoder process in the GPU process list
```

---

## Reverse Proxy

**Not applicable.** Plex is accessed directly at `<host>:32400` — there's no NPM entry for it. Plex's own apps, browser clients, and `https://app.plex.tv` discover the server via Plex's relay/account-linking system (set up on first run via `PLEX_CLAIM`).

If you ever want to expose Plex via NPM, you'd need to disable host networking (which breaks DLNA discovery) and add Plex to `proxy_net` with port `32400` exposed. That's an explicit tradeoff and not recommended.

---

## Tailscale Access & Plexamp

*(This section documents the tailnet-only access model this stack was built for. If you're exposing Plex on the LAN or via port-forwarding instead, it doesn't apply — Plex's normal discovery handles those.)*

Plex clients don't connect to your server by IP — they ask **plex.tv** for the list of "connections" your server has published, then try each one. By default that list contains:

- The server's LAN IP (`192.168.x.x:32400`) — useless to a client off-LAN
- The server's public WAN IP (`<wan>:32400`) — only works if you've port-forwarded, which this deployment doesn't

Without a Tailscale URL in that list, **Plexamp will report "No servers found"** when run off-LAN, even when the tailnet path works perfectly at the network layer. Plexamp specifically has no manual server-entry UI — it's 100% reliant on plex.tv discovery.

### Configuration (Settings → Network)

| Field | Value |
|-------|-------|
| **Custom server access URLs** | `http://<tailscale-ip>:32400` (e.g. `http://100.x.y.z:32400`) — published to plex.tv so off-LAN clients find the tailnet path |
| **LAN Networks** | `192.168.1.0/24,100.64.0.0/10` — adds the Tailscale CGNAT range so tailnet streams aren't throttled as "remote" (replace the first entry with your LAN range) |
| **Secure connections** | `Preferred` (not `Required`) — `Required` rejects `http://` URLs; Tailscale handles transport encryption so HTTP fallback is fine |
| **Remote Access** | Safe to leave disabled — Tailscale is the remote-access path |

After saving, plex.tv picks up the new URL within ~30 seconds. Plexamp may need a sign-out / sign-in to flush its cached connection list.

### Verification

From a tailnet client, before debugging Plexamp:

```bash
curl -v http://<tailscale-ip>:32400/identity
```

A 200 response with `machineIdentifier` in the XML means the network + firewall path is good and any remaining issue is in Plex's discovery (custom URL not yet propagated, Plexamp cache stale, secure-connection setting too strict).

---

## Deployment

### Prerequisites

- Your NAS NFS share mounted on the host (e.g. `/mnt/nas`; see the top-level `system/<hostname>/fstab.example`)
- NVIDIA Container Toolkit installed and Docker daemon configured with `nvidia` runtime
- A claim token from [plex.tv/claim](https://www.plex.tv/claim) — log in to your Plex account first, the token appears on that page; valid for ~4 minutes

### Bring up

```bash
cd /opt/stacks/plex
cp .env.example .env

# Fetch claim token from https://www.plex.tv/claim and paste into .env
nano .env  # set PLEX_CLAIM

# Pre-create local dirs (match your ${PLEX_CONFIG} / ${PLEX_TRANSCODE})
sudo mkdir -p /path/to/plex/{library,transcode}

docker compose up -d
```

### First-run setup

1. Open `http://<host-ip>:32400/web` from a browser that can reach the host (LAN or tailnet).
2. Plex auto-detects the claim token from the environment and binds the server to your Plex account.
3. Add libraries:
   - **Movies** → `/movies`
   - **TV Shows** → `/tv`
   - **Music** → `/music`
4. Configure off-LAN access for remote clients (Plexamp, mobile, etc.):
   - See [Tailscale Access & Plexamp](#tailscale-access--plexamp) if you're using a tailnet — set Custom server access URLs, add the CGNAT range to LAN Networks, set Secure connections to `Preferred`
   - Leave **Remote Access** disabled if a VPN/tailnet is your remote path
5. **Enable hardware transcoding** (requires Plex Pass — see [GPU Acceleration](#gpu-acceleration)).
6. Verify GPU usage with `nvidia-smi` during a transcode.
7. Clear `PLEX_CLAIM` from `.env` — it's no longer needed and would cause confusion on subsequent restarts (Plex ignores expired tokens, but it's clutter).

---

## Backup

Paths to include in Kopia / Borgmatic snapshots:

- `${PLEX_CONFIG}` — users, library DB, metadata cache, watch history, plugin config

Skip from snapshots:
- `${PLEX_TRANSCODE}` — volatile scratch, regenerated as needed
- The media paths themselves — bulk media on the NAS, backed up at the NAS level

> Plex's metadata cache inside `${PLEX_CONFIG}/Library/Application Support/Plex Media Server/Metadata` can grow to tens of GB on large libraries. Rebuildable but expensive to regenerate. Keep it in snapshots.

---

## Notes & Gotchas

- **Host networking is non-negotiable for DLNA.** Plex's discovery features (smart-TV pickup, Plex Companion, GDM) all rely on UDP multicast that doesn't traverse Docker bridge networks. Switching to bridge mode breaks every TV app that finds Plex by browsing the network.
- **No `no-new-privileges`.** Intentionally omitted. Plex's binary uses setuid helpers for certain features, and combined with host networking, the security flag adds friction without meaningful benefit. Jellyfin has the same exception on slightly different grounds (NVIDIA Container Toolkit compatibility).
- **`PLEX_CLAIM` is one-shot.** The token from plex.tv/claim is valid for ~4 minutes. After the server claims itself, the token is useless. If you're restoring on a new host or need to re-claim, generate a fresh token first.
- **Hardware transcoding requires Plex Pass.** Without it, all transcodes happen on the CPU even if NVENC is configured. The GPU acceleration in this compose is wired up; whether it's actually used depends on your Plex subscription.
- **`runtime: nvidia` is legacy.** The modern way is the `deploy.resources.reservations.devices` block, which is also present. Two GPU grants are harmless but redundant; remove `runtime: nvidia` next time you touch compose.
- **Transcode dir on local NVMe — non-negotiable.** Same reason as Jellyfin: Plex writes intermediate transcode segments at high frequency, NFS introduces latency that produces playback stutter.
- **Plexamp has no manual server entry.** Unlike the main Plex apps, Plexamp's UI offers no way to type in a server address — it consumes plex.tv's connection list verbatim. If the tailnet URL isn't in **Custom server access URLs**, Plexamp won't find the server off-LAN regardless of how healthy the actual network path is. See [Tailscale Access & Plexamp](#tailscale-access--plexamp).
- **Plex coexists with Jellyfin.** Both serve the same library files. They don't conflict because each has its own metadata DB; each app builds its own watch history against the shared NAS paths. Both can run simultaneously without issue.

---

## References

- [LinuxServer.io — docker-plex](https://hub.docker.com/r/linuxserver/plex)
- [Plex Media Server — hardware transcoding](https://support.plex.tv/articles/115002178853-using-hardware-accelerated-streaming/)
- [Plex Media Server — networking](https://support.plex.tv/articles/200430283-network/)
- [Plex Media Server — custom server access URLs](https://support.plex.tv/articles/200430283-network/#toc-1)
- [plex.tv/claim — generate claim token](https://www.plex.tv/claim)
- [Top-level README](../../README.md)
