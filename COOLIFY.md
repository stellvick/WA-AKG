# Deploying WA-AKG on Coolify (Git + Docker Compose)

This repo deploys as a Coolify **Application** with the build pack **Docker Compose**.
The build pack is chosen at creation and cannot be switched later.

## One-time setup

1. **New Resource → Application → connect this GitHub repo** (branch `main`).
2. **Build pack = Docker Compose.** Compose location: `docker-compose.yml` (repo root).
3. **Environment Variables** — Coolify creates these from the `${VAR}` names in the
   compose. Set the required ones (the deploy is blocked until they exist):
   | Variable | Notes |
   |---|---|
   | `AUTH_SECRET` | required — `openssl rand -base64 32` |
   | `MYSQL_ROOT_PASSWORD` | required — internal DB password |
   | `ADMIN_EMAIL` / `ADMIN_PASSWORD` | first-run SuperAdmin (change the default) |
   | `NEXT_PUBLIC_*` | public branding/swagger flags — **Build Variables** (inlined by Next at build; rebuild after changing) |
   | `MYSQL_DATABASE` | defaults to `wa_akg` |
4. **Domain** — Configuration → **Domains for `app`**: `https://<your-host>:3000`.
   The `3000` is the **container** port, not a host publish. Coolify's proxy issues the
   certificate. (Alternatively declare `SERVICE_FQDN_APP_3000: <your-host>` in the compose.)
5. **Deploy.**

## What the Compose file must not do (Coolify rules)

- No top-level `version:` key.
- No published `ports:` — Coolify's proxy routes the domain; a host port would bypass the
  proxy and block the rolling-update path.
- No hard-coded secrets — use `${VAR}` so they become editable Coolify variables.
- Do not rely on the **Healthcheck** UI page for a Compose resource; the `healthcheck:`
  blocks in the compose file are authoritative.

## Runtime behaviour

The image entrypoint runs `prisma db push` (syncs the MySQL schema), optionally creates
the first SuperAdmin, then starts the custom Next.js + Socket.IO server on port 3000.

Compose resources deploy **stop-the-world** (`docker compose up`) — there is no
zero-downtime replacement, so expect a short restart window on each deploy.

`data/` and `uploads/` (WhatsApp session store + media) are **named volumes**, server-local
and **not a backup** — back them up separately.

## Notes / caveats

- The server sends a heartbeat to `api-wa-akg.aikeigroup.net` every 30s (upstream
  telemetry, includes your `BASE_URL`). Remove or gate it in
  `src/server/index.ts` if you do not want that.
- The Baileys library is patched via `patches/@whiskeysockets+baileys+7.0.0-rc.9.patch`
  (applied by `patch-package` at install time) — keep `patches/` in the build context.
