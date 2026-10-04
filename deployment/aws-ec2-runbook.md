# Mole on AWS — EC2 go-live runbook (Path A)

One EC2 box runs **backend + web + admin behind nginx**; Postgres is **on the box (Docker) now,
RDS later**; **S3** holds uploads + recordings; **CloudFront** optionally fronts public assets. Video
is **LiveKit Cloud** (or Zoom) — SaaS, nothing to host. This is the simplest production shape and the
one to launch on.

> **What we're using right now (demo + early low traffic):**
> - **1× EC2 `t3.medium`** (2 vCPU / 4 GB) in **ap-south-1**, + 2 GB swap, + Elastic IP.
> - **Postgres in Docker on the same box** ([`deploy/postgres/`](../../deploy/postgres/)) — cron jobs
>   poll localhost (zero network cost), one bill, simple. **Daily `pg_dump` → S3** (Step 3A.1).
> - Video on **LiveKit Cloud** (free tier for the demo) — no media server on the box.
> - **Cost ≈ a few dollars** for the demo (t3.medium ~$0.045/hr; stop it between demo days).
> - **Move to RDS (Step 3B) when traffic grows** — `pg_dump` → restore + repoint `DATABASE_URL`.

**End state**
```
                  GoDaddy DNS
                        │
        ┌───────────────┴───────────────┐
   api./app./admin.yourdomain.com   (CloudFront → S3 public assets, optional)
        │
   ┌────▼───────────────────────────────┐        ┌─────────────┐
   │  EC2 t3.medium (Ubuntu 22.04)      │        │  S3 bucket  │
   │  nginx :80/:443                    │◄──────►│ uploads +   │
   │   ├─ backend  127.0.0.1:3000       │        │ recordings/ │
   │   ├─ web      127.0.0.1:3001       │        │ (private)   │
   │   ├─ admin    127.0.0.1:3002       │        └─────────────┘
   │   └─ postgres 127.0.0.1:5432 (docker) ──┐   daily pg_dump → S3
   └──────────────────────────────────────┘ │        ▲
                        ▲                    └────────┘
             LiveKit Cloud (or Zoom) — video SaaS, nothing to host
             (Zoom recording webhook → S3, if using Zoom)
```
> Postgres is on-box (Docker) now; at Step 3B it moves to a same-AZ **RDS** and the
> arrow above points there instead.

**Confirmed from the codebase (don't second-guess these):**
- Prisma datasource uses **only `DATABASE_URL`** — no `DIRECT_URL` on RDS.
- Ports are fixed: backend **3000**, web **3001**, admin **3002**.
- Web reads `NEXT_PUBLIC_API_URL` = **origin only** (app appends `/api/v1`); it's **compile-time**.
- Admin is a Vite SPA; axios uses relative `/api/v1`; build with `--base=/admin/` for a subpath.
- Storage uses **`AWS_*`** env names; `STORAGE_PROVIDER=s3` + `AWS_S3_BUCKET` activates S3.
- Zoom recording webhook: `POST /api/v1/recordings/zoom-webhook` (HMAC-verified) → downloads to
  `s3://<bucket>/recordings/{bookingId}/*.mp4`. Playback is a **backend presigned URL**
  (`GET /api/v1/recordings/:bookingId`) so the **bucket stays private**.
- Recordings carry a 24h access `expiresAt` **but nothing deletes the S3 object** → you **must** add
  an S3 lifecycle expiry rule (Step 4.4) or storage grows forever.

**Region:** examples use **ap-south-1 (Mumbai)**. Swap if you prefer. CloudFront/ACM certs for
CloudFront **must** be created in **us-east-1** — noted where relevant.

---

## Billing: Free Tier now → Activate credits later

You do **not** need the company or domain to launch. Create the AWS account today on a personal
email + card and deploy; apply for startup credits once the company exists.

- **12-month Free Tier** starts the moment the account is created — covers a **t3.micro** EC2,
  **db.t4g.micro** RDS, and **5 GB S3** largely free for the first year. Fine for testing/soft-launch.
  (For real ~500 sessions/day you'll size up to t3.large + db.t4g.small, which is beyond Free Tier and
  bills normally — see Steps 3 and 5.)
- **AWS Activate credits apply forward, not backward** — they offset bills **after** approval, never
  past spend. So keep spend low (Free Tier + small instances) until credits land, or apply early.
- **Apply only after** you have a **company + `@company.com` email + website** — the Activate form
  rejects gmail. The AWS account's root email and the application email don't have to match; optionally
  change the root email to the company domain (Account settings) before applying, for clean ownership.
- **One account, apply once.** Don't spin up multiple accounts to stack credits — Activate is
  once-per-startup and abuse gets flagged.

**India (Activate credit amounts — USD credits, same globally, verify current figures):**
| Tier | Credit | Path |
|---|---|---|
| **Activate Founders** | **~$1,000** (valid 2 yrs) | Bootstrapped, apply directly at `aws.amazon.com/activate`. No provider needed. |
| **Activate Portfolio** | **$5,000 → $100,000** | Requires an **approved provider association** (accelerator/VC/incubator — e.g. a program your startup joins). Bigger tier = bigger credit. |

Plus AWS **Business Support credits** + training. Credits are **USD-denominated** — the "India region"
doesn't change the *amount*; it only changes which region you spend them in (use `ap-south-1`).
Note the **bigger lever for Mole is Zoom's own startup credits**, since Zoom Video minutes outspend the
AWS bill — apply to **Zoom for Startups / ISV** too.

> Exact Activate amounts + eligibility shift periodically; treat the table as recent-known, not live.
> Confirm on the Activate page before applying.

---

## Step 0 — Before you touch the console

Have these ready:
- A **domain** (ours is on **GoDaddy** — DNS setup in Step 5.3).
- Provider credentials in hand: **Twilio** (SID/token/number), **Razorpay** (key/secret/webhook
  secret), **Zoom Video SDK** (Step 8), **WhatsApp** token/phone-id/verify-token, **FCM** service
  account JSON.
- The repo pushed to GitHub (private is fine — you'll deploy-key or clone with a token).
- Decide the three hostnames:
  - `api.yourdomain.com` → backend
  - `app.yourdomain.com` → web
  - `admin.yourdomain.com` → admin

Generate secrets now (run locally, paste into `.env` later):
```bash
openssl rand -hex 32   # JWT_ACCESS_SECRET
openssl rand -hex 32   # JWT_REFRESH_SECRET
openssl rand -hex 32   # TEMP_TOKEN_SECRET
openssl rand -hex 32   # UPI_WEBHOOK_SECRET
```

---

## Step 1 — IAM: an admin user + a deploy user (don't use root)

1. AWS Console → **IAM** → **Users** → **Create user** `mole-admin`.
2. Attach **AdministratorAccess** (you, for setup only). Enable MFA.
3. Create a **least-privilege app user** the app will use for S3:
   - IAM → **Users** → `mole-app-s3` → **no console access**, programmatic only.
   - Create access key → **save the key id + secret** (used in `.env` as `AWS_ACCESS_KEY_ID` /
     `AWS_SECRET_ACCESS_KEY`). You'll attach an S3 policy in Step 4.3.
   > Better long-term: skip the static keys and give the **EC2 instance an IAM role** (Step 5.6).
   > The runbook shows keys because they're the fastest path to launch; swap to a role after.

---

## Step 2 — Networking: security groups

Console → **VPC** → use the **default VPC** (fine for launch). Create two security groups:

**`mole-app-sg`** (for EC2) — inbound:
| Type | Port | Source | Why |
|---|---|---|---|
| SSH | 22 | **My IP** | admin only |
| HTTP | 80 | 0.0.0.0/0 | nginx + certbot |
| HTTPS | 443 | 0.0.0.0/0 | nginx |

No 3000/3001/3002 — those bind to `127.0.0.1`; nginx is the only public door.

**`mole-rds-sg`** (for RDS) — inbound:
| Type | Port | Source |
|---|---|---|
| PostgreSQL | 5432 | **`mole-app-sg`** (the SG, not an IP) |

RDS is reachable **only** from the app box. Never make it public.

---

## Step 3 — Postgres

Two options. **Right now (demo + early low traffic) we run Step 3A** — Postgres in
Docker on the same EC2 box. Move to **Step 3B (RDS)** once you have real users.
Migration is a `pg_dump` → restore into RDS + a `DATABASE_URL` change — no code
change (Prisma doesn't care where Postgres lives).

Why on-box first: no second billed service, cron jobs poll **localhost** (zero
network cost), and it's one thing to operate for a throwaway demo. The trade-off is
a single point of failure + a ≤24h loss window between backups — acceptable for a
demo, not for revenue. That's exactly when you switch to 3B.

### Step 3A — Docker Postgres on the box (use now)

Files live in [`deploy/postgres/`](../../deploy/postgres/) (compose + backup script + README).
Docker is installed in Step 5.4; then:

```bash
cd /opt/mole
export POSTGRES_PASSWORD='a-strong-password'          # save it somewhere safe
docker compose -f deploy/postgres/docker-compose.yml up -d
```
Connection string (Postgres is bound to `127.0.0.1:5432` — never exposed):
```
DATABASE_URL=postgresql://mole:a-strong-password@localhost:5432/mole
```
The compose file uses a **named volume** (`mole-pgdata`, survives container
recreate) and `restart: unless-stopped`. Also set the EC2 root volume to
**`DeleteOnTermination=false`** so terminating the instance doesn't nuke the data.

**Daily backup → S3 (do this before the demo, not after a loss).** See Step 3A.1.

### Step 3A.1 — Daily backup to S3 (required)

A backup on the same EBS volume dies with the volume — it **must** go off-box.
1. Create a private bucket, e.g. `mole-db-backups`, **enable versioning**, and add a
   **lifecycle rule** to expire objects after ~30 days.
2. Give the instance role (Step 5.6) `s3:PutObject` on `mole-db-backups/*`.
3. Schedule the backup script (`deploy/postgres/backup-db.sh` — `pg_dump | gzip |
   aws s3 cp`, then prunes local copies):
   ```bash
   chmod +x /opt/mole/deploy/postgres/backup-db.sh
   crontab -e
   # daily 03:00 UTC:
   0 3 * * * BACKUP_S3_BUCKET=mole-db-backups /opt/mole/deploy/postgres/backup-db.sh >> /var/log/mole-backup.log 2>&1
   ```
4. **Test a restore once** (untested backup = no backup):
   ```bash
   aws s3 cp s3://mole-db-backups/postgres/mole-<ts>.sql.gz .
   gunzip -c mole-<ts>.sql.gz | docker exec -i mole-postgres psql -U mole -d mole
   ```
5. Optional belt-and-suspenders: also take periodic **EBS snapshots** of the volume.

### Step 3B — RDS Postgres (switch to this when traffic grows)

Gives you point-in-time recovery, automated failover, and managed backups — worth
it once you have real users. Console → **RDS** → **Create database**:
1. **Standard create**, engine **PostgreSQL 16**.
2. Template **Production** (or Dev/Test to save cost while testing).
3. Instance class **db.t4g.small** (2 GB — right for ~500 sessions/day; `micro` only for testing).
4. Storage: **20 GB gp3**, enable **storage autoscaling** (cap ~100 GB).
5. **Credentials:** master username `mole`, set a strong password → save it.
6. Connectivity:
   - VPC: default. **Public access: No.**
   - VPC security group: **`mole-rds-sg`** (remove the default).
   - **AZ: the same AZ as the EC2 box** — same-AZ private traffic is free; cross-AZ is $0.01/GB.
7. Additional config: initial database name **`mole`**. Enable **automated backups** (7 days).
8. Create. Wait ~5–10 min. Copy the **endpoint** (e.g. `mole.abc123.ap-south-1.rds.amazonaws.com`).

Migrate the data across, then repoint the env:
```bash
# dump from the on-box container, restore into RDS
docker exec mole-postgres pg_dump -U mole -d mole --clean --if-exists \
  | psql "postgresql://mole:PASSWORD@ENDPOINT:5432/mole"
```
```
DATABASE_URL=postgresql://mole:PASSWORD@ENDPOINT:5432/mole
```
Then `pm2 restart mole-backend`, stop the container, and drop the backup cron's
`mole-postgres` dependency (RDS has its own automated backups).

---

## Step 4 — S3 bucket (uploads + recordings)

One bucket, private. Uploads and recordings share it (recordings live under `recordings/`).

### 4.1 Create the bucket
Console → **S3** → **Create bucket**:
- Name: `mole-media-prod` (globally unique — pick your own).
- Region: **ap-south-1** (same as everything else).
- **Block all public access: ON** (leave it on — assets are served via presigned URLs / CloudFront OAC).
- Default encryption: **SSE-S3** (on by default).
- Create.

### 4.2 CORS (so the web/mobile clients can load presigned media)
Bucket → **Permissions** → **CORS** → paste:
```json
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["GET", "PUT", "HEAD"],
    "AllowedOrigins": [
      "https://app.yourdomain.com",
      "https://admin.yourdomain.com"
    ],
    "ExposeHeaders": ["ETag"],
    "MaxAgeSeconds": 3000
  }
]
```

### 4.3 IAM policy for the app user
IAM → `mole-app-s3` → **Add permissions** → **inline policy** → JSON:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:GetObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::mole-media-prod/*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::mole-media-prod"
    }
  ]
}
```

### 4.4 Lifecycle rule — REQUIRED for recordings
Nothing in the app deletes recording objects; only access expires (24h). Reclaim the storage:

Bucket → **Management** → **Lifecycle rules** → **Create rule**:
- Name `expire-recordings`.
- Scope: **Prefix** `recordings/`.
- Action: **Expire current versions** after **2 days** (keep a small grace over the 24h access window).
- Create.

> Want to *keep* recordings but cheaply? Instead of expire, add a **transition to Glacier Deep
> Archive after 7 days**. But the app treats recordings as 24h-ephemeral, so **expire** matches the
> product behaviour and keeps S3 near-free.

---

## Step 5 — EC2 instance

### 5.1 Launch
Console → **EC2** → **Launch instance**:
- Name `mole-app`.
- AMI **Ubuntu Server 22.04 LTS**.
- Type: **t3.medium (2 vCPU / 4 GB) — what we're using now** for the demo + early
  low traffic. Video runs on LiveKit Cloud, so no media server sits on this box;
  4 GB comfortably runs backend + web (Next.js) + admin + the on-box Postgres, with
  the 2 GB swap in Step 5.4 covering the `next build` RAM spike. Scale to **t3.large
  (8 GB)** when you approach ~500 sessions/day.
- Key pair: create/download `mole-key.pem` (`chmod 400 mole-key.pem`).
- Network: default VPC, **security group `mole-app-sg`**.
- Storage: **30 GB gp3**. Set the root volume's **`Delete on termination` to No** so a
  stop/terminate can't wipe the on-box database (Step 3A).
- Advanced → enable **T3 Unlimited** (avoids CPU-credit throttling under sustained load).
- Launch. **Stop the instance between demo days** to stop compute charges (the Elastic
  IP + EBS persist).

### 5.2 Elastic IP (mandatory)
EC2 → **Elastic IPs** → **Allocate** → **Associate** with `mole-app`.
> Without it the public IP changes on stop/start and breaks DNS + the API URLs baked into web/mobile.

### 5.3 DNS — GoDaddy (our registrar)

The domain is on **GoDaddy**, so manage DNS there directly — no Route 53 needed for
Path A. Point three subdomains at the Elastic IP.

1. GoDaddy → **My Products** → your domain → **DNS** (Manage DNS).
2. Under **Records → Add**, create three **A** records (GoDaddy auto-appends the
   domain, so the **Name** is just the subdomain label, not the full host):

   | Type | Name | Value | TTL |
   |---|---|---|---|
   | A | `api` | `<Elastic IP>` | 600 |
   | A | `app` | `<Elastic IP>` | 600 |
   | A | `admin` | `<Elastic IP>` | 600 |

3. **Delete/adjust GoDaddy's defaults that collide:** the parking `A @ → Parked`
   record and the `CNAME www → @`. Leave the `NS`/`SOA` records alone. If you want
   the apex (`yourdomain.com`) to serve too, add `A @ → <Elastic IP>`.
4. **TTL 600** (10 min) during setup so mistakes propagate out fast — raise it to
   1 hr once stable.
5. Propagation is usually minutes (can be up to ~1 hr). Verify before certbot:
   ```bash
   dig +short api.yourdomain.com    # must return the Elastic IP
   ```

> TLS: certbot uses **HTTP-01** validation (Step 7.3), which only needs these A
> records resolving + port 80 open — **no extra DNS/TXT records** on GoDaddy.

> Prefer AWS-native DNS (smoother for CloudFront/ACM later)? Alternative: create a
> **Route 53 hosted zone** for the domain, then in GoDaddy → **Nameservers → Change**
> replace GoDaddy's NS with the **four Route 53 nameservers**. Adds ~$0.50/mo; not
> needed for the demo.

### 5.4 SSH in + prep the box
```bash
ssh -i mole-key.pem ubuntu@<Elastic IP>

sudo apt update && sudo apt upgrade -y
sudo apt install -y git nginx

# Node 20 + pnpm + PM2
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm i -g pnpm@10 pm2

# Docker + compose plugin (for the on-box Postgres, Step 3A)
sudo apt install -y docker.io docker-compose-v2
sudo usermod -aG docker ubuntu   # log out/in (or `newgrp docker`) to use docker without sudo

# Swap (required on 4 GB / t3.medium — stops the next build OOM)
sudo fallocate -l 2G /swapfile && sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### 5.5 Get the code
```bash
cd /opt && sudo git clone https://github.com/rrahulraja/mole.git
sudo chown -R ubuntu:ubuntu /opt/mole
cd /opt/mole && git checkout main && pnpm install
```
> Private repo? Use a **deploy key** or `git clone https://<TOKEN>@github.com/...`.

### 5.6 (Recommended) Instance role instead of static keys
EC2 → instance → **Actions → Security → Modify IAM role** → attach a role with the same S3 policy as
Step 4.3. Then **leave `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` empty or absent** in `.env` —
`services/storage/s3Client` passes `credentials` only when BOTH keys are set, otherwise it omits them
so the SDK falls back to its default provider chain (the instance role via IMDS). `AWS_REGION` is still
required. Restart with `pm2 restart mole-backend` after attaching the role. Do this after first launch
if you want to move fast now.

---

## Step 6 — Configure + build

### 6.1 Backend env
```bash
cd /opt/mole/backend
cp .env.example .env
nano .env
```
Fill (only the live-relevant keys shown):
```dotenv
NODE_ENV=production
PORT=3000

# Database — DATABASE_URL only; schema does not use DIRECT_URL.
# On-box Docker Postgres now (Step 3A); swap for the RDS endpoint at Step 3B.
DATABASE_URL=postgresql://mole:PASSWORD@localhost:5432/mole

# Auth secrets (from Step 0)
JWT_ACCESS_SECRET=...
JWT_REFRESH_SECRET=...
TEMP_TOKEN_SECRET=...
UPI_WEBHOOK_SECRET=...

# Public URLs
API_BASE_URL=https://api.yourdomain.com
ADMIN_APP_URL=https://admin.yourdomain.com
CDN_BASE_URL=                     # set to CloudFront domain after Step 9, else leave blank

# Storage → S3 (one bucket; recordings go under recordings/)
STORAGE_PROVIDER=s3
AWS_REGION=ap-south-1
AWS_S3_BUCKET=mole-media-prod
AWS_S3_KEY_PREFIX=                 # optional namespacing, leave blank
AWS_S3_ENDPOINT=                  # BLANK for real AWS S3 (only set for R2/MinIO)
AWS_ACCESS_KEY_ID=...             # omit if using the instance role (Step 5.6)
AWS_SECRET_ACCESS_KEY=...

# SMS/OTP → Twilio
SMS_PROVIDER=twilio
TWILIO_ACCOUNT_SID=...
TWILIO_AUTH_TOKEN=...
TWILIO_FROM_NUMBER=+1...

# Payments → Razorpay
PAYMENT_PROVIDER=razorpay
RAZORPAY_KEY_ID=...
RAZORPAY_KEY_SECRET=...
RAZORPAY_WEBHOOK_SECRET=...
MERCHANT_VPA=...
MERCHANT_NAME=Mole

# Video — 'livekit' (what we're using now) or 'zoom'
VIDEO_PROVIDER=livekit
# LiveKit Cloud (from the LiveKit project — wss URL + API key/secret)
LIVEKIT_URL=wss://your-project.livekit.cloud
LIVEKIT_API_KEY=...
LIVEKIT_API_SECRET=...
# Zoom Video SDK (only if VIDEO_PROVIDER=zoom — see Step 8)
# ZOOM_SDK_KEY=...
# ZOOM_SDK_SECRET=...
# ZOOM_ACCOUNT_ID=...

# WhatsApp (via Twilio — reuses TWILIO_ACCOUNT_SID / TWILIO_AUTH_TOKEN)
TWILIO_WHATSAPP_FROM=whatsapp:+14155238886   # sandbox; swap for your approved sender

# Push
FCM_SERVICE_ACCOUNT=...           # the service-account JSON (single line / path per your config)
# SENTRY_DSN=...                  # optional
```

### 6.2 Build + migrate + seed
Postgres must be up first (Step 3A `docker compose up -d`, or RDS at 3B).
```bash
cd /opt/mole/backend
pnpm prisma generate             # generate the Prisma client (backend imports it at runtime)
# No `pnpm build` — backend runs via tsx (start:prod), not compiled dist. See Step 7.1.
pnpm prisma migrate deploy       # applies all migrations to the DB
pnpm db:seed                     # optional: config defaults / demo data
ADMIN_PHONE=+91XXXXXXXXXX pnpm seed:admin   # first admin — logs in by phone + OTP
```
> Admin login is **phone + OTP** (no email/password). `seed:admin` reads `ADMIN_PHONE`
> — pass a real number you can receive the OTP on; it defaults to a placeholder otherwise.

### 6.3 Web + admin builds (compile-time API URL)
```bash
# Web (Next.js) — origin only; app appends /api/v1
cd /opt/mole/web
echo 'NEXT_PUBLIC_API_URL=https://api.yourdomain.com' > .env.local
pnpm build

# Admin (Vite) — served under its own subdomain at root base
cd /opt/mole/admin
pnpm build
```
> `NEXT_PUBLIC_*` is **baked at build** — changing the API URL later means a **rebuild** of web.

---

## Step 7 — Run under PM2 + nginx + TLS

### 7.1 PM2
Create `/opt/mole/ecosystem.config.js`:
```js
module.exports = {
  apps: [
    // Runs via tsx (start:prod), NOT `node dist`. @mole/shared ships raw .ts
    // (no build step — Next/Metro transpile it directly); plain node can't
    // resolve its extensionless imports, so the backend runs through tsx too.
    { name: 'mole-backend', cwd: '/opt/mole/backend', script: 'pnpm',
      args: 'start:prod', interpreter: 'none',
      env: { NODE_ENV: 'production', PORT: '3000' } },
    // PORT drives `next start` (it has no -p flag); without it Next binds 3000
    // and collides with the backend.
    { name: 'mole-web', cwd: '/opt/mole/web', script: 'pnpm', args: 'start',
      interpreter: 'none', env: { NODE_ENV: 'production', PORT: '3001' } },
    // `pnpm exec` (from the admin dir) — NOT `--filter … preview -- …`; the
    // `--` separator leaks into vite there, which then ignores the port and
    // falls back to 4173.
    { name: 'mole-admin', cwd: '/opt/mole/admin', script: 'pnpm',
      args: 'exec vite preview --host 127.0.0.1 --port 3002', interpreter: 'none' },
  ],
};
```
```bash
cd /opt/mole && pm2 start ecosystem.config.js && pm2 save
pm2 startup        # run the command it prints (survives reboot)
pm2 status         # all three: online
```
> Single backend instance = correct. The in-process `node-cron` jobs hold a lock that assumes **one**
> instance — never `pm2 scale` the backend.

### 7.2 nginx — one server block per subdomain
`sudo nano /etc/nginx/sites-available/mole`:
```nginx
server {                                   # backend API
  listen 80; server_name api.yourdomain.com;
  client_max_body_size 50M;                # KYC uploads
  location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $remote_addr;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
server {                                   # web (Next.js SSR)
  listen 80; server_name app.yourdomain.com;
  location / {
    proxy_pass http://127.0.0.1:3001;
    proxy_set_header Host $host;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
  }
}
server {                                   # admin (Vite preview)
  listen 80; server_name admin.yourdomain.com;
  location / {
    proxy_pass http://127.0.0.1:3002;
    proxy_set_header Host localhost;       # keeps vite's host-check happy
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
  }
}
```
```bash
sudo rm -f /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/mole /etc/nginx/sites-enabled/mole
sudo nginx -t && sudo systemctl reload nginx
```

### 7.3 TLS (certbot — free)

**Prerequisites (HTTP-01 validation fails otherwise):**
- The three GoDaddy **A records resolve to this box** (Step 5.3) — check each first:
  ```bash
  for h in api app admin; do echo -n "$h: "; dig +short $h.yourdomain.com; done
  # all three must print the Elastic IP
  ```
- **Port 80 open** to the world in `mole-app-sg` (Step 2) and nginx running.

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx --redirect --agree-tos -m you@yourdomain.com \
  -d api.yourdomain.com -d app.yourdomain.com -d admin.yourdomain.com
```
`--redirect` makes certbot add the HTTP→HTTPS 301 to every block; it also rewrites
them for 443 and installs a renewal timer.

Verify:
```bash
curl -I https://api.yourdomain.com          # 200/301, valid cert
sudo certbot renew --dry-run                 # confirms auto-renew works
```
> Certs auto-renew via the systemd timer. If you later move DNS to Route 53, nothing
> changes here — certbot still validates over HTTP-01 on port 80.

---

## Step 8 — Video provider

**We're using LiveKit Cloud now** (`VIDEO_PROVIDER=livekit`): no console/webhook setup beyond the
three `LIVEKIT_*` env vars in Step 6.1 — create a project at `cloud.livekit.io`, copy its wss URL +
API key/secret. Recording (LiveKit Egress → S3) is a later follow-up. The rest of this step applies
**only if you switch to Zoom** (`VIDEO_PROVIDER=zoom`).

### Zoom Video SDK (alternative — video + recording)

1. **Zoom Marketplace** (`marketplace.zoom.us`) → **Develop → Build App** → **Video SDK**.
   - Copy **SDK Key** + **SDK Secret** → `.env` `ZOOM_SDK_KEY` / `ZOOM_SDK_SECRET`.
2. **Server-to-Server OAuth** (recording download uses account-credentials grant):
   - Build a **Server-to-Server OAuth** app too → copy the **Account ID** → `.env` `ZOOM_ACCOUNT_ID`.
   - Scopes: enable **Video SDK** + **cloud recording** read/download.
3. **Enable Cloud Recording** for the account (Zoom account settings → Recording).
4. **Recording webhook** → point Zoom at your backend:
   - In the app's **Feature/Event Subscriptions**, add endpoint:
     `https://api.yourdomain.com/api/v1/recordings/zoom-webhook`
   - Subscribe to the **recording completed** event.
   - Zoom sends a **validation** challenge — the endpoint is HMAC-verified; make sure the secret Zoom
     shows matches what the controller validates.
5. Restart backend after env changes: `pm2 restart mole-backend`.

Flow at runtime: session ends → Zoom finishes cloud recording → POSTs the webhook → backend downloads
the file → uploads to `s3://mole-media-prod/recordings/{bookingId}/…` → students play via the presigned
`GET /api/v1/recordings/:bookingId`. The Step 4.4 lifecycle rule expires the S3 object after 2 days.

---

## Step 9 — CloudFront (optional at launch)

Recordings + private docs are already served via **backend presigned S3 URLs**, so **you can go live
without CloudFront.** Add it to (a) cut S3 egress on public assets (avatars, static images) and (b) get
a CDN edge for the web app's media.

1. **ACM cert in us-east-1** (CloudFront only reads certs from us-east-1): request a public cert for
   `cdn.yourdomain.com`, DNS-validate.
2. **CloudFront → Create distribution:**
   - Origin: the S3 bucket `mole-media-prod` (**REST endpoint**, not website endpoint).
   - **Origin access: Origin Access Control (OAC)** → let CloudFront create it; then it prints a
     bucket policy — **paste that policy** onto the bucket so CloudFront (and only it) can read.
   - Viewer protocol: **Redirect HTTP→HTTPS**.
   - Alternate domain name: `cdn.yourdomain.com`; attach the ACM cert.
3. GoDaddy DNS: add a **CNAME** `cdn` → the CloudFront `*.cloudfront.net` domain. (ACM cert
   validation for CloudFront adds a **CNAME** record too — copy the name/value ACM shows into GoDaddy.)
4. Set `.env` `CDN_BASE_URL=https://cdn.yourdomain.com`, `pm2 restart mole-backend`.
> Keep recordings served through the **presigned backend URL** (private), not public CloudFront —
> CDN is for public assets only.

---

## Step 10 — Mobile app → prod API

The APK bakes the API URL at build time. In `mobile/`, set the production build env:
```
EXPO_PUBLIC_API_URL=https://api.yourdomain.com
```
Then build/submit via EAS — see `docs/deployment/aws-deploy.md` → "Releasing the Android app".

---

## Step 11 — Smoke test (do all before announcing)

| # | Check | How |
|---|---|---|
| 1 | Web loads over TLS | open `https://app.yourdomain.com` |
| 2 | Admin loads | `https://admin.yourdomain.com`, log in with `seed:admin` creds |
| 3 | API health | `curl https://api.yourdomain.com/health` → `{"status":"ok"}` |
| 4 | **OTP send + verify** | request OTP on web → SMS arrives (Twilio) → verify logs in |
| 5 | OTP throttle | send twice fast → 2nd returns `429 OTP_RATE_LIMITED` |
| 6 | Upload | complete a KYC/photo upload → object appears in S3 |
| 7 | Booking → payment | book a session, pay (Razorpay test) → status `confirmed` |
| 8 | **Video** | both join → Zoom Video SDK session connects |
| 9 | **Recording** | end session → after Zoom processes, object lands in `recordings/` and playback works |
| 10 | Push | trigger a notification → FCM arrives (Android dev build) |
| 11 | Crons | `pm2 logs mole-backend` shows cron ticks (e.g. `token-cleanup`) |

---

## Step 12 — Observability & alerting

The backend already ships a lot of this — don't rebuild it:
- **Health endpoint:** `GET /health` → `{"status":"ok"}` (root path, excluded from logs).
- **Structured logs:** pino JSON with a per-request `x-request-id` and **global secret redaction**
  (auth headers, OTP, password never logged). In prod it's machine-parseable JSON on stdout.
- **Error tracking:** Sentry initialises when `SENTRY_DSN` is set, and **redacts `phone`** before
  send. Errors are captured in the Express error handler.

### 12.1 Errors — Sentry (managed, free tier)
1. `sentry.io` → create a **Node** project → copy the DSN.
2. Set `SENTRY_DSN=...` in `backend/.env`, `pm2 restart mole-backend`.
3. Add `@sentry/react-native` to the mobile app with the same org for client crashes.
> Self-hosted alternative: **GlitchTip** (open-source, Sentry-API-compatible — the same SDK/DSN
> points at your own server) or **self-hosted Sentry**. See the OSS table below.

### 12.2 Logs — CloudWatch agent (ships PM2 output)
```bash
sudo apt install -y amazon-cloudwatch-agent
# minimal config: tail the PM2 logs into a log group
sudo tee /opt/aws/amazon-cloudwatch-agent/etc/config.json >/dev/null <<'JSON'
{ "logs": { "logs_collected": { "files": { "collect_list": [
  { "file_path": "/root/.pm2/logs/mole-backend-out.log", "log_group_name": "/mole/backend", "log_stream_name": "{instance_id}-out" },
  { "file_path": "/root/.pm2/logs/mole-backend-error.log", "log_group_name": "/mole/backend", "log_stream_name": "{instance_id}-err" }
] } } } }
JSON
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/etc/config.json -s
```
(The EC2 instance role needs `CloudWatchAgentServerPolicy`.) Because logs are JSON, you can query them
with **CloudWatch Logs Insights** by `x-request-id`, status, etc.

### 12.3 Metrics + alarms — CloudWatch → SNS → your phone
1. **SNS topic** `mole-alerts` → subscribe your email (and/or an SMS endpoint).
2. Create alarms (Console → CloudWatch → Alarms), each notifying `mole-alerts`:
   | Alarm | Metric | Threshold |
   |---|---|---|
   | EC2 CPU high | `CPUUtilization` | >80% for 5 min |
   | EC2 status check | `StatusCheckFailed` | ≥1 |
   | RDS CPU high | RDS `CPUUtilization` | >80% |
   | RDS storage low | `FreeStorageSpace` | < 2 GB |
   | RDS connections | `DatabaseConnections` | near max |
   | S3 recordings size | `BucketSizeBytes` (recordings bucket) | > your cap → lifecycle broke |

### 12.4 Uptime — external ping
CloudWatch can't see a fully-dead box. Add a free external monitor hitting **`/health`** every 1–5 min:
**UptimeRobot** / **BetterStack** / **Checkly** (managed free tiers), or self-host **Uptime Kuma**
(open-source, one Docker container — the nicest free option).

### 12.5 Cost alarms (Mole-specific — the real risk)
Your dominant spend is **Zoom minutes + S3 recordings**, not the server:
- **AWS Budgets** → monthly budget with an 80% email alert.
- **CloudWatch `BucketSizeBytes`** on the recordings bucket (12.3) — the early warning that the
  lifecycle expiry rule stopped reclaiming space.
- Watch the **Zoom billing dashboard** — Video SDK minutes are metered and can run away.

### 12.6 Free & open-source stacks (if you'd rather self-host)
All run as Docker containers on the same box (or a small separate one):

| Need | Managed free | Open-source self-host |
|---|---|---|
| Errors | Sentry free tier | **GlitchTip** (Sentry-compatible), self-hosted Sentry |
| Metrics + dashboards | Grafana Cloud free, AWS CloudWatch | **Prometheus + Grafana** (scrape `node_exporter` + a `/metrics` endpoint) |
| Logs | CloudWatch Logs, Grafana Cloud Loki | **Grafana Loki + Promtail** (pino JSON ships straight in), or **OpenSearch** |
| Uptime | UptimeRobot / BetterStack | **Uptime Kuma** |
| All-in-one (traces+logs+metrics) | — | **SigNoz** (OpenTelemetry-native), or **Grafana LGTM** stack |
| APM / tracing | Grafana Cloud Tempo | **SigNoz**, **Jaeger** + OpenTelemetry SDK |

> The logger already emits JSON "ready for Loki/SigNoz", so the cheapest real setup is:
> **Sentry/GlitchTip (errors) + Loki (logs) + Grafana + Uptime Kuma** — all free/OSS. For a single
> launch box, **managed free tiers (Sentry + CloudWatch + UptimeRobot + AWS Budgets)** are less to
> operate; self-host only when you want everything in one pane (SigNoz) or to cut managed costs.

### 12.7 Self-hosted observability box — step by step (separate EC2, 7-day retention)

Run the whole stack on its **own small EC2** so a Grafana/Loki spike can't touch the API box. Keep it
**same VPC + same AZ** as the app box → log/metric traffic goes over the **private IP and is free**
(cross-AZ would be ~$0.01/GB; internet egress $0.09/GB — avoid both). Stack: **Loki** (logs) +
**Grafana** (UI) + **Uptime Kuma** (uptime), plus **Prometheus** (metrics) once the backend exposes
`/metrics`. All Docker. Cost ≈ **t3.small ~$15 + ~30 GB gp3 ~$3 = ~$18/mo**, ~$0 transfer.

**1. Launch the obs instance**
- **t3.small** (2 GB min; t3.medium if you add Prometheus + SigNoz later), Ubuntu 22.04, **30 GB gp3**.
- **Same VPC and same Availability Zone** as `mole-app` (check the app instance's AZ and match it).
- Install Docker + compose:
  ```bash
  curl -fsSL https://get.docker.com | sh
  sudo apt install -y docker-compose-plugin
  ```

**2. Security group `mole-obs-sg`** (private-first — nothing public except your own IP):
| Port | Source | Why |
|---|---|---|
| 22 | **your IP** | SSH |
| 3100 | **`mole-app-sg`** | Loki push (app → obs, private) |
| 3000 | **your IP** | Grafana UI |
| 3001 | **your IP** | Uptime Kuma UI |
| 9090 | your IP (optional) | Prometheus UI |

And on **`mole-app-sg`**: if you add Prometheus, allow the metrics port (e.g. **9464**) inbound
**from `mole-obs-sg`** so Prometheus can scrape the app box. Log shipping needs no app-side inbound
(the app *pushes* to Loki).

**3. `docker-compose.yml` on the obs box** (`/opt/obs/docker-compose.yml`):
```yaml
services:
  loki:
    image: grafana/loki:3.0.0
    command: -config.file=/etc/loki/config.yml
    volumes: [ "./loki-config.yml:/etc/loki/config.yml", "loki-data:/loki" ]
    ports: [ "3100:3100" ]
    restart: unless-stopped
  grafana:
    image: grafana/grafana:11.1.0
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=CHANGE_ME
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes: [ "grafana-data:/var/lib/grafana" ]
    ports: [ "3000:3000" ]
    restart: unless-stopped
  uptime-kuma:
    image: louislam/uptime-kuma:1
    volumes: [ "kuma-data:/app/data" ]
    ports: [ "3001:3001" ]
    restart: unless-stopped
volumes: { loki-data: {}, grafana-data: {}, kuma-data: {} }
```

**4. `loki-config.yml` — 7-day retention** (`/opt/obs/loki-config.yml`):
```yaml
auth_enabled: false
server: { http_listen_port: 3100 }
common:
  instance_addr: 127.0.0.1
  path_prefix: /loki
  storage: { filesystem: { chunks_directory: /loki/chunks, rules_directory: /loki/rules } }
  replication_factor: 1
  ring: { kvstore: { store: inmemory } }
schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v13
      index: { prefix: index_, period: 24h }
limits_config:
  retention_period: 168h            # 7 days
compactor:
  working_directory: /loki/compactor
  delete_request_store: filesystem
  retention_enabled: true           # actually deletes past 7d
  retention_delete_delay: 2h
```
```bash
cd /opt/obs && sudo docker compose up -d
```

**5. Ship the app's logs → Loki (promtail on the app box)**
On **`mole-app`**, run promtail to tail the PM2 logs and push to Loki over the **private IP**:
```yaml
# /opt/promtail/config.yml   (use the obs box PRIVATE ip)
server: { http_listen_port: 9080 }
positions: { filename: /tmp/positions.yaml }
clients:
  - url: http://OBS_PRIVATE_IP:3100/loki/api/v1/push
scrape_configs:
  - job_name: mole-backend
    static_configs:
      - targets: [ localhost ]
        labels: { job: mole-backend, host: app }
        __path__: /root/.pm2/logs/mole-backend-*.log
```
```bash
sudo docker run -d --name promtail --restart unless-stopped \
  -v /opt/promtail/config.yml:/etc/promtail/config.yml \
  -v /root/.pm2/logs:/root/.pm2/logs:ro \
  grafana/promtail:3.0.0 -config.file=/etc/promtail/config.yml
```
Because the logs are pino JSON, add a `| json` stage in Grafana Explore to filter by `x-request-id`,
status, etc.

**6. Wire Grafana**
Open `http://OBS_PUBLIC_IP:3000` (or SSH-tunnel `-L 3000:localhost:3000` and use localhost — no public
exposure). Log in, add **Loki** datasource `http://loki:3100`. Import a Loki logs dashboard. In
**Uptime Kuma** (`:3001`) add an HTTP monitor on `https://api.yourdomain.com/health`, and set its
**data retention to 7 days** in Settings.

**7. Metrics (optional, later)** — the backend has `/health` but no `/metrics` exporter yet. Once a
`prom-client` `/metrics` endpoint exists, add Prometheus to the compose with
`--storage.tsdb.retention.time=7d` and a scrape job targeting `APP_PRIVATE_IP:9464`, then add it as a
Grafana datasource. Until then, logs + uptime already cover the essentials.

> **Want one container instead of four?** **SigNoz** (OpenTelemetry-native: logs + metrics + traces in
> one UI) is the alternative — heavier (needs t3.medium, ClickHouse), and 7-day retention is a
> ClickHouse **TTL = 7 days** setting. Point the backend's OTel exporter at it. Good when you want
> tracing; the Loki+Grafana+Kuma trio above is lighter for a launch.

## Step 13 — Post-launch hygiene

- **DB backups:** RDS automated backups are on (Step 3). Optionally add a nightly `pg_dump` to S3.
- **Monitoring:** CloudWatch alarms on EC2 CPU + RDS CPU/storage; add `SENTRY_DSN` for app errors.
- **Secrets:** rotate the Twilio + WhatsApp tokens that were shared in chat; never commit `.env`.
- **Zoom cost:** Video SDK is **metered per participant-minute** — watch the Zoom billing dashboard;
  it's your single largest cloud line item (far above the EC2 bill).
- **Updates / redeploy:** see [Redeploying an update](#redeploying-an-update) below.

## Redeploying an update

Rebuild and restart **only the apps the release touched**. Check with
`git diff --stat <deployed-sha> origin/main` before pulling.

### 1. Pull (always)
```bash
cd /opt/mole
git status                              # must be clean — no hand edits on the box
git rev-parse HEAD > ~/pre-deploy-sha   # for rollback
git pull origin main
pnpm install --frozen-lockfile
```
`pnpm install` must print the **`dedupe-web-react`** postinstall step (root
`scripts/dedupe-web-react.mjs`, PR #236). Without it, `next build` fails on `/404`
with a null `useContext` (hoisted React 18/19 mismatch).

### 2. Backend: only if `backend/` or `shared/` changed
```bash
cd /opt/mole/backend
pnpm prisma generate              # if schema.prisma changed (safe to always run)
pnpm prisma migrate deploy        # if prisma/migrations/ changed
pm2 restart mole-backend
```
**No `pnpm build`.** The backend runs via **tsx** (`pnpm start:prod`), not compiled
`dist/`. `@mole/shared` ships raw TS that plain `node` can't resolve.

### 3. Web: only if `web/` or `shared/` changed
```bash
cd /opt/mole/web
pnpm build                        # uses .env.local (NEXT_PUBLIC_API_URL baked in)
pm2 restart mole-web
```

### 4. Admin: only if `admin/` or `shared/` changed
```bash
cd /opt/mole/admin
pnpm build
pm2 restart mole-admin
```

### 5. Verify
```bash
pm2 status                                     # all online, restarts not climbing
pm2 logs <app> --lines 50                      # no startup errors
curl -s  https://api.<domain>/health                     # backend → {"status":"ok"}
curl -sI https://app.<domain>/welcome       | head -1    # web
```

### Rollback
```bash
cd /opt/mole && git checkout $(cat ~/pre-deploy-sha)
pnpm install --frozen-lockfile
# rebuild + restart the same apps as above
git checkout main                              # after the fix lands
```
Migrations don't roll back automatically. If a release ran `migrate deploy`,
check whether the old code works with the new schema before rolling back.

> `mobile/` changes don't deploy here. They ship in a new APK/AAB build, or OTA once
> configured (see `google-play-release-plan.md` and `mobile-ota-updates.md`).

## Troubleshooting

| Symptom | Fix |
|---|---|
| nginx default page | removed `sites-enabled/default`? `nginx -t` passed? |
| Admin "host not allowed" | ensure `proxy_set_header Host localhost;` in the admin block |
| Web API 404/CORS | `NEXT_PUBLIC_API_URL` is the **origin only**; rebuild web after changing it |
| Upload 403 to S3 | app user/role missing the Step 4.3 policy, or wrong `AWS_S3_BUCKET`/region |
| Recording never lands in S3 | Zoom webhook URL/secret wrong; `pm2 logs mole-backend`; cloud recording enabled? |
| Playback 403 | presigned URL expired or bucket policy blocks the app identity |
| DB connect refused | `mole-rds-sg` must allow 5432 **from `mole-app-sg`**; RDS not public |
| Build OOM | add swap (Step 5.4) or build in CI |
| Cron double-fires | you scaled the backend — keep it a **single** instance |
