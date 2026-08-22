# Mole on AWS — ECS Fargate go-live runbook (Path B)

Containerised, managed shape: **backend on ECS Fargate** behind an **ALB**, **web on ECS Fargate**
(or Amplify), **admin on S3 + CloudFront**, **RDS** Postgres, **S3** for uploads + Zoom recordings.
Video is **Zoom Video SDK** (cloud). More moving parts than the EC2 runbook, scales cleanly, costs
more (ALB + Fargate + CloudFront). **Only choose this over EC2 when one box is no longer enough.**

**End state**
```
        Route 53 ── ACM certs
           │
   ┌───────┴─────────────────────────────────────────────┐
   │                                                       │
 app.  (Amplify OR ECS svc)        api.yourdomain.com      admin.  (CloudFront→S3)
   │                                    │
   │                          ┌─────────▼──────────┐
   │                          │  ALB (HTTPS)       │
   │                          └─────────┬──────────┘
   │                          ┌─────────▼──────────┐   ┌───────────┐
   └──────────────────────────│ ECS Fargate        │◄─►│  RDS PG16 │
                              │  backend service    │   └───────────┘
                              │  desired count = 1  │   ┌───────────┐
                              │  (cron lock!)       │◄─►│ S3 bucket │
                              └─────────────────────┘   │ private   │
                                        ▲               └───────────┘
                             Zoom Video SDK cloud
                          (recording webhook → ALB → S3)
```

**Hard constraints (same codebase facts as the EC2 runbook):**
- **Backend desired count MUST be 1.** The backend runs `node-cron` in-process behind a single-holder
  lock. Two tasks = **double-fired crons** (double payouts, double notifications). Do **not** enable
  autoscaling on the backend service until the crons are extracted into a separate scheduled task.
- Prisma uses **only `DATABASE_URL`**. Run `prisma migrate deploy` as a **one-off task**, never in the
  service's boot command (task restarts would race the migration).
- Web is **SSR Next.js** → needs a Node runtime (Fargate task or Amplify), not a static bucket.
- Zoom recording webhook `POST /api/v1/recordings/zoom-webhook` must be reachable via the ALB
  (public). Playback is a backend **presigned S3 URL** → bucket stays private.
- Recordings have a 24h access expiry but **no S3-delete cron** → the S3 **lifecycle rule is required**.

**Region:** `ap-south-1` in examples. CloudFront/ACM-for-CloudFront certs → **us-east-1**.

> **Shared AWS resources.** RDS (Step 3), the S3 bucket + lifecycle (Step 4), Route 53 DNS, and the
> Zoom app (Step 9) are configured **exactly as in `aws-ec2-runbook.md`**. This doc restates the
> essentials but points there for full detail so the two don't drift.

---

## Step 0 — Prerequisites

- Docker installed locally, **AWS CLI v2** configured (`aws configure` with an admin IAM user).
- Domain in Route 53 (or DNS you control).
- Provider creds ready: Twilio, Razorpay, Zoom Video SDK, WhatsApp, FCM.
- Generate the app secrets (store in **SSM**, Step 6):
  ```bash
  openssl rand -hex 32   # x4: JWT_ACCESS_SECRET, JWT_REFRESH_SECRET, TEMP_TOKEN_SECRET, UPI_WEBHOOK_SECRET
  ```
- Set shell vars used throughout:
  ```bash
  export AWS_REGION=ap-south-1
  export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
  export ECR=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
  ```

---

## Step 1 — Networking (VPC + security groups)

Use the default VPC (has public + it's simplest). Create SGs:

| SG | Inbound | Source |
|---|---|---|
| `mole-alb-sg` | 80, 443 | 0.0.0.0/0 |
| `mole-backend-sg` | 3000 | `mole-alb-sg` |
| `mole-web-sg` (if web on ECS) | 3001 | `mole-alb-sg` |
| `mole-rds-sg` | 5432 | `mole-backend-sg` (+ `mole-web-sg` if web queries DB — it doesn't; web calls the API) |

Fargate tasks run in the VPC; give them the public subnets + "assign public IP" (or private subnets +
a NAT gateway — NAT adds ~$32/mo, so public subnets are the cheaper launch choice).

---

## Step 2 — RDS Postgres

Identical to `aws-ec2-runbook.md` Step 3. Summary:
- PostgreSQL 16, **db.t4g.small**, 20 GB gp3, **not public**, SG **`mole-rds-sg`**, DB name `mole`,
  master `mole` + strong password, automated backups on.
- Connection string (goes into SSM as a secret in Step 6):
  ```
  DATABASE_URL=postgresql://mole:PASSWORD@ENDPOINT:5432/mole
  ```

---

## Step 3 — S3 bucket (uploads + recordings)

Identical to `aws-ec2-runbook.md` Step 4 — do all four parts:
1. Bucket `mole-media-prod`, **block all public access ON**, SSE-S3.
2. **CORS** allowing `https://app.yourdomain.com` + `https://admin.yourdomain.com` (GET/PUT/HEAD).
3. IAM policy (`s3:PutObject/GetObject/DeleteObject` + `ListBucket`) — here it attaches to the **ECS
   task role** (Step 5), not a user.
4. **Lifecycle rule REQUIRED:** expire `recordings/` prefix after **2 days** (no delete cron exists).

---

## Step 4 — Build + push the backend image to ECR

### 4.1 Dockerfile
Add `backend/Dockerfile` (monorepo-aware — build from repo root context):
```dockerfile
# --- build stage ---
FROM node:20-slim AS build
WORKDIR /app
RUN apt-get update && apt-get install -y openssl && corepack enable
COPY . .
RUN pnpm install --frozen-lockfile \
 && pnpm --filter @mole/backend prisma generate \
 && pnpm --filter @mole/backend build

# --- runtime stage ---
FROM node:20-slim
WORKDIR /app
RUN apt-get update && apt-get install -y openssl && rm -rf /var/lib/apt/lists/*
COPY --from=build /app /app
ENV NODE_ENV=production PORT=3000
EXPOSE 3000
CMD ["node", "backend/dist/index.js"]
```
> `openssl` is needed for Prisma on `-slim`. The build context is the **repo root** (pnpm workspace),
> not `backend/`.

### 4.2 Create the ECR repo + push
```bash
aws ecr create-repository --repository-name mole-backend --region $AWS_REGION
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR

# build from repo root with the backend Dockerfile
docker build -f backend/Dockerfile -t mole-backend .
docker tag mole-backend:latest $ECR/mole-backend:latest
docker push $ECR/mole-backend:latest
```

---

## Step 5 — IAM roles for ECS

Two roles:
1. **Task execution role** `mole-ecsTaskExecutionRole` — lets ECS pull from ECR + write logs + read
   SSM secrets. Attach AWS managed **`AmazonECSTaskExecutionRolePolicy`**, plus an inline policy for
   SSM:
   ```json
   { "Version":"2012-10-17","Statement":[
     {"Effect":"Allow","Action":["ssm:GetParameters","ssm:GetParameter"],
      "Resource":"arn:aws:ssm:ap-south-1:ACCOUNT_ID:parameter/mole/*"} ]}
   ```
2. **Task role** `mole-ecsTaskRole` — what the app itself uses at runtime. Attach the **S3 policy**
   from Step 3.3 (so `STORAGE_PROVIDER=s3` works with **no static keys** — omit
   `AWS_ACCESS_KEY_ID`/`SECRET` entirely).

---

## Step 6 — Secrets in SSM Parameter Store

Store every secret as a **SecureString** under `/mole/`. Example:
```bash
put() { aws ssm put-parameter --name "/mole/$1" --value "$2" --type SecureString --overwrite --region $AWS_REGION; }

put DATABASE_URL            'postgresql://mole:PASSWORD@ENDPOINT:5432/mole'
put JWT_ACCESS_SECRET       '...'
put JWT_REFRESH_SECRET      '...'
put TEMP_TOKEN_SECRET       '...'
put UPI_WEBHOOK_SECRET      '...'
put TWILIO_ACCOUNT_SID      '...'
put TWILIO_AUTH_TOKEN       '...'
put RAZORPAY_KEY_ID         '...'
put RAZORPAY_KEY_SECRET     '...'
put RAZORPAY_WEBHOOK_SECRET '...'
put ZOOM_SDK_KEY            '...'
put ZOOM_SDK_SECRET         '...'
put ZOOM_ACCOUNT_ID         '...'
put TWILIO_AUTH_TOKEN       '...'   # also signs WhatsApp sends + inbound webhook
put FCM_SERVICE_ACCOUNT     '...'
```
Non-secret config (`NODE_ENV`, `PORT`, `STORAGE_PROVIDER`, `AWS_REGION`, `AWS_S3_BUCKET`,
`API_BASE_URL`, `ADMIN_APP_URL`, `TWILIO_FROM_NUMBER`, `TWILIO_WHATSAPP_FROM`, `SMS_PROVIDER`,
`PAYMENT_PROVIDER`, `MERCHANT_*`, `CDN_BASE_URL`) go as **plain env** in the task def (Step 7).

---

## Step 7 — ECS cluster, task def, service (backend)

### 7.1 Cluster
```bash
aws ecs create-cluster --cluster-name mole --region $AWS_REGION
```
Create a CloudWatch log group:
```bash
aws logs create-log-group --log-group-name /ecs/mole-backend --region $AWS_REGION
```

### 7.2 Task definition
`backend-taskdef.json` (fill `ACCOUNT_ID`, region, image tag):
```json
{
  "family": "mole-backend",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::ACCOUNT_ID:role/mole-ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::ACCOUNT_ID:role/mole-ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "backend",
      "image": "ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/mole-backend:latest",
      "portMappings": [{ "containerPort": 3000, "protocol": "tcp" }],
      "environment": [
        { "name": "NODE_ENV", "value": "production" },
        { "name": "PORT", "value": "3000" },
        { "name": "STORAGE_PROVIDER", "value": "s3" },
        { "name": "AWS_REGION", "value": "ap-south-1" },
        { "name": "AWS_S3_BUCKET", "value": "mole-media-prod" },
        { "name": "SMS_PROVIDER", "value": "twilio" },
        { "name": "TWILIO_FROM_NUMBER", "value": "+1..." },
        { "name": "PAYMENT_PROVIDER", "value": "razorpay" },
        { "name": "MERCHANT_NAME", "value": "Mole" },
        { "name": "API_BASE_URL", "value": "https://api.yourdomain.com" },
        { "name": "ADMIN_APP_URL", "value": "https://admin.yourdomain.com" },
        { "name": "TWILIO_WHATSAPP_FROM", "value": "whatsapp:+14155238886" }
      ],
      "secrets": [
        { "name": "DATABASE_URL",            "valueFrom": "arn:aws:ssm:ap-south-1:ACCOUNT_ID:parameter/mole/DATABASE_URL" },
        { "name": "JWT_ACCESS_SECRET",       "valueFrom": "arn:aws:ssm:ap-south-1:ACCOUNT_ID:parameter/mole/JWT_ACCESS_SECRET" },
        { "name": "JWT_REFRESH_SECRET",      "valueFrom": "arn:aws:ssm:ap-south-1:ACCOUNT_ID:parameter/mole/JWT_REFRESH_SECRET" },
        { "name": "TEMP_TOKEN_SECRET",       "valueFrom": "arn:aws:ssm:ap-south-1:ACCOUNT_ID:parameter/mole/TEMP_TOKEN_SECRET" },
        { "name": "UPI_WEBHOOK_SECRET",      "valueFrom": "arn:aws:ssm:ap-south-1:ACCOUNT_ID:parameter/mole/UPI_WEBHOOK_SECRET" },
        { "name": "TWILIO_ACCOUNT_SID",      "valueFrom": "arn:aws:ssm:ap-south-1:ACCOUNT_ID:parameter/mole/TWILIO_ACCOUNT_SID" },
        { "name": "TWILIO_AUTH_TOKEN",       "valueFrom": "arn:aws:ssm:ap-south-1:ACCOUNT_ID:parameter/mole/TWILIO_AUTH_TOKEN" },
        { "name": "RAZORPAY_KEY_ID",         "valueFrom": "arn:aws:ssm:ap-south-1:ACCOUNT_ID:parameter/mole/RAZORPAY_KEY_ID" },
        { "name": "RAZORPAY_KEY_SECRET",     "valueFrom": "arn:aws:ssm:ap-south-1:ACCOUNT_ID:parameter/mole/RAZORPAY_KEY_SECRET" },
        { "name": "RAZORPAY_WEBHOOK_SECRET", "valueFrom": "arn:aws:ssm:ap-south-1:ACCOUNT_ID:parameter/mole/RAZORPAY_WEBHOOK_SECRET" },
        { "name": "ZOOM_SDK_KEY",            "valueFrom": "arn:aws:ssm:ap-south-1:ACCOUNT_ID:parameter/mole/ZOOM_SDK_KEY" },
        { "name": "ZOOM_SDK_SECRET",         "valueFrom": "arn:aws:ssm:ap-south-1:ACCOUNT_ID:parameter/mole/ZOOM_SDK_SECRET" },
        { "name": "ZOOM_ACCOUNT_ID",         "valueFrom": "arn:aws:ssm:ap-south-1:ACCOUNT_ID:parameter/mole/ZOOM_ACCOUNT_ID" },
        { "name": "FCM_SERVICE_ACCOUNT",     "valueFrom": "arn:aws:ssm:ap-south-1:ACCOUNT_ID:parameter/mole/FCM_SERVICE_ACCOUNT" }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/mole-backend",
          "awslogs-region": "ap-south-1",
          "awslogs-stream-prefix": "backend"
        }
      }
    }
  ]
}
```
```bash
aws ecs register-task-definition --cli-input-json file://backend-taskdef.json --region $AWS_REGION
```

### 7.3 Run migrations as a ONE-OFF task (before the service)
```bash
aws ecs run-task --cluster mole --launch-type FARGATE \
  --task-definition mole-backend \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx],securityGroups=[sg-backend],assignPublicIp=ENABLED}" \
  --overrides '{"containerOverrides":[{"name":"backend","command":["node","backend/node_modules/prisma/build/index.js","migrate","deploy"]}]}' \
  --region $AWS_REGION
```
> Simpler alternative: keep a tiny `migrate` npm script and override the command to
> `["pnpm","--filter","@mole/backend","prisma","migrate","deploy"]` (pnpm is in the image via
> corepack). Run `seed:admin` the same one-off way to create the first admin.

### 7.4 ALB + target group + HTTPS
1. **ACM** (in `ap-south-1`, same region as the ALB): request a cert for `api.yourdomain.com`,
   DNS-validate.
2. **EC2 → Load Balancers → Create → Application Load Balancer** `mole-alb`, internet-facing, public
   subnets, SG `mole-alb-sg`.
3. **Target group** `mole-backend-tg`, target type **IP**, protocol HTTP **3000**, health-check path
   **`/health`** (the backend serves `GET /health` → `{"status":"ok"}`).
4. ALB **listeners**: HTTP:80 → redirect to HTTPS; HTTPS:443 → forward to `mole-backend-tg`, attach
   the ACM cert.

### 7.5 Create the service (desired count = 1)
```bash
aws ecs create-service --cluster mole --service-name mole-backend \
  --task-definition mole-backend --desired-count 1 --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx,subnet-yyy],securityGroups=[sg-backend],assignPublicIp=ENABLED}" \
  --load-balancers "targetGroupArn=arn:...:targetgroup/mole-backend-tg/...,containerName=backend,containerPort=3000" \
  --region $AWS_REGION
```
> **desired-count = 1 is not a placeholder — it's a correctness requirement** (cron lock). Do not add
> service autoscaling to the backend.

### 7.6 DNS
Route 53: **A/ALIAS** `api.yourdomain.com` → the ALB.

---

## Step 8 — Web + admin

### Web (Next.js SSR) — pick one:
**Option A — Amplify Hosting (simplest).**
1. Amplify → **Host web app** → connect the GitHub repo, branch `main`.
2. Set the app root to `web/` (monorepo). Build settings: `pnpm install && pnpm --filter @mole/web build`.
3. Env var **`NEXT_PUBLIC_API_URL=https://api.yourdomain.com`** (compile-time).
4. Add custom domain `app.yourdomain.com` (Amplify provisions the cert).

**Option B — ECS Fargate (a second service).** Build a `web/Dockerfile` (`next build` then
`next start -p 3001`), push to ECR, register a `mole-web` task def with `NEXT_PUBLIC_API_URL` baked at
**build** time (it's compile-time — pass it as a Docker build-arg, not a runtime env), add a
`mole-web-tg` on 3001, a second ALB listener rule for host `app.yourdomain.com`. More work than
Amplify; only do it to keep everything in ECS.

### Admin (Vite static) — S3 + CloudFront:
```bash
cd admin && pnpm build          # root base; served at admin.yourdomain.com
aws s3 mb s3://mole-admin-prod --region $AWS_REGION
aws s3 sync dist/ s3://mole-admin-prod --delete
```
- **ACM cert in us-east-1** for `admin.yourdomain.com` (CloudFront requirement).
- CloudFront distribution: origin = `mole-admin-prod` (OAC, bucket private), default root object
  `index.html`, **SPA error mapping**: 403/404 → `/index.html` (200) so client routing works.
- Route 53 CNAME `admin.yourdomain.com` → the distribution.

---

## Step 9 — Zoom Video SDK (video + recording)

Identical to `aws-ec2-runbook.md` Step 8. Key point for ECS: the recording webhook target is the
**ALB** URL:
```
https://api.yourdomain.com/api/v1/recordings/zoom-webhook
```
Set `ZOOM_SDK_KEY` / `ZOOM_SDK_SECRET` / `ZOOM_ACCOUNT_ID` in SSM (Step 6), enable cloud recording,
subscribe to the recording-completed event. Runtime flow: session ends → Zoom webhook → ALB → backend
task downloads → S3 `recordings/{bookingId}/…` → presigned playback. Lifecycle rule (Step 3.4)
expires the object.

---

## Step 10 — CloudFront for media (optional at launch)

Same as `aws-ec2-runbook.md` Step 9 — front `mole-media-prod` with CloudFront + OAC for **public**
assets, set `CDN_BASE_URL` in the task env, redeploy the service. Recordings stay on the private
presigned path. You can launch without this.

---

## Step 11 — Mobile app

`EXPO_PUBLIC_API_URL=https://api.yourdomain.com`, then EAS build/submit — see
`docs/deployment/aws-deploy.md` → "Releasing the Android app".

---

## Step 12 — Smoke test

Same checklist as `aws-ec2-runbook.md` Step 11: web TLS, admin login, OTP send+verify+throttle,
upload→S3, booking→payment, Zoom video, recording→S3→playback, push, and **cron ticks** — check
`aws logs tail /ecs/mole-backend --follow` for `token-cleanup` etc. (proves the single task is running
crons).

---

## Step 13 — Redeploy flow (CI-friendly)

```bash
# build + push a new image
docker build -f backend/Dockerfile -t $ECR/mole-backend:latest . && docker push $ECR/mole-backend:latest
# run migrations one-off (if schema changed)
aws ecs run-task ... --overrides '{"containerOverrides":[{"name":"backend","command":["pnpm","--filter","@mole/backend","prisma","migrate","deploy"]}]}'
# roll the service to the new image
aws ecs update-service --cluster mole --service mole-backend --force-new-deployment --region $AWS_REGION
```
Web: Amplify auto-builds on `git push`. Admin: re-`sync` to S3 + CloudFront invalidation
(`aws cloudfront create-invalidation --distribution-id XXX --paths '/*'`).

---

## Observability & alerting

Same backend building blocks as the EC2 runbook — `GET /health`, pino JSON logs with request-ids +
secret redaction, and Sentry (redacts `phone`, captured in the error handler when `SENTRY_DSN` is set).
ECS-specific wiring:

- **Logs:** already shipping — the task def uses the `awslogs` driver → CloudWatch group `/ecs/mole-backend`.
  Query with **Logs Insights** by `x-request-id` / status. The **execution role** needs CloudWatch
  Logs write (covered by `AmazonECSTaskExecutionRolePolicy`).
- **Container metrics:** enable **CloudWatch Container Insights** on the `mole` cluster
  (`aws ecs update-cluster-settings --cluster mole --settings name=containerInsights,value=enabled`)
  for per-task CPU/memory.
- **Errors:** set `SENTRY_DSN` as an SSM secret (Step 6) and add it to the task def `secrets`.
- **Alarms → SNS → your phone:** create an SNS topic `mole-alerts`, then CloudWatch alarms on
  **ALB `HTTPCode_Target_5XX_Count`** (spike), **`UnHealthyHostCount`** (≥1), ECS **service CPU/memory**,
  RDS **CPU / FreeStorageSpace / DatabaseConnections**, and the recordings bucket **`BucketSizeBytes`**
  (lifecycle-broke early warning).
- **Uptime:** external monitor (UptimeRobot / **Uptime Kuma** self-host) hitting
  `https://api.yourdomain.com/health`.
- **Cost:** AWS Budgets alert + watch the **Zoom** billing dashboard — Video minutes + S3 recordings
  dwarf the compute bill.

**Free & open-source options** (errors → GlitchTip; logs → Loki; metrics/dashboards → Prometheus +
Grafana; uptime → Uptime Kuma; all-in-one → SigNoz) are tabulated in `aws-ec2-runbook.md` → Step 12.6 —
they apply identically here.

### Self-hosted observability box (separate EC2, 7-day retention) — ECS specifics

Stand up the **exact same obs box** as `aws-ec2-runbook.md` **Step 12.7** (Loki + Grafana +
Uptime Kuma via docker-compose, 7-day Loki retention, `mole-obs-sg`). Keep it **same VPC + same AZ**
as your Fargate tasks so shipping is over the **private IP and free**. Only the app-side log shipping
differs — Fargate has **no host to run promtail on**, so use one of:

**Option A — keep CloudWatch, don't self-host logs (simplest).** The task def already uses the
`awslogs` driver → `/ecs/mole-backend`. Query in CloudWatch Logs Insights. Add only **Uptime Kuma**
(uptime) + Grafana with a CloudWatch datasource. Least moving parts; you pay CloudWatch ingestion
(~$0.50/GB).

**Option B — ship task logs to Loki via a FireLens (Fluent Bit) sidecar.** Replaces the `awslogs`
driver on the backend container with `awsfirelens`, and adds a Fluent Bit sidecar that forwards to Loki
on the obs box's private IP. In the task definition:
```jsonc
// backend container: swap logConfiguration
"logConfiguration": {
  "logDriver": "awsfirelens",
  "options": {
    "Name": "loki",
    "Url": "http://OBS_PRIVATE_IP:3100/loki/api/v1/push",
    "Labels": "{job=\"mole-backend\"}"
  }
},
// add a second container to the same task:
{
  "name": "log_router",
  "image": "grafana/fluent-bit-plugin-loki:latest",
  "essential": true,
  "firelensConfiguration": { "type": "fluentbit" },
  "memoryReservation": 64
}
```
The task's security group must be allowed **outbound** to `mole-obs-sg:3100` (and `mole-obs-sg` must
allow `3100` inbound from the task SG). Because it's same-AZ private, this transfer is free.

**Metrics on ECS:** simplest is **CloudWatch Container Insights**
(`aws ecs update-cluster-settings --cluster mole --settings name=containerInsights,value=enabled`).
Prometheus scraping Fargate tasks needs ECS service discovery (more setup) — only worth it once the
backend exposes a `/metrics` endpoint. **Uptime Kuma** monitors `https://api.yourdomain.com/health`
exactly as on EC2.

> **Recommendation:** for ECS, **Option A + Uptime Kuma** is the least-effort path (keep CloudWatch for
> logs, self-host only uptime). Go to **Option B** (FireLens → Loki) when CloudWatch ingestion cost at
> ~2 GB/day starts to matter — the self-hosted box then undercuts it, with 7-day retention capping disk.

## Cost note vs EC2

ECS adds an **ALB (~$16–20/mo)** + Fargate vCPU/GB-hours + (if Amplify) its build/hosting + CloudFront.
Expect **~$40–70/mo more than the EC2 runbook** for the same traffic — you're buying rolling deploys,
task isolation, and easy horizontal scale of the **web** tier (never the backend). For ~500
sessions/day the EC2 runbook is cheaper and sufficient; move here when ops/scale demand it.

## Gotchas specific to ECS

| Symptom | Cause / fix |
|---|---|
| Duplicate notifications / double payouts | backend desired-count > 1 — **the crons double-fired**. Set it back to 1. |
| Migration race on deploy | you put `migrate deploy` in the container `CMD` — remove it; run as a one-off task. |
| Web calls hit localhost / wrong API | `NEXT_PUBLIC_API_URL` is **compile-time**; for the ECS/Docker web it must be a **build-arg**, not a runtime env. Rebuild the image. |
| Prisma "openssl" / engine error | add `openssl` to both Docker stages (done in the Step 4 Dockerfile). |
| Task can't read secrets | execution role missing `ssm:GetParameters` on `/mole/*`. |
| Upload 403 | S3 policy is on the **task role**, not execution role. |
| ALB health check flapping | health-check path must be **`/health`** (public, returns 200); adjust threshold if cold starts are slow. |
