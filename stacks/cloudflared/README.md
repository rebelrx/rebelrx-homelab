# cloudflared

Cloudflare Tunnel client for exposing services to the internet without opening inbound ports. Cloudflared makes an outbound-only connection to Cloudflare's edge; public hostnames and routing rules live in the Cloudflare Zero Trust dashboard.

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `cloudflared` | `cloudflare/cloudflared:latest` | Cloudflare Tunnel daemon — outbound connector to Cloudflare's edge |

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Reach the proxied services (e.g. NPM) that tunnel routes point at |

> Cloudflared joins `proxy_net` so its ingress rules can target other containers by name (e.g. `http://npm:80`). Adjust to whatever network your backends live on.

---

## Ports

**None.** Cloudflared establishes an outbound-only connection to Cloudflare's edge — no inbound ports are opened or published. This is the whole point of the tunnel model.

---

## Volumes

None. Token-based tunnels are stateless on the host; all configuration lives in the Cloudflare Zero Trust dashboard. Nothing to bind mount or back up locally.

---

## Environment Variables

### `.env` (Compose interpolation)

| Variable | Example | Purpose |
|----------|---------|---------|
| `TUNNEL_TOKEN` | *(secret)* | Cloudflare Tunnel token from the Zero Trust dashboard — required, compose refuses to start if empty |

> Keep the `.env` file mode at `600`. The token grants control of the tunnel; treat it like a password.

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing)
- A Cloudflare account with a domain onboarded to Cloudflare DNS
- A tunnel created in the [Cloudflare Zero Trust dashboard](https://one.dash.cloudflare.com/) with its token

### Bring up

```bash
cd /opt/stacks/cloudflared
cp .env.example .env
nano .env   # paste TUNNEL_TOKEN
chmod 600 .env

docker compose up -d
```

### First-run setup

1. In the Zero Trust dashboard, create a tunnel (**Networks → Tunnels → Create a tunnel → Cloudflared**) and copy the token into `.env` as `TUNNEL_TOKEN`.
2. Under the tunnel's **Public Hostnames**, add each service you want to expose — set the public hostname (e.g. `app.example.com`) and the internal service URL (e.g. `http://npm:80`, reachable over `proxy_net`).
3. Start the stack; the tunnel should show **Healthy** in the dashboard within a few seconds.
4. Public routing and access policies are all managed in the dashboard, not in this compose file.

---

## Notes & Gotchas

- **Outbound-only, no inbound ports.** The tunnel dials out to Cloudflare, so no firewall/NAT changes are needed on the host. This is a different exposure model from the reverse-proxy stacks in this repo — Cloudflared terminates at Cloudflare's edge, NPM terminates on your LAN.
- **No in-container healthcheck.** The `cloudflared` image is minimal and ships no shell utilities (`wget`/`curl`), so a compose `healthcheck` isn't practical. Monitor tunnel health from the Zero Trust dashboard instead, where the connector reports Healthy/Down.
- **Config lives in Cloudflare, not here.** Public hostnames, path routing, and access policies are all configured in the dashboard. The compose file only supplies the token and the network attachment.
- **Token = full tunnel control.** Anyone with the token can run a connector for your tunnel. Keep `.env` at `600` and never commit it.

---

## References

- [cloudflared on GitHub](https://github.com/cloudflare/cloudflared)
- [Cloudflare Tunnel docs](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
- [Cloudflare Zero Trust dashboard](https://one.dash.cloudflare.com/)
- [Top-level README](../../README.md)
