# Deploying Mole on AWS — cost breakdown and Play Store release

> **Step-by-step runbooks live elsewhere** — this doc is the overview + cost/Play-Store reference:
> - **Path A — single EC2 + RDS (start here):** [`aws-ec2-runbook.md`](./aws-ec2-runbook.md)
> - **Path B — managed/split (ECS Fargate + Amplify + S3/CloudFront):** [`aws-ecs-runbook.md`](./aws-ecs-runbook.md)

Mole is a pnpm monorepo of **four deployables** plus a database. Video runs on **LiveKit Cloud**
(or Zoom) — the media server is **fully offloaded**, so nothing on your box carries video traffic.
That removes the old UDP/VM requirement and keeps the compute small.

| Component | What it is | Runtime need |
|---|---|---|
| `backend/` | Express + Prisma, **in-process `node-cron`** | A **long-running** Node process (never serverless-sleeps) |
| `web/` | Next.js 14 App Router (**SSR** — not static export) | A Node server, or a Next-aware host |
| `admin/` | Vite SPA → static `dist/` | Any static host + CDN |
| `shared/` | TS, no build | Bundled into the others |
| Postgres | Prisma datastore | Managed DB or self-run |
| Video | **LiveKit Cloud** / Zoom | External SaaS — nothing to host |

> Key constraints that rule out the "cheap serverless" AWS options:
> - The backend runs **`node-cron` in-process** and holds a **cron lock** — it must be
>   **one always-on instance**, not Lambda and not an autoscaling group that double-fires crons.
> - `web` is **SSR Next.js**, so it needs a Node server (or Amplify/its adapter), not a plain S3 bucket.

**Two sensible AWS shapes:**
- **Path A — Single EC2 + RDS (recommended to start).** One VM runs backend + web + admin behind
  nginx; Postgres is RDS; uploads go to S3. Simplest, cheapest, one thing to operate. Full steps:
  [`aws-ec2-runbook.md`](./aws-ec2-runbook.md).
- **Path B — Managed/split.** RDS + ECS Fargate (backend) + Amplify Hosting (web) + S3/CloudFront
  (admin). More moving parts, scales cleanly, costs more. Full steps:
  [`aws-ecs-runbook.md`](./aws-ecs-runbook.md). Go here only when one box stops being enough.

---

## AWS vs Heroku — which is cheaper?

Short answer: **Heroku is lower-effort to *start*; AWS Path A (EC2+RDS) wins on price once you're
past the smallest tier.** With video on LiveKit Cloud there's no self-hosting constraint pulling you
either way — both hosts work.

### Heroku shape

| Mole part | Heroku |
|---|---|
| backend | 1 **web dyno** (Standard-1X, always-on) |
| web (Next SSR) | 1 more dyno |
| admin (static) | tiny dyno / or push to S3+CloudFront anyway |
| Postgres | **Heroku Postgres** add-on |
| uploads | **must be S3** — Heroku dynos have an ephemeral filesystem (wiped on every deploy/restart), so `STORAGE_PROVIDER=local` loses files. Not optional. |
| cron | **in-process `node-cron` works** on an always-on dyno (don't use Heroku Scheduler — it spins a *new* dyno and would double-run). Never scale the dyno past 1. |

### Rough monthly cost (USD, small production)

| | AWS Path A (1 EC2 + RDS) | Heroku |
|---|---|---|
| Compute | t3.small ~$15 / t3.medium ~$30 | 2× Standard-1X dynos = **$50** |
| Database | db.t4g.micro ~$13 (or micro free-tier yr 1) | Postgres Standard-0 = **$50** (Mini $5 has no HA/backups) |
| Storage/uploads | S3 pennies + 30 GB EBS ~$3 | S3 pennies |
| Bandwidth | first ~100 GB cheap | included-ish |
| TLS/DNS | certbot free + Route 53 $0.50/zone | included |
| **Realistic total** | **~$30–45/mo** | **~$100/mo** |
| Free tier | 12 mo: t3.micro + db.t4g.micro + S3 largely free | none meaningful |

Video is billed separately by LiveKit/Zoom on either host (see the video-cost notes in the
integrations docs).

**Verdict:**
- **Absolute lowest effort, willing to pay for it:** Heroku. `git push`, done.
- **Cheaper long-term, fine with one-time EC2 setup:** **AWS Path A** — roughly half the cost, mirrors
  a standard VPS/PM2 setup. Recommended for Mole.
- **Managed AWS Path B** costs *more* than Path A (ALB ~$16 + Fargate + Amplify + CloudFront) — it
  buys scaling and blue/green, not savings. Not worth it until traffic demands it.

---

## Releasing the Android app to the Play Store

Mobile is Expo (`mobile/app.json`: package **`com.mole.app`**, `versionCode 1`, Firebase via
`google-services.json`). Ship it with **EAS Build** → Play Console.

### One-time setup

1. **Accounts**
   - Google Play Console developer account — **$25 one-time**.
   - Expo account: `npx eas login`.

2. **EAS in the repo**
   ```bash
   cd mobile
   npm i -g eas-cli
   eas build:configure          # creates eas.json with build profiles
   ```
   Ensure the **production** profile builds an **AAB** (Play Store requires Android App Bundle, not
   APK):
   ```json
   // mobile/eas.json
   {
     "build": {
       "production": { "android": { "buildType": "app-bundle" } },
       "preview":    { "android": { "buildType": "apk" } }   // apk for sideload testing
     }
   }
   ```

3. **Signing key** — let EAS generate and store the upload keystore (`eas build` prompts on first
   run and keeps it). **Back it up** (`eas credentials`) — lose it and you can't update the listing
   without a key reset.

4. **Bake the production API URL** — the app reads `EXPO_PUBLIC_API_URL` at build time. Set it in the
   production build profile's `env` (or an `.env` EAS uses) to your real backend:
   ```
   EXPO_PUBLIC_API_URL=https://api.yourdomain.com
   ```
   A wrong/localhost URL here ships a broken app — verify before building.

5. **`google-services.json`** — the Firebase config (FCM push) must be present at
   `mobile/google-services.json` and match package `com.mole.app`. For CI builds add it as an EAS
   **file secret** so it isn't committed.

### Build → test → submit

```bash
cd mobile

# 1. Bump the version for every store upload (Play rejects a duplicate versionCode).
#    Either bump manually in app.json (versionCode 1 -> 2 ...) or set "autoIncrement": true in eas.json.

# 2. Production build (AAB) — runs in EAS cloud, ~15-25 min.
eas build --platform android --profile production

# 3. Grab a testable APK too (optional, installs on a device without the store):
eas build --platform android --profile preview
```

Then either **submit automatically**:

```bash
# needs a Google Play service-account JSON with the "Release" permission,
# referenced as serviceAccountKeyPath in eas.json -> submit.production.android
eas submit --platform android --profile production --latest
```

…or upload the `.aab` by hand in **Play Console → your app → Release**.

### Play Console listing (first submission gates)

Google won't publish until these are filled:
- App name, short + full description, **app icon (512×512)**, **feature graphic (1024×500)**, ≥2
  phone **screenshots**.
- **Privacy Policy URL** (mandatory, live page not a PDF): `https://app.moleedtech.com/privacy-policy` (web app, monorepo #313/#314).
- **Data safety** form — declare what you collect (phone number, location, camera/mic, uploaded KYC
  docs). Be accurate; it's reviewed.
- **Content rating** questionnaire.
- **Target audience** + ads declaration.
- **App access** — Mole is phone-OTP gated, so **provide test login instructions / a test number**
  for the reviewer, or they'll reject for "can't access the app".
- Permissions: `CAMERA`, `RECORD_AUDIO`, location — the reviewer checks each has a justified in-app
  use (video sessions, KYC, nearby-educator matching). Your `app.json` permission strings already
  state these.

### Release tracks (use them in order)

1. **Internal testing** — up to 100 testers by email, live in minutes, no review wait. Start here.
2. **Closed testing** — required proof-of-testing: Google now needs **12 testers for 14 continuous
   days** before a *personal* developer account can go to production. Plan for this delay.
3. **Open testing** — public opt-in.
4. **Production** — full public listing. First production review can take a few days.

### Updates later

```bash
# bump versionCode, then:
eas build --platform android --profile production
eas submit --platform android --profile production --latest
```
- **JS-only change?** Because `app.json` has `updates.enabled: true` with `runtimeVersion` policy
  `appVersion`, you can ship pure-JS fixes over **EAS Update** (OTA) without a new store build —
  but any **native** change (new native module, SDK bump, permission change) needs a fresh
  `eas build` + store upload.

### iOS note

Not asked, but same flow with `--platform ios` needs an **Apple Developer account ($99/yr)** and
`eas submit` to App Store Connect. `app.json` already carries the iOS bundle config.

---

## TL;DR

- **Deploy:** AWS **Path A** = one EC2 (t3.medium + Elastic IP) + RDS Postgres + S3 uploads — full
  steps in [`aws-ec2-runbook.md`](./aws-ec2-runbook.md). Video is LiveKit Cloud (nothing to host).
- **Cheaper:** **AWS (~$30–45/mo) beats Heroku (~$100/mo)** for Mole once past the smallest tier.
  Heroku only wins on zero-setup convenience.
- **Play Store:** EAS Build → **AAB** (bump `versionCode` each time) → `eas submit` → Play Console.
  Budget for the **$25 fee**, the **privacy policy + data-safety** forms, a **test login for the
  reviewer**, and Google's **12-testers-for-14-days** closed-testing gate before production.
