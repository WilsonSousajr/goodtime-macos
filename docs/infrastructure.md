# Infrastructure

This doc covers procuring a VPS, installing **Coolify**, and deploying Supabase via Coolify's built-in template. Coolify handles Docker, Caddy, TLS, env vars, logs, and backups — most hand-rolled setup we'd otherwise need just disappears.

## 1. VPS

**Provider:** Hostinger
**Plan:** KVM 4 — 4 vCPU x86, 16 GB RAM, 200 GB NVMe SSD, ~$8–10/mo (24-month commitment for the best price)
**Region:** São Paulo
**OS:** Ubuntu 24.04 LTS

Hostinger includes automatic weekly VM snapshots (kept ~1 week) as a baseline backup layer. The application-level backup (Postgres `pg_dump` to B2) is configured in Coolify — see § 8.

## 2. DNS

You'll need a domain. Two A records:

```
A    goodtime.<your-domain>.   → <vps-ipv4>     # Supabase API consumed by clients
A    coolify.<your-domain>.    → <vps-ipv4>     # Coolify admin UI
```

Coolify's built-in Caddy obtains Let's Encrypt certs automatically once these records resolve.

## 3. OS provisioning

Hostinger gives root access. Run on the VPS as `root`:

```bash
# Create a deploy user for your own direct SSH (Coolify itself runs services as root via Docker)
adduser --disabled-password --gecos "" deploy
usermod -aG sudo deploy
mkdir -p /home/deploy/.ssh
cp ~/.ssh/authorized_keys /home/deploy/.ssh/
chown -R deploy:deploy /home/deploy/.ssh
chmod 700 /home/deploy/.ssh
chmod 600 /home/deploy/.ssh/authorized_keys

# Baseline updates + tools
apt update && apt upgrade -y
apt install -y ufw fail2ban unattended-upgrades curl ca-certificates
```

SSH hardening, firewall rules, fail2ban, unattended-upgrades: see `docs/security.md`.

## 4. Install Coolify

Coolify's one-line installer pulls down Docker, Caddy, and the Coolify control plane:

```bash
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | sudo bash
```

After install:

1. Open `http://<vps-ipv4>:8000` in a browser and create the admin user (first user wins — this is you).
2. **Coolify Settings → Instance → Instance Domain**: set to `coolify.<your-domain>`. Coolify provisions a Let's Encrypt cert automatically. After this completes, close port `8000` in UFW (see security doc).
3. **Coolify Profile → SSH Keys**: add your SSH key so Coolify can clone your GitHub repo when needed.

## 5. Deploy Supabase via Coolify

Coolify ships Supabase as a one-click service:

1. **New Project** → name it `Goodtime`.
2. **New Resource → Service → Supabase** from the catalog.
3. **Service Domain**: `goodtime.<your-domain>`. Coolify wires Caddy and TLS for you.
4. **Environment Variables**: click "Generate" on each of `JWT_SECRET`, `ANON_KEY`, `SERVICE_ROLE_KEY`, `POSTGRES_PASSWORD`, `DASHBOARD_USERNAME`, `DASHBOARD_PASSWORD`. Save.
5. **Google OAuth** env vars (see § 6).
6. **Deploy**.

Coolify owns the lifecycle from here — restarts, log aggregation in the UI, env var encryption at rest, redeploys on config change.

The Postgres port (`5432`) is **not** exposed externally. The Supabase API gateway on `goodtime.<your-domain>` is the only public surface.

## 6. Google OAuth

In [Google Cloud Console](https://console.cloud.google.com):

1. Create a project (or reuse the one Goodtime Android already uses).
2. Configure OAuth consent screen (External; you're the only test user — add your email under Test users).
3. Create an OAuth 2.0 Client ID, type **Web application**.
4. Authorized redirect URI: `https://goodtime.<your-domain>/auth/v1/callback`.
5. Note client ID + secret → paste into Coolify's Supabase service env vars:
   - `GOTRUE_EXTERNAL_GOOGLE_ENABLED=true`
   - `GOTRUE_EXTERNAL_GOOGLE_CLIENT_ID=<your-id>`
   - `GOTRUE_EXTERNAL_GOOGLE_SECRET=<your-secret>`
   - `GOTRUE_EXTERNAL_GOOGLE_REDIRECT_URI=https://goodtime.<your-domain>/auth/v1/callback`
6. Restart the Supabase service in Coolify.

Mobile (Android) and Mac both use the **web OAuth flow** via Supabase — no native client IDs needed.

## 7. Schema migrations

This repo's `backend/supabase/migrations/` holds versioned SQL files. Two ways to apply them:

**Option A — Coolify Terminal:** open Coolify → Supabase service → Terminal, run `psql ... < migration.sql`. Manual but visible for one-offs.

**Option B — GitHub Actions (preferred for repeatability):** the `deploy-backend.yml` workflow (see `docs/ci-cd.md`) hits a Coolify deploy webhook on tag push. The Coolify service is configured with a "Post-deployment command" that runs `psql -f /migrations/*.sql` against the local Postgres. Migrations are mounted into the container via a Coolify "Persistent Volume" pointing at the repo's `backend/supabase/migrations/`.

## 8. Backups

Two layers:

1. **Hostinger weekly VM snapshots** — automatic, ~1-week retention. Baseline disaster recovery.
2. **Coolify Postgres backups** — configure in Coolify → Supabase service → **Backups** tab:
   - Schedule: daily, 03:00 BRT
   - Target: **Backblaze B2** (S3-compatible). Add B2 application key + bucket name in Coolify Storage settings.
   - Retention: 30 days
   - Coolify runs `pg_dump`, encrypts, uploads, and prunes automatically.

Restore tested quarterly (calendar reminder).

## 9. Monitoring

**Uptime Kuma**, deployed as another Coolify service:

1. Coolify → New Resource → Service → **Uptime Kuma**
2. Service Domain: `status.<your-domain>` (add the DNS A record).
3. Configure monitors:
   - `https://goodtime.<your-domain>/auth/v1/health`
   - `https://goodtime.<your-domain>/rest/v1/`
   - `https://coolify.<your-domain>` (Coolify itself)
4. Add a push notification channel (Telegram, Pushover, ntfy.sh — your call).

Coolify's own dashboard shows CPU / RAM / per-service logs — adequate for daily eyeballing. Uptime Kuma is the alarm bell.

## 10. Operational runbook

| Action | How |
|---|---|
| Coolify admin UI | `https://coolify.<your-domain>` |
| SSH into VPS | `ssh deploy@goodtime.<your-domain>` |
| Restart Supabase | Coolify → Supabase service → Restart |
| Tail Supabase logs | Coolify → Supabase service → Logs |
| Apply new migrations | Push a `v*.*.*-backend` tag (triggers `deploy-backend.yml` → Coolify webhook), or use the Coolify Terminal for one-offs |
| Manual backup now | Coolify → Supabase service → Backups → Run now |
| Restore backup | Coolify → Supabase service → Backups → Restore. Or download dump from B2 and `pg_restore` via Coolify Terminal |
| Update Coolify itself | Coolify → Settings → Update (one click) |
| Update Supabase containers | Coolify → Supabase service → Redeploy with "Pull latest images" toggled |
