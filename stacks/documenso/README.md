# documenso

Self-hosted document signing platform with [Documenso](https://documenso.com/) — open-source DocuSign alternative.

Publicly accessible via VPS-side Caddy → Tailscale → Documenso on the homelab host (see [Reverse Proxy](#reverse-proxy)).

---

## Services

| Service | Image | Purpose |
|---------|-------|---------|
| `documenso` | `documenso/documenso:latest` | Documenso web app, signing engine, API |
| `documenso-db` | `postgres:15` | PostgreSQL database |

---

## Networks

| Network | Type | Purpose |
|---------|------|---------|
| `proxy_net` | External | Internal reverse-proxy plane (LAN access via NPM, if configured) |
| `documenso` | Internal bridge | App ↔ PostgreSQL |

> Only `documenso` joins `proxy_net`. The DB is reachable only on the internal `documenso` network.

---

## Ports

| Binding | Container port | Reachable from | Purpose |
|---------|----------------|----------------|---------|
| `${TAILSCALE_IP}:3000` | `3000` | Tailnet peers only | Public-facing access via VPS Caddy reverse proxy |

> The app is bound **only to the host's Tailscale IP** — not `0.0.0.0` and not `127.0.0.1`. This exposes Documenso to tailnet peers (including the VPS Caddy reverse proxy) without exposing it to the LAN or the public internet directly. The host port binding is required because the VPS is on the tailnet, not on `proxy_net`.
>
> **Running standalone without the VPS/Tailscale bastion?** Replace the `${TAILSCALE_IP}:3000:3000` mapping in `compose.yaml` with `3000:3000` to publish on all interfaces (LAN-reachable), or `127.0.0.1:3000:3000` for loopback-only behind a local reverse proxy. If you do, `TAILSCALE_IP` in `.env` becomes unused.
>
> If you also want LAN-only access via NPM, add an NPM proxy host pointing to `documenso:3000` over `proxy_net` — but note `NEXT_PUBLIC_WEBAPP_URL` is the single source of truth for email links, so the public URL is what recipients see regardless.

---

## Volumes

| Host path (`.env` var) | Container path | Mode | Purpose |
|------------------------|----------------|------|---------|
| `${DOCUMENSO_CERT}` | `/opt/documenso/cert.p12` | **ro** | PKCS#12 signing certificate (used to cryptographically seal completed PDFs) |
| `${DOCUMENSO_DB}` | `/var/lib/postgresql/data` | rw | PostgreSQL data directory |

> **Important:** the cert is mounted as a **single file**, not a directory. The host path `${DOCUMENSO_CERT}` must exist as a regular file *before* `docker compose up` — otherwise Docker silently creates a directory at that path and the bind mount becomes a directory-to-file mismatch. See the [Deployment](#deployment) section for the correct cert placement order.
>
> The cert file must be readable by the container's `nodejs` user (UID `1001`, GID `65533`). Recommended: `chown 1001:65533 cert.p12 && chmod 640 cert.p12`.

---

## Environment Variables

### `.env` (Compose interpolation)

#### App config

| Variable | Example | Purpose |
|----------|---------|---------|
| `NEXT_PUBLIC_WEBAPP_URL` | `https://documenso.example.com` | Public URL — baked into all signing-request and completion email links. Changing this breaks in-flight envelopes. |
| `NEXT_PRIVATE_INTERNAL_WEBAPP_URL` | `http://documenso:3000` | Internal URL used for container-to-container job submission. Must resolve from inside the container. |
| `TAILSCALE_IP` | `100.x.y.z` | Host Tailscale IP, used as the bind address for the published port. Find with `tailscale ip -4` on the host. |

#### Secrets

| Variable | Purpose |
|----------|---------|
| `NEXTAUTH_SECRET` | NextAuth session signing key. Generate with `openssl rand -hex 32` |
| `NEXT_PRIVATE_ENCRYPTION_KEY` | Primary at-rest encryption key for stored data (min 32 chars). Generate with `openssl rand -hex 32` |
| `NEXT_PRIVATE_ENCRYPTION_SECONDARY_KEY` | Secondary key for rotation (min 32 chars). Generate with `openssl rand -hex 32` |
| `NEXT_PRIVATE_SIGNING_PASSPHRASE` | Passphrase for the `cert.p12` PKCS#12 cert. **Must match the passphrase used when generating the cert.** Store in a password manager — if lost, the cert must be regenerated. |

#### Database

| Variable | Example | Purpose |
|----------|---------|---------|
| `POSTGRES_USER` | `documenso` | Postgres username |
| `POSTGRES_PASSWORD` | *(secret)* | Postgres password. Generate with `openssl rand -base64 32 \| tr -d '\n=/+' \| head -c 32` |
| `POSTGRES_DB` | `documenso` | Postgres database name |

> The DB connection URL is constructed inside `compose.yaml` as `postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@documenso-db:5432/${POSTGRES_DB}` and passed to Documenso as both `NEXT_PRIVATE_DATABASE_URL` and `NEXT_PRIVATE_DIRECT_DATABASE_URL`. No need to set those directly.

#### SMTP (Resend)

| Variable | Example | Purpose |
|----------|---------|---------|
| `NEXT_PRIVATE_SMTP_HOST` | `smtp.resend.com` | SMTP server hostname |
| `NEXT_PRIVATE_SMTP_PORT` | `587` | SMTP port (587 = STARTTLS, 465 = implicit TLS — both work with Resend) |
| `NEXT_PRIVATE_SMTP_USERNAME` | `resend` | Literal string `resend` for Resend SMTP |
| `NEXT_PRIVATE_SMTP_PASSWORD` | *(Resend API key, `re_...`)* | Sending-scoped API key from the Resend dashboard |
| `NEXT_PRIVATE_SMTP_FROM_NAME` | `Documenso` | Display name on outbound emails |
| `NEXT_PRIVATE_SMTP_FROM_ADDRESS` | `no-reply@mail.example.com` | From address. **Domain must exactly match a verified Resend domain** — Resend rejects with `550 This API key is not authorized to send emails from <domain>` otherwise. |

> SMTP transport is hardcoded to `smtp-auth` in `compose.yaml`. Documenso also supports the Resend REST API, AWS SES, and Mailgun — to switch, change `NEXT_PRIVATE_SMTP_TRANSPORT` in compose and adjust env vars per Documenso's transport docs.

#### Signing

| Variable | Example | Purpose |
|----------|---------|---------|
| `NEXT_PRIVATE_SIGNING_TRANSPORT` | `local` | Use a local PKCS#12 file (rather than a cloud KMS) for signing |
| `NEXT_PRIVATE_SIGNING_LOCAL_FILE_PATH` | `/opt/documenso/cert.p12` | **Container path** to the cert (must match the right-hand side of the volume mount) |
| `NEXT_PRIVATE_SIGNING_PASSPHRASE` | *(secret)* | See Secrets section above |

> Without `NEXT_PRIVATE_SIGNING_TRANSPORT=local` and `NEXT_PRIVATE_SIGNING_LOCAL_FILE_PATH`, Documenso falls back to a hardcoded default of `./example/cert.p12` (a dev-only path that doesn't exist in the production container), causing the `internal.seal-document` background job to fail with ENOENT after every signing flow.

#### Access control

| Variable | Example | Purpose |
|----------|---------|---------|
| `NEXT_PUBLIC_DISABLE_SIGNUP` | `true` | Disables the public signup form entirely. Admin must provision new users (via SQL or admin panel). Recommended for single-user / closed instances. |

> Alternative: `NEXT_PRIVATE_ALLOWED_SIGNUP_DOMAINS=domain1,domain2` (comma-separated, no spaces) whitelists signup by email domain — useful for multi-user instances where you control the email domain. The two options are mutually exclusive in practice; use one or the other.

> External recipients signing documents are **unaffected** by either setting — they sign via tokenized magic links without needing accounts.

#### Telemetry

| Variable | Example | Purpose |
|----------|---------|---------|
| `DOCUMENSO_DISABLE_TELEMETRY` | `true` | Disable anonymous telemetry (app version, install ID, node ID only — no document or user data). Omit to opt in. |

#### Volumes

| Variable | Example | Purpose |
|----------|---------|---------|
| `DOCUMENSO_CERT` | `/path/to/documenso/certs/cert.p12` | Host path to PKCS#12 signing cert |
| `DOCUMENSO_DB` | `/path/to/documenso/postgres` | Host path to Postgres data dir |

---

## Reverse Proxy

### Public (VPS Caddy → Tailnet)

External traffic enters at the VPS, where Caddy terminates TLS and forwards over the tailnet to the homelab host on its Tailscale IP.

```caddy
documenso.example.com {
    encode zstd gzip
    request_body {
        max_size 50MB
    }
    reverse_proxy 100.x.y.z:3000 {
        header_up X-Forwarded-Proto https
        header_up X-Real-IP {remote_host}
    }
    log {
        output file /var/log/caddy/documenso.log {
            roll_size 50mb
            roll_keep 10
        }
        format json
    }
}
```

Notes:
- `X-Forwarded-Proto https` is **required**; without it, Next.js generates `http://` links in outbound emails.
- `request_body { max_size 50MB }` — Caddy's default 10MB body limit will reject larger PDF uploads.
- `100.x.y.z` is the host's Tailscale IP (`tailscale ip -4`).
- DNS: A record for `documenso.example.com` → VPS public IP, DNS-only (or proxied) to match the convention used by your other VPS-fronted services.

### Internal (NPM, optional)

If you want LAN/tailnet-direct access bypassing the VPS, add an NPM proxy host:

| Hostname | Upstream host | Upstream port | Scheme |
|----------|---------------|---------------|--------|
| *(internal hostname)* | `documenso` | `3000` | `http` |

Set NPM Advanced tab: `client_max_body_size 50M;` for the same reason as the Caddy `request_body` directive.

> Note: `NEXT_PUBLIC_WEBAPP_URL` is the single source of truth for email links — internal NPM access works for the UI but signing-request emails will still contain the public URL.

---

## Deployment

### Prerequisites

- `proxy_net` external network (`docker network create proxy_net` if missing); `documenso` internal network is created automatically by compose
- Tailscale running on the host with a known Tailscale IP
- A Resend account with the sending domain verified (DNS records added at your DNS provider)
- A Resend API key scoped to **Sending access** for the verified domain
- VPS Caddy and DNS configured per the [Reverse Proxy](#reverse-proxy) section

### Step 1 — Generate the signing certificate

The cert is a PKCS#12 bundle of a self-signed cert and its private key, used to cryptographically seal completed PDFs. Generate it before bringing up the stack — see the [Volumes](#volumes) note about the bind-mount footgun.

```bash
# Working directory
mkdir -p ~/documenso-cert && cd ~/documenso-cert

# Private key + self-signed cert (10-year validity)
openssl genrsa -out private.key 2048
openssl req -new -x509 -key private.key -out certificate.crt -days 3650 \
  -subj "/CN=Documenso Self-Hosted/O=Homelab/C=US"

# Generate and CAPTURE the passphrase
PASS=$(openssl rand -base64 24)
echo ""
echo "================================================"
echo "PASSPHRASE (copy to password manager NOW):"
echo "$PASS"
echo "================================================"
read -p "Press Enter once saved..."

# Bundle into PKCS#12
openssl pkcs12 -export \
  -out cert.p12 \
  -inkey private.key \
  -in certificate.crt \
  -passout pass:"$PASS"

# Place at the host volume path (matches ${DOCUMENSO_CERT}) — directory first, then file
sudo mkdir -p /path/to/documenso/certs
sudo mv cert.p12 /path/to/documenso/certs/cert.p12

# Ownership must match the container's `nodejs` user (UID 1001, GID 65533)
sudo chown 1001:65533 /path/to/documenso/certs/cert.p12
sudo chmod 640 /path/to/documenso/certs/cert.p12

# Clean up
rm ~/documenso-cert/private.key ~/documenso-cert/certificate.crt
```

Verify the cert is a regular file (not a directory) and the passphrase matches:

```bash
file /path/to/documenso/certs/cert.p12
# → regular file, no read permission  (the "no read permission" is for your shell user, not the container — that's correct)

sudo openssl pkcs12 -in /path/to/documenso/certs/cert.p12 -noout -passin pass:'<your passphrase>'
# → silent return = passphrase correct; "Mac verify error" = mismatch (regenerate)
```

### Step 2 — Resend setup

1. Sign up at [resend.com](https://resend.com).
2. **Domains → Add Domain**: use a dedicated subdomain like `mail.example.com` (keeps sending reputation isolated from your apex domain).
3. Add the DNS records Resend shows (MX, SPF, DKIM, optionally DMARC) to your DNS provider. Wait for verification (usually 5–30 minutes).
4. **API Keys → Create API Key**: name it `documenso-prod`, permission **Sending access**, domain scoped to your verified domain. Copy the key (`re_...`) — Resend only shows it once.

### Step 3 — Bring up the stack

```bash
cd /opt/stacks/documenso
cp .env.example .env

# Generate secrets
echo "NEXTAUTH_SECRET=$(openssl rand -hex 32)" >> .env
echo "NEXT_PRIVATE_ENCRYPTION_KEY=$(openssl rand -hex 32)" >> .env
echo "NEXT_PRIVATE_ENCRYPTION_SECONDARY_KEY=$(openssl rand -hex 32)" >> .env
echo "POSTGRES_PASSWORD=$(openssl rand -base64 32 | tr -d '\n=/+' | head -c 32)" >> .env

# Fill in the rest interactively
nano .env  # Tailscale IP, public URL, Resend SMTP block, signing passphrase
chmod 600 .env

docker compose up -d
docker compose logs -f documenso
```

You should see:

```
🔐 Checking certificate configuration...
✅ Certificate file found and readable - document signing is ready!
```

If you see `⚠️ Certificate not found or not readable`, the cert path/permissions/ownership are wrong — recheck Step 1.

### Step 4 — First-run setup

1. Wait ~90 seconds for Documenso to run database migrations (matches the `start_period` on the healthcheck).
2. Open the app at your public URL (e.g. `https://documenso.example.com`) and create the first admin account.
3. **End-to-end test**: create a test envelope with yourself + an external Gmail/Outlook recipient, sign through, and verify:
   - Signing-request emails arrive at both addresses (check Resend dashboard → Emails for delivery confirmation)
   - Status transitions from "Pending" → "Completed" within seconds of the last signature
   - Completion emails arrive with the signed PDF attached
   - Opening the signed PDF in Adobe Reader shows the signature panel (validity will display as "unknown" since the cert is self-signed — this is expected for self-hosted instances)

---

## Backup

Critical paths to include in Kopia / Borgmatic snapshots:

- `${DOCUMENSO_DB}` — PostgreSQL data (signed documents, audit logs, users)
- `${DOCUMENSO_CERT}` — signing cert. **Without it, previously signed documents still verify (the cert is embedded in their signatures), but Documenso can't seal new documents.**
- `.env` — contains encryption keys, SMTP credentials, and signing passphrase. **Without `NEXT_PRIVATE_ENCRYPTION_KEY`, the DB is partially unrecoverable** — encrypted-at-rest fields (some user data, integration secrets) won't decrypt.

> For Postgres specifically, a periodic `pg_dump` is safer than relying solely on file-level snapshots:
>
> ```bash
> docker compose exec -T documenso-db pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB" \
>   | gzip > /path/to/documenso/dumps/documenso-$(date +%F).sql.gz
> ```

---

## Notes & Gotchas

- **`NEXT_PUBLIC_WEBAPP_URL` is baked into outbound emails.** Changing it invalidates the links in any in-flight signing requests. Complete or void pending envelopes before changing the public URL.
- **The bind-mount footgun.** If `${DOCUMENSO_CERT}` doesn't exist as a regular file at `docker compose up` time, Docker silently creates a **directory** at that host path. The cert mount then becomes a directory-to-file mismatch, and Documenso fails to seal documents with a confusing "certificate not found" error even though the env vars look correct. **Always place the cert file before first compose-up.**
- **Cert file ownership matters.** The container runs as `nodejs` (UID 1001, GID 65533). `chmod 600` works only if the file is *owned by* UID 1001. Easier: `chown 1001:65533` + `chmod 640`. The `:ro` volume mount plus the cert's passphrase protect the file at rest.
- **Resend from-address domain match.** Resend rejects sends from any domain not verified on your account, with a 550 SMTP error. The `NEXT_PRIVATE_SMTP_FROM_ADDRESS` domain must exactly match a verified Resend domain (e.g. the subdomain `mail.example.com`, not the apex `example.com`).
- **Half-signed envelopes from a previous broken state.** If signing fails at the seal step (e.g. cert issue), the envelope can be left in a "signed but not sealed" state. The internal `seal-document-sweep` cron runs every 15 minutes and *may* retry, but documents that exceeded the retry limit (`BackgroundTaskExceededRetriesError`) won't auto-recover. Delete them from the UI and create a fresh test envelope.
- **Signed documents are legally meaningful.** The `cert.p12` baked into a signed PDF becomes part of its trust chain. If you swap the cert later, previously signed documents still verify against the old cert (it's embedded in the signature), but new signatures use the new cert. Don't delete old certs — archive them.
- **Self-signed cert displays as "unknown identity" in Adobe Reader.** The signature is still cryptographically valid and tamper-evident — Adobe just doesn't recognize a self-signed issuer. For internal/personal workflows this is fine; for legally binding documents requiring CA-backed signing, buy a document-signing cert from a recognized CA.
- **Encryption keys cannot be lost casually.** `NEXT_PRIVATE_ENCRYPTION_KEY` and its secondary are used to encrypt sensitive fields in the DB. If both are lost, those fields are unrecoverable. Store them in a password manager outside this host.
- **Admin role is granted by DB, not UI.** Documenso's OSS build doesn't expose an admin-promotion UI. To grant `ADMIN` (which unlocks `/admin` for instance management): `docker exec -it documenso-db psql -U documenso -d documenso -c "UPDATE \"User\" SET roles = ARRAY['ADMIN', 'USER']::\"Role\"[] WHERE email = '<user_email>';"` then log out and back in.
- **Account deletion can fail with "unknown error" if you're the sole org owner.** The UI's delete flow blocks orphaning organisations but the error message is unhelpfully generic. Either delete the org first, or wipe the DB (`docker compose down && sudo rm -rf /path/to/documenso/postgres && docker compose up -d`) for a clean restart — appropriate when there are no historical documents to preserve.
- **Image is `:latest`.** WUD watches `^latest$` only — Documenso doesn't currently publish stable version tags via Docker Hub in a way that'd let you pin to a release line. Treat updates as "test in a fresh tab before approving".

---

## References

- [Documenso self-hosting docs](https://docs.documenso.com/docs/self-hosting)
- [Documenso GitHub](https://github.com/documenso/documenso)
- [Resend SMTP docs](https://resend.com/docs/send-with-smtp)
- [Top-level README](../../README.md)
