# Deploying Mole for free (Cloudflare + Supabase + a free Node host)

Practical guide to host the whole stack on free tiers. Repo is a pnpm monorepo:
`backend` (Express + Prisma), `web` (Next.js 14), `admin` (Vite SPA), `shared` (`@mole/shared`).

## What goes where — and the one caveat

| Component | Free host | Notes |
|---|---|---|
| **Postgres** | **Supabase** (free) | 500 MB, pooled connections. |
| **File uploads** | **Supabase Storage** or **Cloudflare R2** (free) | Both are S3-compatible; the backend's `S3StorageProvider` works with either. |
| **admin** (Vite SPA) | **Cloudflare Pages** (free) | Static build — ideal fit. |
| **web** (Next.js 14) | **Cloudflare Pages** (`next-on-pages`) **or Vercel Hobby** (free) | Vercel is the least-friction option for Next.js. |
| **backend** (Express) | **Render / Fly.io / Koyeb** (free) | ⚠️ NOT Cloudflare Workers. |

> **Why not the backend on Cloudflare?** It's a long-running Express server with **7 in-process `node-cron` jobs**, `multer` uploads, and webhooks. Cloudflare Workers are stateless edge functions with no persistent process and no in-process cron — running this there means a full rewrite (Hono + Cloudflare Cron Triggers + R2). Not worth it. Use a free Node host instead.
>
> **Free-tier cron caveat:** free Node hosts **sleep on idle** (Render sleeps after 15 min; wakes on the next HTTP request). While asleep, the in-process cron jobs (no-show detection, auto-complete, reminders, wallet expiry) **do not fire**. Fine for demo/testing. For reliable schedules either (a) keep the service awake with a free external pinger like cron-job.org hitting a health URL every 10 min, (b) use an always-on host (Fly.io / Koyeb single instance), or (c) move scheduled work to Supabase `pg_cron` / an external scheduler that calls backend endpoints.

---

## 0. Prerequisites
- Accounts (all free): **GitHub** (repo already at `rrahulraja/mole`), **Supabase**, **Cloudflare**, and a Node host (**Render** recommended to start).
- Local: Node ≥ 20, `pnpm` (via `corepack enable`).
- Decide two public URLs up front (you'll set them after the first deploy):
  - `API_URL` = backend URL (e.g. `https://mole-api.onrender.com`)
  - web URL + admin URL (Cloudflare Pages, e.g. `https://mole-web.pages.dev`, `https://mole-admin.pages.dev`)

---

## 1. Postgres — Supabase

1. Supabase → **New project**. Pick a region near your users. Save the **database password** (keep it OUT of git — see §9).
2. Project → **Settings → Database → Connection string**. You need **two** URLs:
   - **Transaction pooler** (host `...pooler.supabase.com`, port **6543**) → runtime `DATABASE_URL`. Append `?pgbouncer=true&connection_limit=1`.
   - **Direct/session** (port **5432**) → `DIRECT_URL` (used for migrations; PgBouncer can't run them).
3. **Prisma change** — add `directUrl` to `backend/prisma/schema.prisma`:
   ```prisma
   datasource db {
     provider  = "postgresql"
     url       = env("DATABASE_URL")   // pooled (6543) for runtime
     directUrl = env("DIRECT_URL")     // direct (5432) for migrations
   }
   ```
4. Run migrations + seed once (locally, with both env vars exported, or as the host's release command):
   ```bash
   export DATABASE_URL="postgresql://...:6543/postgres?pgbouncer=true&connection_limit=1"
   export DIRECT_URL="postgresql://...:5432/postgres"
   pnpm --filter @mole/backend exec prisma migrate deploy
   pnpm --filter @mole/backend exec prisma generate
   pnpm --filter @mole/backend db:seed          # platform config defaults
   pnpm --filter @mole/backend seed:admin        # first admin user
   ```

---

## 2. File storage (uploads) — free, S3-compatible

Pick one; both expose an S3 API the existing `S3StorageProvider` can use.

- **Supabase Storage:** Storage → create a bucket → **Settings → Storage → S3 connection** gives an endpoint + access key/secret. Set `STORAGE_PROVIDER=s3` and the `S3_*` env vars to the Supabase endpoint/bucket/region/keys.
- **Cloudflare R2** (10 GB free): R2 → create bucket → create an **S3 API token**. Endpoint is `https://<accountid>.r2.cloudflarestorage.com`. Set the same `S3_*` vars to it.

(Confirm the exact `S3_*` var names in `backend/src/lib/config.ts` and set them in the backend host's env.)

---

## 3. Backend — Render (free, persistent Node)

**Render → New → Web Service → connect the GitHub repo.** Because it's a pnpm monorepo that depends on `@mole/shared`, install/build from the repo root with filters.

- **Root directory:** (leave repo root)
- **Build command:**
  ```bash
  corepack enable && pnpm install --frozen-lockfile && pnpm --filter @mole/backend exec prisma generate && pnpm --filter @mole/backend build
  ```
- **Start command:**
  ```bash
  pnpm --filter @mole/backend start
  ```
  (`start` = `node dist/index.js`.)
- **Pre-deploy / release command** (Render "Pre-Deploy Command"):
  ```bash
  pnpm --filter @mole/backend exec prisma migrate deploy
  ```
- **Health check path:** an existing GET route (e.g. `/api/v1/config/onboarding`, which is public).
- **Instance type:** Free.

**Env vars** (see §7 for the list) — set `DATABASE_URL`, `DIRECT_URL`, JWT secrets, `ALLOWED_ORIGINS`, storage `S3_*`, `NODE_ENV=production`, `PORT` (Render sets `PORT` automatically — make sure `src/index.ts` reads `process.env.PORT`).

**Or use Docker** (portable across Render/Fly/Koyeb) — `backend/Dockerfile`:
```dockerfile
# ---- build ----
FROM node:20-slim AS build
RUN corepack enable
WORKDIR /repo
COPY pnpm-workspace.yaml package.json pnpm-lock.yaml .npmrc ./
COPY shared ./shared
COPY backend ./backend
RUN pnpm install --frozen-lockfile
RUN pnpm --filter @mole/backend exec prisma generate
RUN pnpm --filter @mole/backend build
# ---- run ----
FROM node:20-slim
RUN corepack enable && apt-get update && apt-get install -y openssl && rm -rf /var/lib/apt/lists/*
WORKDIR /repo
COPY --from=build /repo /repo
ENV NODE_ENV=production
WORKDIR /repo/backend
CMD ["node", "dist/index.js"]
```
Run migrations as a one-off/release step (`prisma migrate deploy`) before the first boot.

**Free alternatives to Render:** **Fly.io** (needs a card; small always-on allowance — best for keeping crons alive), **Koyeb** (one always-on free service). Same build/start/Docker applies.

---

## 4. admin (Vite SPA) — Cloudflare Pages

**One required code change:** the admin API client uses a *relative* base (`baseURL: "/api/v1"` in `admin/src/api/axois.ts`), which only works when same-origin. On Pages it must call the backend's absolute URL. Change it to read an env var:
```ts
const base = import.meta.env.VITE_API_URL ?? '';
export const api = axios.create({ baseURL: `${base}/api/v1` });
```

Then in **Cloudflare → Pages → Create → connect repo**:
- **Framework preset:** None
- **Build command:** `corepack enable && pnpm install && pnpm --filter @mole/admin build`
- **Build output directory:** `admin/dist`
- **Root directory:** (repo root)
- **Env var:** `VITE_API_URL = https://<your-backend-url>`
- **SPA routing:** add `admin/public/_redirects` containing:
  ```
  /*  /index.html  200
  ```

---

## 5. web (Next.js 14)

**Easiest — Vercel Hobby (free):** import the repo, set **Root Directory = `web`**, framework auto-detects Next.js. Add env `NEXT_PUBLIC_API_URL` and `NEXT_PUBLIC_VIDEO_PROVIDER`. Deploy. (Vercel handles the monorepo + Next build with zero config.)

**Or Cloudflare Pages** with `@cloudflare/next-on-pages`:
- **Build command:** `corepack enable && pnpm install && pnpm --filter @mole/web exec next-on-pages`
- **Build output directory:** `web/.vercel/output/static`
- **Compatibility flags:** add `nodejs_compat`; set a recent compatibility date.
- **Env:** `NEXT_PUBLIC_API_URL`, `NEXT_PUBLIC_VIDEO_PROVIDER`.
- Caveat: App-Router pages run on the edge runtime under next-on-pages; if a page uses an unsupported Node API the build/route fails — Vercel avoids this. Prefer Vercel unless you specifically want everything on Cloudflare.

---

## 6. Wire them together (CORS + URLs)

1. Set the backend's **`ALLOWED_ORIGINS`** to your web + admin URLs, comma-separated:
   `ALLOWED_ORIGINS=https://mole-web.pages.dev,https://mole-admin.pages.dev`
   (`backend/src/app.ts` reads this; without it CORS is `*`.)
2. Set **web** `NEXT_PUBLIC_API_URL` and **admin** `VITE_API_URL` to the backend URL, then redeploy those two.
3. (Optional) Custom domains: add them in Cloudflare Pages + your host, then update `ALLOWED_ORIGINS` and the two frontend env vars accordingly.

---

## 7. Backend env var reference (set on the Node host)

Essential (confirm names/extras in `backend/src/lib/config.ts`):
- `NODE_ENV=production`, `PORT` (host-provided)
- `DATABASE_URL` (pooled 6543), `DIRECT_URL` (direct 5432)
- `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET`, `TEMP_TOKEN_SECRET`
- `ALLOWED_ORIGINS`
- Storage: `STORAGE_PROVIDER=s3`, `S3_ENDPOINT`, `S3_REGION`, `S3_BUCKET`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`, (optional `CDN_BASE_URL`)
- Video: `VIDEO_PROVIDER` (`zoom`|`livekit`); LiveKit `LIVEKIT_URL`/`LIVEKIT_API_KEY`/`LIVEKIT_API_SECRET`, or Zoom `ZOOM_SDK_KEY`/`ZOOM_SDK_SECRET`/`ZOOM_ACCOUNT_ID`/`ZOOM_WEBHOOK_SECRET`
- Payments/OTP/notifications as used: UPI/payment webhook secret, Twilio (OTP + voice + WhatsApp `TWILIO_WHATSAPP_FROM`), FCM/Firebase.

Frontends:
- web: `NEXT_PUBLIC_API_URL`, `NEXT_PUBLIC_VIDEO_PROVIDER`
- admin: `VITE_API_URL`

---

## 8. Deploy order + smoke test
1. Supabase project + migrations + seed (§1).
2. Storage bucket (§2).
3. Backend on Render with all env vars (§3) → note its URL.
4. admin + web on Pages/Vercel with the backend URL (§4, §5).
5. Set `ALLOWED_ORIGINS` on the backend to the two frontend URLs; redeploy backend.
6. Smoke: open admin → login (seeded admin); open web → phone/OTP → onboarding; hit a public backend route in the browser to confirm it's up.

---

## 9. Secret hygiene (do this now)
`supabase-pass` (your DB password) and any `.env` are **untracked but NOT git-ignored** — one `git add -A` from being committed to a public-ish repo. Add them to `.gitignore`:
```
supabase-pass
.env
.env.*
!.env.example
```
Never commit real secrets; set them only in each host's dashboard env settings.
