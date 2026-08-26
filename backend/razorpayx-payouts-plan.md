# RazorpayX Payouts — integration plan

Automate educator payouts using the **RazorpayX Payouts API** so an **admin
click actually moves money** to the educator's bank account, replacing today's
manual "type the UTR by hand" settlement.

**Scope (locked):** admin-triggered payouts **only** — no cron / auto-payout.
An admin releases a payout; money goes to the educator's account; status
reconciles via webhook.

---

## 1. Current state (what exists)

- `Educator.pendingPayout` (Float) accrues per completed booking
  (`services/payout/accrue.ts`, guarded once by `Booking.payoutAccrued`).
- `PayoutHistory` rows record settlements. Enum `PayoutStatus = pending | paid`.
- **No money moves today.** `admin.controller.settlePayout` / `triggerPayout`
  just set `paid` + decrement `pendingPayout`, with `reference` = a UTR the admin
  types manually.
- Educator **already stores bank details**: `bankAccountName`,
  `bankAccountNumber`, `bankIfscCode`, `bankAccountType`. No UPI/VPA field, no
  stored Razorpay ids.
- The payment gateway uses the `razorpay` SDK with `RAZORPAY_KEY_ID/SECRET`
  (orders only — a **different product** from RazorpayX).

---

## 2. RazorpayX model

RazorpayX is separate from the PG. Same Basic-auth keys, **but the account needs
RazorpayX activation + a funded virtual/current account** (`account_number`).
There is a RazorpayX **test mode** with test balance for development.

Three entities, created bottom-up and then reused:

1. **Contact** — the educator (name/email/phone/type). Create once →
   store `razorpayContactId`.
2. **Fund Account** — a payout destination on a contact.
   Bank: `account_type=bank_account` + `{name, ifsc, account_number}`.
   Create once → store `razorpayFundAccountId`.
3. **Payout** — the transfer:
   `{account_number:<RazorpayX acct>, fund_account_id, amount:<paise>, currency:INR,
   mode:IMPS, purpose:payout, queue_if_low_balance:true, reference_id, narration}`
   with an **`X-Payout-Idempotency`** header.

**Status is asynchronous** (webhook-driven):
`queued → processing → processed` (success, returns the **UTR**) or
`reversed / failed / rejected`. So a payout **cannot** be marked paid
synchronously — the trigger fires it; a webhook confirms it.

---

## 3. Design

### 3.1 Schema (`schema.prisma` + migration)

```prisma
model Educator {
  // …
  razorpayContactId     String? @map("razorpay_contact_id")
  razorpayFundAccountId String? @map("razorpay_fund_account_id")
}

model PayoutHistory {
  // …
  providerPayoutId String? @map("provider_payout_id") // RazorpayX payout id (pout_…)
}

enum PayoutStatus {
  pending      // queued, money not yet moved (today's manual queue)
  processing   // RazorpayX payout created, awaiting webhook  ← new
  paid         // webhook: processed (UTR stored in `reference`)
  failed       // webhook: failed/reversed/rejected           ← new
  reversed     // money returned after being paid             ← new
}
```

### 3.2 Config (`lib/config.ts` + `.env.example`)

```
PAYOUT_PROVIDER=razorpayx|manual   # default 'manual' = today's typed-UTR behaviour
RAZORPAYX_ACCOUNT_NUMBER=          # the funded RazorpayX virtual/current account
RAZORPAYX_WEBHOOK_SECRET=          # HMAC secret for payout webhooks
# Auth reuses RAZORPAY_KEY_ID / RAZORPAY_KEY_SECRET.
```

`manual` mode keeps the exact current flow, so the feature ships dark and is
flipped on only once RazorpayX is activated + funded.

### 3.3 Service — `services/payout/razorpayx.ts`

Thin **axios** client (the `razorpay` SDK's payout coverage is inconsistent;
Contacts/FundAccounts/Payouts are cleaner over raw REST with Basic auth).

- `ensureFundAccount(educator)` → if `razorpayFundAccountId` unset, create the
  Contact (if needed) + Fund Account from the stored bank details, persist both
  ids, return the fund-account id. Idempotent. Errors if bank details missing.
- `createPayout({ fundAccountId, amountRupees, referenceId, narration, idempotencyKey })`
  → `POST /v1/payouts` (amount ×100 → paise), returns `{ payoutId, status }`.

### 3.4 Admin trigger flow (rewrite of the settle path)

When `PAYOUT_PROVIDER=razorpayx`, the admin action:

1. Validate `amount ≤ pendingPayout` (as today) **and** bank details present.
2. `ensureFundAccount(educator)`.
3. Create/claim a `PayoutHistory` row → status **`processing`**,
   `providerPayoutId` = the RazorpayX id, **idempotency key = PayoutHistory.id**
   (a retried click for the same row never double-pays).
4. **Do NOT decrement `pendingPayout` yet** — that waits for the webhook.

`manual` mode → unchanged (immediate `paid` + decrement + typed reference).

### 3.5 Webhook — `POST /api/v1/webhooks/razorpayx`

- Verify `X-Razorpay-Signature` = HMAC-SHA256(rawBody, `RAZORPAYX_WEBHOOK_SECRET`).
  Needs the raw body (mirror the LiveKit webhook's `express.text` route setup).
- `payout.processed` → row `paid`, store UTR in `reference`, **decrement
  `pendingPayout`** (guarded `updateMany where status=processing` so it fires
  once), notify the educator (FCM + in-app).
- `payout.failed` / `payout.rejected` → row `failed`, **no decrement**, alert
  admin.
- `payout.reversed` → row `reversed`, **re-credit `pendingPayout`** (money came
  back), alert admin.

### 3.6 Money-safety invariant

`pendingPayout` decrements **only on webhook confirmation of `processed`**, never
on the trigger click — preserving the existing rule *"money must never be marked
paid by anything that cannot prove the money moved."*

---

## 4. Gotchas

- **Activation is a hard prerequisite.** PG keys usually have no payout access;
  RazorpayX needs separate KYC/activation. Verify before building against live.
- **Idempotency is mandatory** — the payout create must carry a stable key
  (PayoutHistory id) so retries/webhook races don't double-pay.
- **Amounts in paise**, integer. `PayoutHistory.amount` is Int rupees → ×100.
- **Reversals land days later** — `pendingPayout` can go back up; the educator
  UI must handle a payout flipping from paid → reversed.
- **Low balance:** `queue_if_low_balance:true` queues instead of hard-failing,
  but then settlement waits on funding the RazorpayX account.
- **Mode/limits:** IMPS caps ~₹5L/txn; fine for tutoring amounts. NEFT/RTGS only
  needed for large sums.
- Raw-body route ordering: the JSON body parser must not consume the webhook
  route before signature verification (same care as the LiveKit/Razorpay-PG
  webhooks).

---

## 5. Build order (when approved)

1. Schema + migration + `generate:prisma`.
2. Config + `.env.example`.
3. `services/payout/razorpayx.ts` (ensureFundAccount + createPayout) + unit tests.
4. Rewrite admin trigger to `processing` (behind `PAYOUT_PROVIDER`).
5. Webhook route + controller + signature verify.
6. Educator earnings UI: surface `processing / failed / reversed` states (web +
   mobile, per parity).
7. Admin UI: show payout status + provider id; disable re-trigger while
   `processing`.

Branch: `feat/razorpayx-payouts` (separate PR from the LiveKit work).
