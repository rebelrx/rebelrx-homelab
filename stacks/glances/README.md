# glances

System + GPU metrics exporter feeding a Home Assistant fleet-monitoring dashboard. Runs the Glances REST API (v4) in host-network mode; Home Assistant's Glances integration polls it over the LAN.

> LAN-only monitoring endpoint. Not exposed through NPM or the public edge, and it ships with no API authentication — access is gated at the host firewall (UFW) to the LAN subnet.

---

## Services

| Service | Image | Description |
|---|---|---|
| `glances` | `nicolargo/glances:ubuntu-latest-full` | Glances in web-server mode (`-w`), serving the REST API v4 + web UI on `:61208`. The **Ubuntu** image is required for NVIDIA GPU monitoring — the Alpine `latest-full` build has no NVML / driver support. |

---

## Networks

| Network | Type | Notes |
|---|---|---|
| host | host | The container shares the host network namespace (`network_mode: host`). Deliberate: host-accurate CPU/mem/network/temperature stats, and the listener is a real host socket that **UFW can filter** (a Docker `-p` publish would DNAT-bypass UFW). |

> This stack does **not** join `proxy_net` and is **not** reverse-proxied.

---

## Ports

| Port | Bind | Proto | Purpose |
|---|---|---|---|
| 61208 | `0.0.0.0` (host mode) | tcp | Glances REST API v4 + web UI. |

> Because this runs in host-network mode, there is **no Docker `ports:` mapping** — the container binds the host socket directly on all interfaces (including `tailscale0`). You can't scope it to loopback via compose the way the reverse-proxied stacks do; the security boundary is the **UFW rule** scoping it to the LAN subnet — see Deployment. To keep it off the tailnet as well: `sudo ufw deny in on tailscale0 to any port 61208 proto tcp`.

---

## Volumes

| Source | Target | Mode | Purpose |
|---|---|---|---|
| `/var/run/docker.sock` | `/var/run/docker.sock` | ro | Container list / per-container stats. |
| `/etc/os-release` | `/etc/os-release` | ro | Host OS identification in the `system` plugin. |
| `./glances.conf` | `/etc/glances/glances.conf` | ro | Trimmed config (plugin/interface/fs filtering). |
| `/` | `/rootfs` | ro, **non-recursive** | Host root-fs disk usage. `bind.recursive: disabled` keeps NAS NFS submounts **out** of the container. |
| `tmpfs` | `/tmp` | rw | Writable scratch for the read-only rootfs (Python cache lands here). |

> Glances holds no persistent state, so there is no `/opt/data/glances/` volume.

---

## Environment Variables

### `.env`

| Variable | Example | Description |
|---|---|---|
| `TIMEZONE` | `America/New_York` | Container timezone (uptime/history timestamps). |

### Hardcoded in `compose.yaml`

| Setting | Value | Description |
|---|---|---|
| `GLANCES_OPT` | `-w -C /etc/glances/glances.conf` | Web-server mode + custom config. |
| `PYTHONPYCACHEPREFIX` | `/tmp/py_caches` | Sends Python bytecode cache to the tmpfs so `read_only: true` works. |
| `NVIDIA_VISIBLE_DEVICES` | `all` | Exposes the GPU to the container. |
| `NVIDIA_DRIVER_CAPABILITIES` | `all` | Guarantees the `utility` cap → NVML for the GPU plugin. |
| `read_only` | `true` | Root filesystem read-only (paired with `tmpfs: /tmp`). |
| `pid` / `network_mode` | `host` | Host-accurate stats + UFW-filterable socket. |
| `deploy…devices` | `driver: nvidia, count: 1, capabilities: [gpu]` | GPU reservation. Honored by `docker compose up` (not `docker stack deploy`). |
| WUD labels | notify-only | `wud.watch=true`, `wud.trigger.docker.enable=false`. |

---

## Reverse Proxy

Not applicable. LAN-only API bound directly to the host on `:61208`; it does not pass through NPM / `proxy_net`. Home Assistant reaches it over the LAN. Exposing system-metrics endpoints through the reverse proxy or public edge is intentionally avoided.

---

## Deployment

```bash
# 1. Stack directory + files
mkdir -p /opt/stacks/glances
cd /opt/stacks/glances
# (drop in compose.yaml, .env.example, glances.conf, README.md)

# 2. Create .env from the example and lock it down
cp .env.example .env
$EDITOR .env            # set TIMEZONE
chmod 600 .env

# 3. Bring it up (via Dockge, or directly)
docker compose up -d

# 4. Allow the API from the LAN only (UFW — this host, not firewalld).
#    Replace the subnet with your own LAN range.
sudo ufw allow from 192.168.1.0/24 to any port 61208 proto tcp comment 'Glances API (LAN)'

# 5. Verify: API v4, GPU, trimmed fs + network (replace <host-ip> with the host's LAN IP)
curl -s http://<host-ip>:61208/api/4/status
curl -s http://<host-ip>:61208/api/4/gpu       # GPU present
curl -s http://<host-ip>:61208/api/4/fs        # one entry: /rootfs
curl -s http://<host-ip>:61208/api/4/network | python3 -c "import sys,json;print([i['interface_name'] for i in json.load(sys.stdin)])"
# expect your configured interfaces, e.g.: ['enp4s0', 'tailscale0']
```

> **GPU prerequisite:** NVIDIA driver + Container Toolkit wired into Docker (`nvidia-ctk runtime configure --runtime=docker`, restart Docker). Confirm with `nvidia-smi` on the host and `docker run --rm --gpus all nicolargo/glances:ubuntu-latest-full nvidia-smi`.

---

## Backup

Stateless service. Nothing to snapshot beyond the tracked files (`compose.yaml`, `.env.example`, `glances.conf`, `README.md`) and the local `.env`. Covered by the homelab repo and the Kopia backup of `/opt/stacks`.

---

## Maintenance

WUD watches this image in **notify-only** mode (`wud.trigger.docker.enable=false`); it never auto-pulls.

```bash
# Manual update after reviewing the notification / changelog
cd /opt/stacks/glances
docker compose pull
docker compose up -d
docker image prune -f
curl -s http://<host-ip>:61208/api/4/status
```

> On the rolling `ubuntu-latest-full` tag, add `wud.watch.digest=true` to the labels if you want WUD to notify on rebuilds (not just tag changes). Alternatively pin `image:` to a version (e.g. `ubuntu-4.5.5`) and let WUD watch semver — consistent with the single-maintainer pinning policy.

---

## Notes & Gotchas

> **`glances.conf` uses Python `configparser` — no inline comments.** A `key=value   # comment` line makes the parser read `True   # comment` as the value and **hard-crashes** on startup (`ValueError: Not a boolean`). Comments must be on their own line. The traceback points at `getboolean`, not your file — the tell is the comment text appearing in the error value.

> **`show=` allowlists beat `hide=` blocklists here.** Both `[fs]` and `[network]` collapse cleanly with `show=` (`/rootfs`; your LAN + `tailscale0` interfaces). `hide_no_ip=True` does **not** hide veths — the host-side veth peer carries a link-local IP — so a blocklist kept leaking. Naming the handful you want is robust to new veth/bridge/interface names.

> **`recursive: disabled` on the `/rootfs` bind is load-bearing.** The default (recursive) bind pulls NAS NFS submounts into the container; the `fs` plugin then stats them every cycle → D-state risk on a wedged export. Non-recursive mounts the root device only. `/api/4/fs` confirms it: one entry on the root device, no NFS.

> **Ubuntu image required for GPU.** The Alpine `latest-full` image has no NVML and silently shows no GPU. Use `ubuntu-latest-full` + the `deploy` reservation + `NVIDIA_*=all`.

> **`read_only: true` needs the tmpfs + pycache.** It works on the ubuntu image only because `tmpfs: /tmp` + `PYTHONPYCACHEPREFIX=/tmp/py_caches` give Python somewhere to write. Drop those and the read-only rootfs blocks startup.

> **Home Assistant — imperial → °F.** Temps report °C from Glances but an imperial HA install ingests/displays them as °F (`tctl` reads ~160 °F). Dashboard tiles convert °F→°C inline (`(x-32)/1.8`), or override the unit per-entity to °C.

> **Home Assistant — shedding entities.** The `show`/`hide` rules stop new entities being produced but don't retract ones already registered from an earlier unfiltered run. After loading the conf, **delete and re-add** the integration to register only the trimmed set. Entity IDs are prefixed from the integration instance name (e.g. `sensor.<host>_*`); GPU slugs read `..._nvidia_<model>_gpu_nvidia0_*`.

---

## References

- Glances documentation — https://glances.readthedocs.io/
- Glances Docker image — https://hub.docker.com/r/nicolargo/glances
- Glances configuration reference — https://glances.readthedocs.io/en/latest/config.html
- Home Assistant Glances integration — https://www.home-assistant.io/integrations/glances/
- Docker Compose GPU support — https://docs.docker.com/compose/gpu-support/
