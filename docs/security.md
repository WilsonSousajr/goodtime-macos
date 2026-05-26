# Security hardening

Single-user backend, but still public-internet-facing. The hardening below is the minimum that prevents the "drive-by botnet" class of compromise.

## 1. SSH

`/etc/ssh/sshd_config.d/99-hardening.conf`:

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
ChallengeResponseAuthentication no
KbdInteractiveAuthentication no
MaxAuthTries 3
LoginGraceTime 30
AllowUsers deploy
Protocol 2
```

Reload: `systemctl restart ssh`. Test from a second terminal **before** closing the working one.

## 2. Firewall (UFW)

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp           # SSH
ufw allow 80/tcp           # HTTP (Coolify Caddy redirects to HTTPS)
ufw allow 443/tcp          # HTTPS
ufw allow 8000/tcp         # TEMPORARY — Coolify admin UI, ONLY during initial setup
ufw enable
```

After Coolify's admin UI is reachable over `https://coolify.<your-domain>`, close port 8000:

```bash
ufw delete allow 8000/tcp
```

Supabase Postgres (`5432`) and the Kong gateway are bound to the Docker network only by Coolify's defaults — never exposed via UFW.

## 3. fail2ban

`/etc/fail2ban/jail.d/sshd.local`:

```ini
[sshd]
enabled  = true
port     = ssh
filter   = sshd
maxretry = 3
findtime = 600
bantime  = 86400
```

## 4. Automatic security updates

```bash
dpkg-reconfigure -plow unattended-upgrades
```

Enable `Unattended-Upgrade::Automatic-Reboot "true";` and a sane reboot time in `/etc/apt/apt.conf.d/50unattended-upgrades`.

## 5. Supabase secrets (managed by Coolify)

- Generate JWT secret, anon key, service role key, Postgres password via Coolify → Supabase service → Environment Variables → **Generate** button on each.
- Coolify stores secrets encrypted at rest in its own internal database — you don't manage `.env` files manually.
- Service role key never leaves the VPS — clients use anon key only.
- Rotate quarterly: regenerate via Coolify → Supabase service → Environment Variables → Generate new values → Redeploy.

## 6. Postgres

- The `postgres` superuser password lives only inside Coolify's encrypted secret store.
- App-level access is via PostgREST + JWT (issued by GoTrue), never direct DB connection from clients.
- RLS enabled on every public-facing table. Policies for a single user:
  ```sql
  alter table sessions enable row level security;
  create policy "owner-only" on sessions
    using (auth.uid()::text = owner_id::text);
  ```

## 7. Caddy (managed by Coolify)

- Coolify provisions and operates Caddy automatically — you never edit the Caddyfile by hand.
- Auto-TLS via Let's Encrypt; auto-renewal.
- HSTS enabled by Coolify's default Caddy template.
- Per-service access logs viewable in the Coolify UI.

## 8. Docker (managed by Coolify)

- All services run as non-root inside their containers (Supabase images already do).
- The Docker socket is exposed only inside the VPS, to Coolify itself — never over the network.
- Container restart policy `unless-stopped` is set by Coolify defaults, so reboots come back clean.

## 8a. Coolify itself

- Coolify admin login uses your chosen email + password. **Enable 2FA** (Profile → Two-Factor Authentication) — this is the single most important hardening step, since the Coolify UI can spawn containers and read all your secrets.
- Limit Coolify admin access to known IPs via Cloudflare Access or a Tailscale-only access policy if you want belt-and-suspenders. (Out of scope for v1 — 2FA is enough.)

## 9. Mobile + Mac client secrets

- Supabase anon key is shipped with the app (it's safe to expose; RLS does the auth work).
- Service role key never leaves the VPS.
- Google OAuth client ID is shipped; client secret stays in Supabase only.
- OAuth tokens at rest: **Keychain** (Mac), **EncryptedSharedPreferences** (Android).

## 10. Audit & response

- Caddy logs viewable in the Coolify per-service Logs tab.
- Supabase auth events to its built-in `auth.audit_log_entries` table.
- fail2ban writes to `/var/log/fail2ban.log`.
- Uptime Kuma alerts → your phone.
- If you suspect compromise: take a snapshot via the Hostinger control panel → stop affected services from the Coolify UI → investigate from the snapshot. Restore from latest backup if needed.

## 11. What this doesn't protect against

- A malicious Google account login (one-user trust model — if your Google account is compromised, so is this).
- Supply-chain attacks on the Supabase / Caddy / Docker images (pin to specific digests if you want this; baseline does not).
- A 0-day in Postgres / GoTrue.

Mitigations for these are out of scope for a one-user personal project.
