# Deployment Guide — Mole Backend

---

## Prerequisites

| Tool | Version |
|------|---------|
| Node.js | 20+ |
| pnpm | 9+ |
| PostgreSQL | 15+ |
| AWS CLI | configured with S3 access |

---

## Environment Setup

Copy and fill all required values:

```bash
cp .env.example .env
```

### Required Variables

```env
# Core
NODE_ENV=production
PORT=3000
DATABASE_URL=postgresql://user:pass@host:5432/mole_prod

# JWT — generate strong random secrets (32+ chars)
JWT_ACCESS_SECRET=
JWT_REFRESH_SECRET=
TEMP_TOKEN_SECRET=

# Twilio (SMS / OTP)
TWILIO_ACCOUNT_SID=ACxxxxxxxxxxxxxxxxxxxxxxxx
TWILIO_AUTH_TOKEN=
TWILIO_FROM_NUMBER=+1xxxxxxxxxx

# Firebase (push notifications)
# Paste full service account JSON as a single-line string
FCM_SERVICE_ACCOUNT={"type":"service_account","project_id":"...","private_key":"..."}

# AWS S3
AWS_REGION=ap-south-1
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_S3_BUCKET=mole-media-prod
AWS_S3_KEY_PREFIX=uploads/
CDN_BASE_URL=https://cdn.mole.app   # or your CloudFront domain

# UPI Payments
MERCHANT_VPA=yourupi@bankname
MERCHANT_NAME=Mole
UPI_WEBHOOK_SECRET=      # shared secret with payment gateway

# Zoom Video SDK
ZOOM_SDK_KEY=
ZOOM_SDK_SECRET=
ZOOM_ACCOUNT_ID=
ZOOM_WEBHOOK_SECRET=     # from Zoom webhook config

# WhatsApp (Twilio — reuses TWILIO_ACCOUNT_SID / TWILIO_AUTH_TOKEN)
TWILIO_WHATSAPP_FROM=whatsapp:+14155238886   # sandbox; swap for your approved sender
API_BASE_URL=https://api.mole.app   # public URL for WhatsApp accept/decline links
```

### Optional (defaults to dev fallbacks)

```env
UPLOAD_DIR=./uploads     # local file storage dir (dev only, use S3 in prod)
```

---

## First Deploy

### 1. Install dependencies

```bash
pnpm install --frozen-lockfile
```

### 2. Run database migrations

```bash
pnpm db:migrate
# or for production:
npx prisma migrate deploy
```

### 3. Seed initial data

```bash
pnpm db:seed
# creates: subjects, topics, first admin user
```

### 4. Build

```bash
pnpm build
# outputs to: dist/
```

### 5. Start

```bash
pnpm start
# runs: node dist/index.js
```

---

## Subsequent Deploys

```bash
git pull
pnpm install --frozen-lockfile
pnpm build
npx prisma migrate deploy   # apply any new migrations
pm2 restart mole-backend    # or your process manager
```

---

## Process Management (PM2)

```bash
pm2 start dist/index.js --name mole-backend
pm2 save
pm2 startup   # enable on system reboot
```

---

## Nginx Reverse Proxy

```nginx
server {
    listen 443 ssl;
    server_name api.mole.app;

    ssl_certificate /etc/letsencrypt/live/api.mole.app/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.mole.app/privkey.pem;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

---

## Database Migrations

```bash
# Create new migration (dev only)
pnpm db:migrate:dev -- --name describe_change

# Apply migrations (production — no schema reset)
npx prisma migrate deploy

# View current migration status
npx prisma migrate status

# Browse data
pnpm db:studio    # http://localhost:5555 — dev only
```

---

## Webhook Configuration

### UPI Webhook
Point your UPI payment gateway to:
```
POST https://api.mole.app/api/v1/payment/webhook
```
Set `UPI_WEBHOOK_SECRET` to match what the gateway sends in the HMAC header.

### Zoom Recording Webhook
In Zoom Marketplace → your Video SDK app → Event Subscriptions:
```
POST https://api.mole.app/api/v1/recordings/zoom-webhook
```
Events to subscribe: `recording.completed`
Set `ZOOM_WEBHOOK_SECRET` from the Zoom webhook secret token.

### WhatsApp Webhook (Twilio)
In Twilio Console → Messaging → your WhatsApp sender/sandbox → "When a message comes in":
```
POST https://api.mole.app/api/v1/webhooks/whatsapp
```
No verify token — Twilio signs each request (checked via `X-Twilio-Signature`).

---

## AWS S3 Setup

### Bucket Policy (allow backend to read/write)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT_ID:user/mole-backend"
      },
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::mole-media-prod/*"
    }
  ]
}
```

### CORS (for direct upload presigned URLs)
```json
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["GET", "PUT", "POST"],
    "AllowedOrigins": ["https://admin.mole.app", "https://api.mole.app"],
    "ExposeHeaders": ["ETag"]
  }
]
```

---

## Firebase Setup

1. Firebase Console → Project Settings → Service Accounts → Generate new private key
2. Download JSON
3. Stringify to single line: `jq -c . serviceAccount.json`
4. Set as `FCM_SERVICE_ACCOUNT` env var

---

## Monitoring

**Sentry** — set `SENTRY_DSN` in `.env` (auto-detected by Sentry SDK).

**Logs** — Pino structured JSON logs to stdout. In production, pipe to your log aggregator (Datadog, Loki, CloudWatch):
```bash
node dist/index.js 2>&1 | pino-pretty     # dev
node dist/index.js | your-log-shipper     # prod
```

**Health check endpoint**:
```
GET /api/v1/health   → 200 OK
```

---

## Mobile App (Expo EAS)

```bash
cd mobile
eas build --platform ios --profile production
eas build --platform android --profile production
eas submit --platform ios
eas submit --platform android
```

Set production `API_BASE_URL` in `constants/config.ts` or via EAS environment variables.

---

## Admin App

```bash
cd admin
pnpm build
# Deploy dist/ to any static host (Netlify, Vercel, Nginx)
```

Set `VITE_API_BASE_URL=https://api.mole.app/api/v1` in production environment.

---

## Troubleshooting

| Issue | Check |
|-------|-------|
| `DATABASE_URL` connection failed | Postgres running? Credentials correct? SSL needed? |
| OTP not sending | `TWILIO_*` vars set? Check Twilio console for errors |
| Push not delivered | `FCM_SERVICE_ACCOUNT` JSON valid? Device FCM token registered? |
| WhatsApp not sending | `TWILIO_WHATSAPP_FROM` set + `whatsapp:`-prefixed? Recipient joined the sandbox / inside 24h window? |
| Zoom session token error | `ZOOM_SDK_KEY`/`ZOOM_SDK_SECRET` set correctly? |
| S3 upload failing | IAM user has `s3:PutObject` permission? Bucket region matches `AWS_REGION`? |
| Payment webhook not triggering | `UPI_WEBHOOK_SECRET` matches gateway config? Endpoint is HTTPS? |
