# Mole — third-party integrations setup guide

Every external account Mole needs, with **account creation → credentials → env keys → configuration
→ testing**, plus which ones are a clean personal→company email move vs. tied to a legal entity.

Env-var names below are the **real ones the code reads** (`backend/.env`). Auth across all apps is
**phone + SMS OTP** — there is **no email integration** (admins included).

**Legend for "move to company":**
✅ easy — just change the account email later · ⚠️ entity-bound — tied to bank/PAN/business
verification/developer identity; register under the company or expect re-KYC.

| # | Service | Purpose | Launch-critical | Move |
|---|---|---|---|---|
| 1 | AWS | Hosting, S3, CloudFront | ✅ yes | ✅ |
| 2 | Domain / Route 53 | DNS + TLS | ✅ yes | ✅ |
| 3 | Twilio | SMS OTP + voice reminder calls | ✅ yes | ✅ |
| 4 | Firebase / FCM | Push notifications | ✅ yes | ✅ |
| 5 | Zoom Video SDK | Video sessions + cloud recording | ✅ yes | ✅ |
| 6 | Razorpay | Payments | ✅ yes | ⚠️ |
| 7 | UPI VPA | Direct UPI collect | ⚠️ only if extensions (else Razorpay covers UPI) | ⚠️ |
| 8 | WhatsApp (Twilio) | Booking accept/decline messages | ✅ yes | ✅ |
| 9 | Google Play Console | Ship Android app | ✅ yes | ⚠️ |
| 10 | Expo / EAS | Build the mobile app | ✅ yes | ✅ |
| 11 | Sentry | Error monitoring | optional | ✅ |
| 12 | MSG91 | Alt SMS (India bulk) | optional | ⚠️ |

> **Recommended split:** register **1–5, 8, 10, 11 now on personal email** (they move cleanly; WhatsApp
> rides the Twilio account from §3). Hold **6, 7, 9** (entity-bound) until the company + bank + PAN
> exist, and register those directly under the company — use their **sandbox/test modes** to build
> against in the meantime.

---

## 1. AWS

**Purpose:** EC2/ECS hosting, RDS Postgres, S3 (uploads + Zoom recordings), CloudFront.

**Create:** `aws.amazon.com` → create account (personal email + card). Enable MFA on root. Create an
IAM admin user for daily use; an IAM app user/role for S3 (least privilege).

**Env keys (backend `.env`):**
```dotenv
STORAGE_PROVIDER=s3
AWS_REGION=ap-south-1
AWS_S3_BUCKET=mole-media-prod
AWS_S3_KEY_PREFIX=            # optional
AWS_S3_ENDPOINT=             # BLANK for real AWS S3 (only set for R2/MinIO)
AWS_ACCESS_KEY_ID=...        # omit if using an EC2/ECS instance role
AWS_SECRET_ACCESS_KEY=...
CDN_BASE_URL=                # CloudFront domain, once set up
```

**Configure:** full steps in `aws-ec2-runbook.md` / `aws-ecs-runbook.md` (bucket, CORS, IAM policy,
**lifecycle rule for `recordings/`**, RDS, CloudFront).

**Startup credits:** apply for **AWS Activate** once you have a company domain email (~$1,000 Founders,
or $5k–100k via a provider). See the billing note in `aws-ec2-runbook.md`.

---

## 2. Domain + Route 53

**Purpose:** `api.` / `app.` / `admin.` hostnames + TLS.

**Create:** buy a domain (Route 53 or any registrar). If not Route 53, you can still create a Route 53
**hosted zone** and point the registrar's nameservers at it, or just add A-records at your registrar.

**Configure:** A-records → your Elastic IP (EC2) or ALIAS → ALB/CloudFront (ECS). TLS via certbot
(EC2) or ACM (ECS/CloudFront). Full steps in the runbooks.

No env keys — but the hostnames feed `API_BASE_URL`, `ADMIN_APP_URL`, `NEXT_PUBLIC_API_URL`,
`EXPO_PUBLIC_API_URL`.

---

## 3. Twilio — SMS OTP + voice reminder calls

**Purpose:** the OTP SMS for **all** logins (students, educators, **admins**) + voice reminder calls.

**Create:**
1. `twilio.com` → sign up, verify your email + phone.
2. Upgrade to a **paid** account (trial can only message verified numbers).
3. **Buy a phone number** with **SMS + Voice** capability:
   - Console → Phone Numbers → Buy a number → filter **SMS + Voice**.
   - ⚠️ **Gotcha (already hit):** US number purchase can fail with
     `20003 Primary compliance profile not approved` — complete **Trust Hub / regulatory bundle
     (KYC)** first, then the purchase succeeds.

**Credentials → env keys (`backend/.env`):**
```dotenv
SMS_PROVIDER=twilio
TWILIO_ACCOUNT_SID=AC...      # Console dashboard
TWILIO_AUTH_TOKEN=...         # Console dashboard (rotate if ever shared)
TWILIO_FROM_NUMBER=+1...      # the number you bought (E.164)
```

**Test:** trigger OTP on any login → SMS arrives → verify. Voice: enable `voice_calls_enabled` in the
admin Config page (off by default).

**Move to company:** ✅ change account email; keep the same number/SID/token. **Rotate the auth token**
if it was ever pasted anywhere. Release a test number when done to stop the monthly charge.

---

## 4. Firebase / FCM — push notifications

**Purpose:** Android push (`sendPushWithRecord` sends FCM when the user has an `fcmToken`).

**Create:**
1. `console.firebase.google.com` → **Add project** (e.g. `mole-prod`).
2. **Add an Android app**, package name **`com.mole.app`** (must match `mobile/app.json`).
3. Download **`google-services.json`** → place at `mobile/google-services.json` (for CI, add as an EAS
   **file secret**, don't commit).
4. Project Settings → **Service accounts** → **Generate new private key** → the JSON the backend uses.

**Env keys (`backend/.env`):**
```dotenv
FCM_SERVICE_ACCOUNT=...       # the service-account JSON (per your config loader)
```

**Test:** on an Android **dev build** (not Expo Go), log in → trigger a notification → push arrives.

**Move to company:** ✅ transfer the Google Cloud project to the company org, or recreate. Package
`com.mole.app` is fixed — keep it. Prod swap = swap credentials only.

---

## 5. Zoom Video SDK — video sessions + cloud recording

**Purpose:** 1:1 video + cloud recording (auto-downloaded to your S3). This is **Video SDK** (metered
per-minute), **not** Meeting SDK (per-host license) — confirmed in `services/zoom`.

**Create:**
1. `marketplace.zoom.us` → **Develop → Build App → Video SDK** → get **SDK Key** + **SDK Secret**.
2. Build a **Server-to-Server OAuth** app too → get the **Account ID** (recording download uses the
   `account_credentials` grant). Scopes: **Video SDK** + **cloud recording** read/download.
3. Enable **Cloud Recording** in Zoom account settings.

**Env keys (`backend/.env`):**
```dotenv
ZOOM_SDK_KEY=...
ZOOM_SDK_SECRET=...
ZOOM_ACCOUNT_ID=...
```

**Configure the recording webhook:**
- In the app's **Event Subscriptions**, add endpoint:
  `https://api.yourdomain.com/api/v1/recordings/zoom-webhook`
- Subscribe to the **recording completed** event. Endpoint is **HMAC-verified** (uses raw body).
- Flow: session ends → Zoom finishes recording → webhook → backend downloads → S3
  `recordings/{bookingId}/*.mp4` → students play via presigned `GET /api/v1/recordings/:bookingId`.

**Test:** run a session end-to-end → recording object appears in S3 → playback works.

**Move to company:** ✅ email swap. **Cost watch:** Video minutes are your **largest** cloud line item —
apply to **Zoom for Startups / ISV** for credits.

**Gotcha:** the app expires recording **access** at 24h but does **not** delete the S3 object — set the
S3 **lifecycle expiry** rule (runbooks Step 4) or storage grows unbounded.

---

## 6. Razorpay — payments

**Purpose:** cards / UPI / netbanking checkout + payment webhooks.

**Create:**
1. `razorpay.com` → sign up.
2. **Start in Test Mode** immediately — Test keys work with no KYC, so you can build now.
3. For **live** payments: complete **business KYC** (PAN, business proof, **settlement bank account**).

**Env keys (`backend/.env`):**
```dotenv
PAYMENT_PROVIDER=razorpay
RAZORPAY_KEY_ID=rzp_test_... / rzp_live_...
RAZORPAY_KEY_SECRET=...
RAZORPAY_WEBHOOK_SECRET=...   # set this — leaving it blank means webhooks aren't verified
MERCHANT_VPA=...              # see #7
MERCHANT_NAME=Mole
```

**Configure webhook:** Razorpay Dashboard → Settings → **Webhooks** → add
`https://api.yourdomain.com/api/v1/payment/webhook/razorpay`, set the **secret** to match
`RAZORPAY_WEBHOOK_SECRET`, subscribe to payment events.

**Test:** Test Mode keys + Razorpay test cards → book → pay → booking `confirmed`.

**Move to company:** ⚠️ **entity-bound.** Test Mode is fine on personal signup, but **live** keys are
tied to the company's KYC + bank. Do the live account **as the company**; don't ship live payments on a
personal KYC you'll have to redo. **Gotcha (open item):** `RAZORPAY_WEBHOOK_SECRET` was blank in
testing — set it before going live or payment confirmations aren't verified.

---

## 7. UPI VPA (merchant) — usually NOT needed

**Do you need it?** **No, if you use Razorpay** — Razorpay accepts UPI (plus cards/netbanking)
natively, so normal booking checkout needs no merchant VPA.

`MERCHANT_VPA` / `MERCHANT_NAME` / `UPI_WEBHOOK_SECRET` are read only by:
- the **`mock` (legacy) payment provider** — a raw `upi://pay?pa=<VPA>` deep link confirmed by a
  hand-rolled HMAC webhook (fee-free, but **no auto-reconciliation** — fragile for prod), and
- the **session-extension flow** (`booking.controller`), which currently **hardcodes** that raw UPI
  intent and **bypasses `PAYMENT_PROVIDER`** — so if you enable extensions today, that one path still
  needs a VPA + the UPI webhook.

**Recommendation:** migrate the extension flow to the Razorpay provider abstraction (like
`createPaymentIntent`), then drop the VPA entirely. Until then, set a VPA only if you ship extensions.

**Move to company:** ⚠️ if you do use it, it's tied to the collecting bank/entity — company account.

---

## 8. WhatsApp via Twilio — booking accept/decline

**Purpose:** send educators a WhatsApp booking request with **Accept / Decline** quick-reply buttons.

> **Runs on the same Twilio account** already used for SMS OTP + voice (§1) — no Meta / Facebook
> account needed. Only the sender differs: a WhatsApp-enabled number, `whatsapp:`-prefixed.

**Create:**
1. **Twilio Console → Messaging → Try it out → WhatsApp** — join the **sandbox** for testing (you get
   `whatsapp:+14155238886` + a join code testers send once). Works immediately, no approval.
2. **Production:** register a **WhatsApp sender** (your own number) under Messaging → Senders →
   WhatsApp senders. Needs a Meta Business Manager link + WhatsApp approval (Twilio walks you through
   it), but the number + billing stay on Twilio.

**Env keys (`backend/.env`):**
```dotenv
# TWILIO_ACCOUNT_SID / TWILIO_AUTH_TOKEN are already set for SMS + voice (§1).
TWILIO_WHATSAPP_FROM=whatsapp:+14155238886   # sandbox; swap for your approved sender in prod
```

**Configure:**
1. **Inbound webhook:** Twilio sandbox / sender settings → "When a message comes in" →
   `https://api.yourdomain.com/api/v1/webhooks/whatsapp` (**POST**). No verify-token handshake — Twilio
   signs each request; the backend checks `X-Twilio-Signature` (best-effort) with your auth token.
2. **Business-initiated template (for messages outside the 24h window):** build the request template in
   the **Twilio Content Template Builder** (Utility category, two "Accept"/"Decline" quick-reply
   buttons), wait for WhatsApp approval, then set `contentSid` + `"enabled": true` in
   `templates/whatsapp/booking_request.json`. Until then the app falls back to a free-text body (works
   inside an open 24h window / the sandbox).

**Test:** join the sandbox from the educator's phone → create a live booking → educator gets the
WhatsApp message → reply **YES** (or tap Accept on the approved template) → booking moves to accepted.

> **Note vs Meta:** Twilio quick-reply button payloads are **static** (no per-message booking id), so a
> button tap resolves to the educator's **latest awaiting** request rather than a specific `ACCEPT_<id>`.
> Plain-text `YES`/`NO` works the same way.

**Move to company:** ⚠️ the WABA sits under a **Meta Business** that goes through business verification;
moving a number between businesses is painful. Do the production WABA **as the company**. Replace the
temporary token with a **permanent System-User token** before launch.

---

## 9. Google Play Console — Android release

**Purpose:** publish the app (`com.mole.app`).

**Create:** `play.google.com/console` → pay the **$25 one-time** fee. Choose **organization** account
type if you can (needs a **D-U-N-S** number) — the individual/organization choice is hard to change
later.

**Configure / submit:** build an **AAB** via EAS (`eas build --profile production`), then `eas submit`
or upload manually. First submission needs: privacy-policy URL, **Data safety** form (phone, location,
camera/mic, KYC docs), content rating, **test login for the reviewer** (the app is OTP-gated).
Full steps in `aws-deploy.md` → "Releasing the Android app".

**Move to company:** ⚠️ developer-identity-bound. Transferring an app between Play accounts is a formal
process — ideally create the Console account **as the company** from the start.

---

## 10. Expo / EAS — mobile builds

**Purpose:** cloud-build the APK/AAB, OTA updates.

**Create:** `expo.dev` → account → `npx eas login` → `eas build:configure`.

**Bake the API URL** (compile-time) in the production build profile:
```
EXPO_PUBLIC_API_URL=https://api.yourdomain.com
```

**Move to company:** ✅ transfer the project / change the account email.

---

## 11. Sentry (optional) — error monitoring

**Create:** `sentry.io` → project (Node for backend). Copy the DSN.
```dotenv
SENTRY_DSN=...
```
**Move:** ✅ email swap.

---

## 12. MSG91 (optional) — alternative SMS provider

**Purpose:** cheaper India bulk SMS as an alternative to Twilio (`SMS_PROVIDER=msg91`).

**Create:** `msg91.com` → account → complete **DLT** registration (Indian regulatory requirement for
bulk SMS) → approve a template.
```dotenv
SMS_PROVIDER=msg91
MSG91_AUTH_KEY=...
MSG91_TEMPLATE_ID=...
MSG91_SENDER_ID=...
MSG91_OTP_VAR=...
```
**Move:** ⚠️ DLT is business-registered — set up under the company. Only needed if Twilio SMS cost
becomes a factor at scale.

---

## Sequencing for a weekend launch

**Do now (personal email, they migrate cleanly):**
AWS · Domain · Twilio · Firebase/FCM · Zoom Video SDK · Expo/EAS · Sentry.

**Do as the company (entity-bound — avoid re-KYC), using sandbox until then:**
Razorpay (build on Test keys) · UPI VPA (only if you ship session extensions) · WhatsApp (Twilio sandbox) · Play Console
(use Internal testing).

**Open items to close before real launch:**
- Razorpay `RAZORPAY_WEBHOOK_SECRET` — currently blank; set + verify.
- WhatsApp — replace the temporary token with a **permanent System-User token**; confirm the
  `subscribed_apps` binding.
- Rotate any Twilio / WhatsApp tokens that were shared during testing.
- Release the temporary Twilio number once testing is done.
