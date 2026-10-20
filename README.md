# Eviworx — Deployment Guide

Full installation guide: https://docs.eviworx.com/en/installation (German: https://docs.eviworx.com/de/installation)

## Prerequisites

- Docker Engine 24+ and Docker Compose v2
- Registry access for your license (see below)
- TLS certificate (PEM format) for your domain
- Min. 8 GB RAM, 4 CPU cores

## 1. Clone

```bash
git clone https://github.com/eviworx/eviworx-deploy.git eviworx
cd eviworx
```

```
eviworx/
  docker-compose.yaml           # Compose file
  .env.example                  # Template for .env (secrets & configuration)
  certs/                        # Place cert.pem + cert.key here
  traefik/
    traefik.yml                 # Traefik static config
    dynamic.yml                 # Traefik routing config
```

## 2. Log in to the Registry

Every license has its own registry access: the user `robot$eviworx+<license key in lower case>` and a token.
You receive them by email — with the trial in "Your Eviworx trial is ready", with a purchased license in
"Your access to the Eviworx registry" (sent to the technical contact).

```bash
docker login registry.eviworx.com -u 'robot$eviworx+<license-key-in-lowercase>'
# Password: your registry token
```

- Keep the **single quotes** around the user. Without them the shell reads `$eviworx` as a variable and Docker reports `unauthorized`.
- The **token is the password**.
- `unauthorized` on `docker compose pull` with correct quotes: the token is outdated (a converted or extended trial gets a new one by email — run `docker login` again) or the license has expired. Contact support@eviworx.com.

## 3. TLS Certificate

```bash
cp /path/to/your/cert.pem certs/cert.pem   # full chain
cp /path/to/your/cert.key certs/cert.key

# Or a self-signed certificate for testing:
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout certs/cert.key -out certs/cert.pem -subj "/CN=localhost"
```

## 4. Create `.env`

```bash
cp .env.example .env
```

Fill in every required value — `docker compose` refuses to start while one is missing:

```bash
openssl rand -hex 32   # POSTGRES_PASSWORD, JOBWORKER_DB_PASSWORD, READONLY_DB_PASSWORD, REDIS_PASSWORD,
                       # SHARE_SECRET, INTERNAL_API_KEY, LICENSE_ENCRYPTION_KEY, TWO_FACTOR_ENCRYPTION_KEY
openssl rand -hex 48   # JWT_SECRET
```

Also set `FRONTEND_URL` and `ADMIN_INITIAL_PASSWORD` (keep the latter in `.env` after the first start).

## 5. Start

```bash
docker compose pull
docker compose up -d
```

First start takes approx. 2-3 minutes (database migration + seed).

## 6. Verify

```bash
# All containers running?
docker compose ps

# Backend health
curl -k https://localhost/api/health/live

# Logs (if something is wrong)
docker compose logs backend --tail 50
docker compose logs db --tail 50
```

## Reverse Proxy / Load Balancer

If Eviworx runs behind an external reverse proxy (e.g. Nginx, HAProxy), two things need to be configured so that client IPs are logged correctly instead of the proxy IP:

### 1. Frontend — `TRUSTED_PROXIES` in `.env`

```env
TRUSTED_PROXIES=198.51.100.0/24,203.0.113.0/24
```

Comma-separated CIDR ranges. Docker-internal and private networks (`172.16.0.0/12`, `10.0.0.0/8`, `192.168.0.0/16`) are always trusted by default.

### 2. Traefik — `forwardedHeaders.trustedIPs` in `traefik/traefik.yml`

Add your external proxy's CIDR to the `trustedIPs` list in **both** `web` and `websecure` entrypoints so Traefik doesn't overwrite the `X-Forwarded-For` header:

```yaml
entryPoints:
  web:
    forwardedHeaders:
      trustedIPs:
        - "10.0.0.0/8"
        - "172.16.0.0/12"
        - "192.168.0.0/16"
        - "198.51.100.0/24"     # <-- your proxy CIDR
  websecure:
    forwardedHeaders:
      trustedIPs:
        - "10.0.0.0/8"
        - "172.16.0.0/12"
        - "192.168.0.0/16"
        - "198.51.100.0/24"     # <-- your proxy CIDR
```

After editing, restart Traefik: `docker compose restart traefik`

## Outbound Proxy

Behind a corporate proxy, set it in `.env` — the backend and all workers then send every outbound
HTTP(S) call through it: license server (activation, heartbeat), Entra ID, Microsoft Graph, Teams,
Webex, Web Push, Cloudflare Turnstile, webhooks. Without these values everything goes direct.

```env
HTTPS_PROXY=http://proxy.company.example:3128
HTTP_PROXY=http://proxy.company.example:3128
# optional: targets that bypass the proxy (host names or .domain suffixes, NO CIDR ranges)
NO_PROXY=erp.internal.company.com,.intranet.company.com
```

Then recreate: `docker compose up -d`.

- **Internal traffic never uses the proxy.** The compose file carries the internal service names in
  `NO_PROXY`; your `NO_PROXY` is appended. Internal webhook targets you allow in `SSRF_ALLOWLIST` must
  also be listed in `NO_PROXY` by name — ranges are not evaluated.
- **TLS inspection:** put the PEM of your corporate CA into `certs/proxy/` and set
  `NODE_EXTRA_CA_CERTS=/etc/eviworx/proxy-ca/<file>.pem`. A missing file is ignored without a
  warning — check it: `docker compose exec backend node -e "fetch('https://lic.eviworx.com').then(r => console.log(r.status))"`
  (a TLS error means the CA is not loaded).
- **Antivirus updates (ClamAV):** use `HTTPS_PROXY` unless you set `CLAMAV_HTTPS_PROXY` — empty means
  direct, a value means its own proxy. freshclam does not read `NODE_EXTRA_CA_CERTS`; exempt the
  ClamAV mirror (`database.clamav.net`) from TLS inspection.
- **Email:** IMAP and SMTP connect directly, never through an HTTP proxy. For Microsoft 365 behind a
  proxy use the Graph connection; otherwise open the mail ports in the firewall.
- **Webhooks without external DNS:** webhook targets are checked before every call (private networks,
  loopback, metadata are blocked). If the containers cannot resolve external names at all and only the
  proxy can, set `SSRF_PROXY_RESOLVES_DNS=true`: names the container cannot resolve are then accepted
  and resolved by the proxy — the proxy ACL is the boundary for them, so block private networks there
  for this host.
- `LICENSE_SERVER_URL` is not needed for a proxy; leave it at its default.

## 7. Product License

Without a license Eviworx starts unlicensed (read-only). Activate your license key and secret — from your
purchase or from the trial (https://lic.eviworx.com/en/trial) — in the web UI after the first login:
**Admin Center > System > Product License**. Entries in `.env` do not activate a license.

Air-gapped installations activate offline with a code from https://lic.eviworx.com/en/activate; after a
renewal they need a new code. Details, rehosting and the data shared with the license server:
https://docs.eviworx.com/en/license

## Update to a New Version

Get the new compose file (it carries the new image tags), then back up and recreate:

```bash
git pull
```


```bash
# 1. Database backup BEFORE the migration runs (schema migrations are not reversible).
#    Dump to a file INSIDE the container and copy it out — piping pg_dump through
#    `docker exec … > file` can produce truncated dumps.
mkdir -p backups && chmod 700 backups
docker exec eviworx-db sh -c 'pg_dump -U "$POSTGRES_USER" -d "$POSTGRES_DB" -Fc -f /tmp/pre.dump \
  && pg_restore --list /tmp/pre.dump | tail -1'      # must print a TOC line, no error
docker cp eviworx-db:/tmp/pre.dump backups/pre-$(date +%Y%m%d-%H%M).dump
docker exec eviworx-db rm /tmp/pre.dump

# 2. Pull and recreate — the backend entrypoint applies pending migrations on start;
#    workers wait for the backend to become healthy (depends_on).
docker compose pull
docker compose up -d --force-recreate

# 3. Verify
docker compose ps
docker compose exec -T backend npx prisma migrate status
```

Rollback: restore the compose file with the previous tags **and** restore the
database dump from step 1 — an old backend image does not run against a newer
schema. Restore the same way (copy in, then run inside the container), with
backend and workers stopped:

```bash
docker cp backups/pre-<stamp>.dump eviworx-db:/tmp/restore.dump
docker exec eviworx-db sh -c 'pg_restore -U "$POSTGRES_USER" -d "$POSTGRES_DB" --clean --if-exists /tmp/restore.dump'
```

Release-specific notes come with the release announcement.

## Services

| Service | Port | Description |
|-|-|-|
| traefik | 80, 443 | Reverse proxy + TLS termination |
| frontend | internal | React SPA (NGINX) |
| backend | internal (3000) | API + Auth + WebSocket |
| db | internal | PostgreSQL 17 |
| redis | internal | Redis 8 (cache + queue) |
| email-worker | internal | Send/receive emails |
| job-worker | internal | CronJobs + background tasks |
| workflow-engine | internal | Workflow automation |
| notification-worker | internal | Webex/Teams/Email notifications |
| clamav | internal | Antivirus scanner |
| av-worker | internal | Attachment scanning |
| report-generator | internal | Custom reports + export |

## License

Copyright 2026 Eviworx Software UG (haftungsbeschränkt)

The files in this repository are licensed under the [Apache License 2.0](LICENSE). The Eviworx container
images and the software they contain are proprietary and require a valid Eviworx license.
