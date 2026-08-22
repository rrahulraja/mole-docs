# Integrations — End-to-End Test Runbook

Goal: exercise **SMS/OTP, WhatsApp, and Voice** against real providers with the
existing code (no rewrites). One booking lifecycle touches all three.

Vendor choice for this round:

| Channel  | Provider                        | Why                                                    |
|----------|---------------------------------|--------------------------------------------------------|
| SMS/OTP  | Twilio **trial**                | one vendor with voice; `TwilioOTPProvider` already coded |
| WhatsApp | Twilio **sandbox**              | same account as SMS/voice; free, no template approval for testing |
| Voice    | Twilio **trial**                | `voice-call.ts` already Twilio-wired, free credit       |

> SMS + voice share **one** Twilio credential set
> (`TWILIO_ACCOUNT_SID` / `TWILIO_AUTH_TOKEN` / `TWILIO_FROM_NUMBER`).

> Public URL required for inbound webhooks (WhatsApp, payment). Use
> `./deploy-local.sh start` to mint cloudflare tunnels, or run backend behind an
> existing tunnel. Note the backend's public base URL as `$PUBLIC_API`.

---

## 1. SMS / OTP — Twilio

`TwilioOTPProvider` generates/verifies the code via `otp-store` and sends the body
through Twilio — same account as voice (§3), so no extra signup.

### Env (`backend/.env`)
```
SMS_PROVIDER=twilio
TWILIO_ACCOUNT_SID=<AC...>
TWILIO_AUTH_TOKEN=<...>
TWILIO_FROM_NUMBER=<+1... SMS-capable Twilio number>
```

### Test
```
curl -X POST $PUBLIC_API/api/v1/auth/otp/send   -H 'Content-Type: application/json' -d '{"phone":"+9199XXXXXXXX"}'
# → real SMS arrives; read code
curl -X POST $PUBLIC_API/api/v1/auth/otp/verify -H 'Content-Type: application/json' -d '{"phone":"+9199XXXXXXXX","code":"123456"}'
# → returns access + refresh tokens
```
Pass = SMS received + verify returns tokens.

**India caveat (important).** Twilio A2P SMS **to Indian numbers** requires DLT
entity + template + sender-ID registration *through Twilio* — heavier than a US/
intl number. On **trial** you can only send to **verified** numbers and the body
is prefixed with a trial notice. So:
- Testing to a **verified non-Indian** number: works immediately.
- Testing to an **Indian** number: may be blocked until Twilio DLT/sender-ID
  registration clears. If it fails, this is why — and it's the reason MSG91 was
  the cheaper/easier India path. Fall back to `SMS_PROVIDER=mock` (OTP logged) to
  unblock the rest of the flow while Twilio India registration is pending.

---

## 2. WhatsApp — Twilio (sandbox)

Code sends via Twilio (`services/whatsapp.ts`), reusing the Twilio account from §1.
The **WhatsApp sandbox** sends + receives real WhatsApp messages, free, with no
template approval — each tester joins once by sending a join code.

### Account
1. Twilio Console → **Messaging → Try it out → Send a WhatsApp message** → the
   **sandbox** shows a number (`whatsapp:+14155238886`) + a join code.
2. From each tester's phone, WhatsApp `join <code>` to that number (valid ~72h).
3. No token to create — WhatsApp sends sign with the existing `TWILIO_AUTH_TOKEN`.

### Env (`backend/.env`)
```
# TWILIO_ACCOUNT_SID / TWILIO_AUTH_TOKEN already set in §1
TWILIO_WHATSAPP_FROM=whatsapp:+14155238886
```

### Inbound webhook (for educator accept/decline replies)
Twilio Console → Messaging → sandbox settings → **When a message comes in**:
- URL: `$PUBLIC_API/api/v1/webhooks/whatsapp` (**POST**)
- No verify token — Twilio signs each request; the backend checks
  `X-Twilio-Signature` best-effort.

### Test
Outbound is triggered by the booking flow (below). Inbound: reply **YES**/**NO** to
a session notification and confirm `handleWhatsAppWebhook` reacts.

---

## 3. Voice — Twilio (trial)

`voice-call.ts` already calls Twilio. Trial calls only **verified** numbers and
prepends a trial notice — fine for testing.

### Account
1. Sign up at https://twilio.com/try-twilio (free trial credit).
2. Get a Twilio phone number (voice-capable).
3. Verify your own phone under **Verified Caller IDs**.

### Env (`backend/.env`)
```
TWILIO_ACCOUNT_SID=<AC...>
TWILIO_AUTH_TOKEN=<...>
TWILIO_FROM_NUMBER=<+1... twilio number>
```

### Enable the admin gate
Voice is off by default behind PlatformConfig `voice_calls_enabled`. Turn it on in
the admin Config page, or:
```
UPDATE platform_config SET value='true' WHERE key='voice_calls_enabled';
-- insert if absent:
-- INSERT INTO platform_config(key,value) VALUES('voice_calls_enabled','true');
```

---

## 3b. Payments — Razorpay (test mode)

Backend `RazorpayProvider` creates orders; booking is confirmed by the
**`payment.captured` webhook** (not the client). Web checkout modal wired
(`web/app/student/payment/page.tsx`). Mobile still mock (native module +
custom Expo build required — parity gap, deferred).

### Env (`backend/.env`)
```
PAYMENT_PROVIDER=razorpay
RAZORPAY_KEY_ID=rzp_test_...
RAZORPAY_KEY_SECRET=...        # Dashboard(Test) > Settings > API Keys (shown once)
RAZORPAY_WEBHOOK_SECRET=...    # the secret you set when creating the webhook
```

### Webhook (Dashboard > Settings > Webhooks)
- URL: `$PUBLIC_API/api/v1/payment/webhook/razorpay`
- Secret: any string; copy into `RAZORPAY_WEBHOOK_SECRET`
- Active events: **`payment.captured`** + **`payment.failed`**
- Public URL required — mint via `./deploy-local.sh start`.

### Test
1. Book a session as student → reach `pending_payment`.
2. Payment page → **Pay with Razorpay** → checkout modal opens.
3. Test success: card `4111 1111 1111 1111`, any future expiry/CVV, or UPI
   `success@razorpay`. Test failure: UPI `failure@razorpay`.
4. Razorpay fires `payment.captured` to the webhook → `finalizeSuccessfulPayment`
   → booking `confirmed`. Page poll (`GET /payment/:id/status`, 3s) navigates to
   `booking-confirmed`.

Pass = modal completes + webhook logs `Processed` + booking `confirmed`.

Watch: backend logs for `[razorpay]`; unmatched order logs `Unknown order_id`;
bad signature logs `Invalid webhook signature` (401 — check `RAZORPAY_WEBHOOK_SECRET`).
No tunnel = webhook never fires; booking stays `pending_payment` (use the
**Simulate Payment Success** button as a local fallback).

---

## 4. One-flow E2E (touches all three)

1. **Login** as student → MSG91 OTP → verify.               [SMS]
2. **Create instant booking** → educator gets WhatsApp;
   if unaccepted, a voice reminder is placed.               [WhatsApp + Voice]
3. **Educator replies "accept"** on WhatsApp → inbound webhook. [WhatsApp inbound]
4. **Pay** → booking confirmed → WhatsApp confirm to educator. [WhatsApp]
5. **Let the accept window lapse** → `educator-delay-check`
   cron places a voice reminder call.                        [Voice]

Watch backend logs for `[voice-call] Call initiated`, `[WhatsApp]` posts, and the
MSG91 200s. Each channel's failure logs a clear reason (creds not set / gate off).

---

## Notes for production (later)
- SMS: complete Twilio **India DLT + sender-ID** registration before sending to
  Indian numbers at scale (or reconsider MSG91 for India cost/deliverability).
- WhatsApp: move off the test number to a real WABA + **approved templates** (a
  business-initiated message outside the 24h window needs a template).
- Voice: upgrade Twilio out of trial (removes verified-number + trial-notice
  limits), or revisit consolidating voice onto MSG91.
- Keep `SMS_PROVIDER=mock` for local/CI so no credits burn on automated runs.
