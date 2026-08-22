# Local testing — run the whole stack on your machine

Zero-cost setup for iterating: Postgres in local Docker, backend + web + admin via pnpm, uploads
either on local disk or Cloudflare R2 (free). Crons run in-process (the backend stays up), so you
get full behavior. A 32 GB machine is massively over-spec'd — the stack idles at ~1–2 GB.

## Prerequisites
Node ≥ 20, `pnpm` (`corepack enable`), Docker (for Postgres).

## 1. Postgres (Docker) — you already have this
```bash
docker run -d --name mole-pg --restart unless-stopped \
  -e POSTGRES_USER=mole -e POSTGRES_PASSWORD=mole -e POSTGRES_DB=mole \
  -p 127.0.0.1:5432:5432 -v mole-pgdata:/var/lib/postgresql/data \
  postgres:16
```

## 2. Backend env
```bash
cp backend/.env.example backend/.env
```
Set in `backend/.env`:
- `DATABASE_URL=postgresql://mole:mole@localhost:5432/mole`
- `DIRECT_URL=postgresql://mole:mole@localhost:5432/mole`
- `JWT_ACCESS_SECRET` / `JWT_REFRESH_SECRET` / `TEMP_TOKEN_SECRET` = any non-empty strings
- `STORAGE_PROVIDER=local` (simplest) — or R2, see §5
- `ALLOWED_ORIGINS=` (blank = allow all; fine for local)

## 3. Install, migrate, seed
```bash
pnpm install
pnpm --filter @mole/backend exec prisma generate
pnpm --filter @mole/backend exec prisma migrate deploy
pnpm --filter @mole/backend db:seed        # platform config defaults
pnpm --filter @mole/backend seed:admin      # first admin login
```

## 4. Run everything
```bash
pnpm dev        # backend :3000, web :3001, admin :3002 (concurrently)
pnpm mobile     # Expo, separate terminal
```
- `web/.env.local`: `NEXT_PUBLIC_API_URL=http://localhost:3000`, `NEXT_PUBLIC_VIDEO_PROVIDER=zoom` (or `livekit`).
- **admin** in dev needs no API URL — its Vite dev server proxies `/api` → `:3000` (see `admin/vite.config.ts`), so the relative `/api/v1` base works. (Only the *production* admin build needs an absolute `VITE_API_URL`.)

Log in to admin at http://localhost:3002 with the seeded admin; open web at http://localhost:3001.

## 5. Storage options
- **`local` (default):** uploads written under `backend/uploads`. Zero external setup — best for testing.
- **Cloudflare R2 (free, S3-compatible):** in `backend/.env`:
  ```
  STORAGE_PROVIDER=s3
  AWS_REGION=auto
  AWS_S3_BUCKET=mole-uploads
  AWS_S3_ENDPOINT=https://<accountid>.r2.cloudflarestorage.com
  AWS_ACCESS_KEY_ID=<from R2 "S3 API token">
  AWS_SECRET_ACCESS_KEY=<from R2 "S3 API token">
  ```
  (Cloudflare → R2 → create bucket → *Manage R2 API Tokens* → create token → use the S3 credentials + endpoint. Presigned upload/download URLs work via the S3 SDK.)

## 6. Testing the mobile app on a real phone
Expo on a device can't reach `localhost` on your machine. Either:
- Point the app's API base at your machine's **LAN IP** (`API_BASE_URL=http://<lan-ip>:3000`) with the phone on the **same Wi-Fi**, and allow port 3000 through your firewall; or
- Expose the backend over HTTPS with a tunnel: `cloudflared tunnel --url http://localhost:3000`.

## 7. Webhooks (payment / Zoom)
External webhooks can't reach `localhost`. For payments use the built-in **`POST /payment/:id/mock-confirm`** (dev only) to complete a booking without a real UPI callback. For real webhook testing, use a `cloudflared`/`ngrok` tunnel and point the provider at the tunnel URL.

## When to move off local
Local is ideal for development/testing. Go to a cloud host only when you need a **public** URL for
real users — see `aws-deploy.md` (AWS free tier) or `free-deploy.md` (Cloudflare Pages + a free Node host).
