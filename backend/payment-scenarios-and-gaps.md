# Payments — scenario audit & gaps

An end-to-end review of every money-touching path (student payment → refund →
educator payout) against what the code handles **today**, the scenarios it does
**not**, and the design to close them.

**Headline:** the current system is **fragile on failure and cannot refund to
source.** Confirmation depends on a single webhook with no fallback, and every
"refund" only credits the in-app wallet — there is no path to return real money
to a student's bank/card. Several paths also refund the **wrong amount**.

Severity: 🔴 money-loss / correctness · 🟠 stuck state / manual toil · 🟡 hardening.

---

## 1. Current flow (as built)

- **Intent:** `POST /payment/:bookingId/intent` upserts a `Payment` (status
  `pending`) and creates a Razorpay order; the order id is stored on
  `Payment.upiTransactionRef`. (`payment.controller.ts:createPaymentIntent`)
- **Confirmation:** **only** via the Razorpay webhook
  `POST /payment/webhook/razorpay` → `payment.captured` →
  `finalizeSuccessfulPayment` → booking `confirmed`, wallet coins debited, session
  provisioned. (`payment.controller.ts:razorpayWebhookHandler`,
  `services/payment/index.ts`)
- **Split payment:** a booking's price = `amountDue` (paid via gateway) +
  `creditsApplied` (paid via wallet coins).
- **Refunds:** wallet-credit only, in three places, all inconsistent (§3).
- **Payout:** educator `pendingPayout` accrues on completion; admin settles
  manually (see `educator-payouts-design.md`).

---

## 2. Money-IN scenarios (student → us)

| # | Scenario | Handled today? | Sev |
|---|----------|----------------|-----|
| 2.1 | Captured + webhook delivered | ✅ `finalizeSuccessfulPayment` (idempotent) | — |
| 2.2 | **Captured but webhook never arrives / delayed** | 🔴 **No.** Booking stays `pending_payment`; **no poll/reconcile** against Razorpay; after 2h the stale cron **cancels it with "no refund needed"** → student charged, booking gone, money kept silently | 🔴 |
| 2.3 | **Webhook secret unset/misconfigured** | 🔴 `verifyWebhookSignature` returns false on blank secret (`RazorpayProvider.ts:48`) → 401 → **nothing ever confirms.** Secret is `optional` in config and was blank in prod per notes | 🔴 |
| 2.4 | Duplicate webhook | ✅ guarded by `Payment.status` already `success/failed` | — |
| 2.5 | `payment.failed` event | 🟠 booking → `failed`; student must rebook (no retry surfaced from that state) | 🟠 |
| 2.6 | **Out-of-order events** (`failed` then late `captured`) | 🔴 the `failed` guard returns "Already processed", so a **later real capture is ignored** → money captured, booking `failed` | 🔴 |
| 2.7 | **Amount mismatch** (captured ≠ expected) | 🔴 **Not checked** — webhook never compares `entity.amount` to `Payment.amount×100` | 🔴 |
| 2.8 | **Double charge** (retry creates a new order, both captured) | 🔴 `retryPayment` mints a fresh order; if the first also captures, two successful payments, **no dedupe/auto-refund** | 🔴 |
| 2.9 | Extension payment (no `Payment` row) | ✅ settles against `BookingExtension` (idempotent `applyExtension`) | — |
| 2.10 | No `WebhookEvent` store | 🟠 events aren't persisted → no audit, no replay, dedupe leans on row state only | 🟠 |

**Root cause for 2.2/2.3/2.6/2.7:** confirmation is a **single webhook with no
fallback and no reconciliation.** `getPaymentStatus` only reads our own DB — it
never asks Razorpay.

---

## 3. Money-OUT / refund scenarios (us → student)

There is **no refund-to-source anywhere.** No `Refund` model, no
`PaymentProvider.refund()`. Every "refund" increments the wallet balance. The
three call sites also disagree on the amount:

| # | Scenario | Code | What it refunds | Gap | Sev |
|---|----------|------|-----------------|-----|-----|
| 3.1 | Student cancels >1h before | `booking.controller.ts:248` | `amountDue` → **credits** | ignores coins (`creditsApplied`); credits-only, not source | 🟠 |
| 3.2 | **Reassignment give-up** (no educator, admin escalates) | `reassignment/index.ts:197` | **only `creditsApplied`** (the coins) | 🔴 **loses the gateway money (`amountDue`) entirely** — student paid real ₹, gets back only the coin part, as credits | 🔴 |
| 3.3 | Dispute resolved with refund | `admin.controller.ts:775` | `refundAmount` → credits, **only if `refundType === 'credits'`** | any other `refundType` (e.g. source) **silently does nothing** | 🔴 |
| 3.4 | **Student demands money back to bank, not credits** | — | — | 🔴 **no code path exists** (the reported scenario) | 🔴 |
| 3.5 | Mixed coins+gateway refund | — | — | no path splits coins→wallet, gateway→source | 🔴 |
| 3.6 | **Refund after educator payout accrued** (session completed, later disputed) | — | — | 🔴 no clawback from educator `pendingPayout`/ledger → platform eats it | 🔴 |
| 3.7 | Refund idempotency / failure / status | — | — | no refund tracking, no `refund.processed`/`failed` webhook handling | 🔴 |

**The reported end-to-end case** ("paid, webhook missed, no educator, student
wants a real refund") hits **2.2 → 3.2 → 3.4** — three separate holes in a row:
the payment may be stuck-then-silently-cancelled, and even when caught, the
give-up path refunds the wrong amount to the wrong place, and a source refund
is impossible.

---

## 4. Design to close the gaps

### 4.1 Refund as a first-class entity

```prisma
model Refund {
  id                String       @id @default(uuid())
  bookingId         String       @map("booking_id")
  paymentId         String?      @map("payment_id")
  // split
  amountToSource    BigInt       @default(0) @map("amount_to_source")  // paise → gateway refund
  amountToCredits   BigInt       @default(0) @map("amount_to_credits") // paise → wallet coins
  currency          String       @default("INR")
  reason            RefundReason                                        // cancellation | reassignment_failed | dispute | duplicate | admin
  status            RefundStatus @default(created)                     // created|processing|processed|failed
  providerRefundId  String?      @unique @map("provider_refund_id")    // rfnd_…
  idempotencyKey    String       @unique @map("idempotency_key")
  initiatedByAdminId String?     @map("initiated_by_admin_id")
  failureReason     String?
  createdAt, updatedAt, processedAt
}
```

`PaymentProvider.refund({ paymentId, amountPaise, speed, idempotencyKey })` →
Razorpay `POST /v1/payments/:id/refund` (partial supported; `speed=optimum` for
instant where eligible, else `normal` = 5–7 days). Handle `refund.processed` /
`refund.failed` webhooks to move `Refund.status` and notify the student. Source
refunds are **async** — the wallet is *not* the confirmation.

### 4.2 One refund service, correct amounts

`refundBooking(bookingId, { channel: 'source'|'credits', amount?, reason, admin? })`
— the single entry point that **replaces** the three ad-hoc blocks. It:

- Refunds the **full** picture by default: `creditsApplied` → wallet (instant),
  `amountDue` → the chosen channel. Partial amounts supported for disputes.
- `channel:'credits'` → instant wallet credit (today's behaviour, but complete).
- `channel:'source'` → create a `Refund`, call Razorpay, confirm on webhook.
- **Clawback (3.6):** if the educator payout already accrued for this booking,
  post a compensating entry against the educator ledger (see payout design) so
  the refund doesn't come out of the platform silently.
- Idempotent per `(bookingId, reason)`.

**Policy:** student-initiated "I want my money back" → `source`. Goodwill / fast
resolution / promo → `credits`. Admin picks in the dispute UI; default for a
failed reassignment where the student didn't consent to credits = **source**.

### 4.3 Fix confirmation robustness (money-IN)

- **Client verify fallback:** on Razorpay checkout success the client already has
  `razorpay_payment_id/order_id/signature` — add `POST /payment/:bookingId/verify`
  that validates the signature and calls `finalizeSuccessfulPayment`. Confirms
  instantly even if the webhook is slow, without trusting the client (signature
  is the proof). Webhook stays the backstop.
- **Payment reconciliation cron:** for every `Payment` still `pending` past N
  minutes, fetch the order/payments from Razorpay; if captured →
  `finalizeSuccessfulPayment`; if failed → mark failed. Closes 2.2/2.6.
- **Stale-cron guard:** `stale-booking-expiry` must **never cancel a
  `pending_payment` without first checking Razorpay** for a capture — otherwise
  it cancels paid bookings (2.2). If captured, confirm (or refund), don't cancel.
- **Amount check (2.7):** in the webhook/verify, assert
  `entity.amount === Payment.amount × 100` before finalizing; mismatch → hold +
  alert, don't confirm.
- **Webhook secret (2.3):** make `RAZORPAY_WEBHOOK_SECRET` effectively required
  in production (boot-time check + alert), and alert on any 401 from the webhook.

### 4.4 Shared webhook + reconciliation infra

- **`WebhookEvent`** table (provider, providerEventId, payload, status) shared by
  PG + payouts: persist-first, dedupe on `providerEventId`, process once, tolerate
  replays/out-of-order. Gives audit + replay for 2.10.
- **Money-in reconciliation (daily):** pull Razorpay settlements/transactions and
  diff against our `Payment`/`Refund` rows — catch captures we missed, refunds we
  didn't record, amount drift. Mirror of the payout L2/L3 recon.

---

## 5. Priority

1. 🔴 **Stop silent loss:** stale-cron guard + payment reconciliation cron + client
   verify fallback (2.2/2.3/2.6).
2. 🔴 **Fix reassignment refund amount** (3.2) — refund `amountDue` too, not just
   coins.
3. 🔴 **Refund-to-source** (`Refund` + `provider.refund()` + refund webhooks) and
   the unified `refundBooking` service (3.1–3.5).
4. 🔴 **Payout clawback on post-accrual refund** (3.6) — needs the educator ledger.
5. 🟠 amount validation, WebhookEvent store, dispute refundType, double-charge
   dedupe, out-of-order handling.

Sequencing note: items 3–4 ride on the same ledger/`WebhookEvent`/state-machine
foundation as `educator-payouts-design.md` — build that base once and both the
payout and refund flows plug into it.

---

## 6. Open questions

1. **Refund default channel** for a failed reassignment where the student is
   silent — source (real money) or credits? (Design assumes source.)
2. **Instant-refund eligibility / cost** — Razorpay `speed=optimum` has fees and
   method limits; acceptable, or always `normal`?
3. **Partial-delivery refunds** (session dropped mid-way, or an extension) — pro-rata
   policy?
4. **Clawback vs platform-absorb** when a completed session is refunded after the
   educator was already paid out — recover from the educator, or eat it?
5. Is `RAZORPAY_WEBHOOK_SECRET` set in production **right now**? (If not, 2.3 means
   confirmations are silently failing.)
