# Educator Payouts — system design

A fintech-grade design for paying educators real money via **RazorpayX Payouts**,
admin-triggered, with a correct ledger, a payout state machine, and end-to-end
reconciliation.

**Guiding stance:** money systems fail closed. We would rather **not pay** than
**pay twice** or **pay the wrong person**. Every decision below trades a little
convenience for auditability and exactly-once behaviour.

**Scope:** admin-triggered payouts only (no cron auto-pay). Single currency (INR).
Destination = educator bank account. RazorpayX is the disbursement rail.

---

## 1. Principles (non-negotiable)

1. **The ledger is the source of truth.** Balances (`pendingPayout`) are a cached
   projection of an append-only ledger — never the other way round.
2. **Append-only, immutable.** Entries are never updated or deleted. Corrections
   are new, signed entries (reversals/adjustments).
3. **Integer minor units.** All money is **paise (`BigInt`)**. No floats, ever.
4. **Exactly-once via idempotency keys.** Every accrual, payout, and webhook
   effect carries a stable key; replays are no-ops.
5. **Explicit state machine.** A payout moves only along defined transitions;
   illegal or stale transitions are rejected, not applied.
6. **Authorize ≠ settle ≠ confirm.** Admin *authorizes*; the system *initiates*
   at RazorpayX; a *webhook* confirms. The balance moves only on confirmation.
7. **Reconciliation is a feature, not a script.** Internal and external recon run
   on a schedule, detect breaks, alert, and **never auto-move money** to "fix" a
   break — a human resolves.
8. **Least privilege + full audit.** Bank details are PII; every admin money
   action is attributable and immutable.

---

## 2. Current state & gaps

| Area | Today | Gap |
|------|-------|-----|
| Balance | `Educator.pendingPayout` **Float**, mutated in place | float money; no history |
| Accrual | `accrue.ts` does `increment: credit` | **no ledger row** — balance unreconstructable |
| Settlement | `PayoutHistory` (`pending\|paid`), admin types UTR; no money moves | no real disbursement, no provider link |
| Wallet | `WalletTransaction` ledger exists | good precedent — payouts lack the equivalent |
| Money type | `PayoutHistory.amount` Int (rupees), `pendingPayout` Float | inconsistent units |

The wallet is already ledgered; **the payout balance is not.** That is the core
gap this design closes — before any external money moves.

---

## 3. Domain model

### 3.1 Money movements (conceptual accounts)

Even with a single-sided educator ledger, name the buckets so reconciliation has
meaning:

- **Educator Payable** — what we owe each educator (Σ their ledger). A liability.
- **RazorpayX Float** — funds we hold at RazorpayX, drawn down by payouts.
- **Bank Settled** — money that has left the float to an educator's bank (UTR).

Invariant across a healthy system:
```
Σ processed payouts (− reversals)  ==  RazorpayX float debits  ==  bank transfers (UTRs)
Educator Payable (our books)       ==  Σ accruals − Σ processed payouts + Σ reversals − Σ penalties
```

### 3.2 Ledger — `EducatorLedgerEntry` (new, append-only)

```prisma
model EducatorLedgerEntry {
  id             String          @id @default(uuid())
  educatorId     String          @map("educator_id")
  seq            BigInt                              // per-educator gapless sequence
  entryType      LedgerEntryType
  amount         BigInt                              // SIGNED paise (see sign table)
  balanceAfter   BigInt          @map("balance_after") // running payable after this entry
  currency       String          @default("INR")
  // provenance (exactly one set, by type)
  bookingId      String?         @map("booking_id")
  payoutId       String?         @map("payout_id")
  adjustmentId   String?         @map("adjustment_id")
  // exactly-once guard
  idempotencyKey String          @unique @map("idempotency_key")
  note           String?
  createdAt      DateTime        @default(now()) @map("created_at")

  @@unique([educatorId, seq])
  @@index([educatorId, createdAt])
  @@map("educator_ledger_entry")
}

enum LedgerEntryType {
  accrual            // + booking net earned
  penalty            // − penalty applied
  penalty_reversal   // + penalty reversed
  payout             // − money sent
  payout_reversal    // + payout reversed (money came back)
  adjustment         // ± manual correction (dual-controlled)
}
```

**Sign convention**

| Type | Sign | idempotencyKey |
|------|------|----------------|
| accrual | + | `accrual:<bookingId>` |
| penalty | − | `penalty:<bookingId>` |
| penalty_reversal | + | `penalty_reversal:<bookingId>` |
| payout | − | `payout:<payoutId>` |
| payout_reversal | + | `payout_reversal:<payoutId>` |
| adjustment | ± | `adjustment:<adjustmentId>` |

`idempotencyKey UNIQUE` is the exactly-once guard: a retried accrual or a
duplicate webhook cannot double-post. `balanceAfter` gives O(1) audit and a
running statement; `seq` makes gaps detectable.

`Educator.pendingPayout` becomes a **cached** copy of the latest `balanceAfter`
(kept for fast reads/queries), reconciled against `SUM(amount)` nightly.

### 3.3 Payout — `Payout` (supersedes `PayoutHistory`)

```prisma
model Payout {
  id                 String       @id @default(uuid())
  educatorId         String       @map("educator_id")
  amount             BigInt                             // paise
  currency           String       @default("INR")
  status             PayoutStatus @default(created)
  mode               String?                            // IMPS | NEFT | UPI
  purpose            String       @default("payout")
  // RazorpayX linkage
  providerPayoutId   String?      @unique @map("provider_payout_id") // pout_...
  fundAccountId      String?      @map("fund_account_id")            // fa_ snapshot used
  utr                String?                                          // bank ref, on processed
  // controls
  initiatedByAdminId String?      @map("initiated_by_admin_id")
  approvedByAdminId  String?      @map("approved_by_admin_id")
  idempotencyKey     String       @unique @map("idempotency_key")    // X-Payout-Idempotency
  failureReason      String?      @map("failure_reason")
  createdAt          DateTime     @default(now()) @map("created_at")
  updatedAt          DateTime     @updatedAt @map("updated_at")
  processedAt        DateTime?    @map("processed_at")
  reversedAt         DateTime?    @map("reversed_at")

  @@index([educatorId, status])
  @@map("payout")
}
```

### 3.4 Payee ids on `Educator`

```prisma
razorpayContactId     String? @map("razorpay_contact_id")     // cont_… (create once)
razorpayFundAccountId String? @map("razorpay_fund_account_id") // fa_…  (create once; null on bank edit)
bankVerifiedAt        DateTime? @map("bank_verified_at")        // penny-drop validation
```

Null `razorpayFundAccountId` whenever bank details change → next payout recreates
the fund account for the new account.

---

## 4. Payout lifecycle (state machine)

```
                 admin authorises
   created ───────────────────────► authorized
      │                                  │
      │ (validation fails)               │ (dual-control, if required)
      ▼                                  ▼
   cancelled                         approved
                                         │  ensureFundAccount + POST /payouts (idempotent)
                                         ▼
                                     initiated ──────────► processing
                                         │                     │
                       webhook: failed / │                     │ webhook: processed
                       rejected / queued │                     ▼
                                         ▼                   processed ──► (webhook: reversed) ──► reversed
                                      failed
```

- **created → authorized**: passes checks (amount ≤ payable, bank present, no
  in-flight payout for this educator).
- **authorized → approved**: only if amount ≥ `payout_dual_control_threshold`
  needs a second admin; else auto-approved.
- **approved → initiated**: `ensureFundAccount`, then `POST /payouts` with the
  idempotency header. **No ledger movement yet.**
- **processing → processed**: `payout.processed` webhook → write `payout` ledger
  entry (−), decrement cached `pendingPayout`, store UTR, notify educator.
- **… → failed**: `payout.failed/rejected` → **no** ledger entry, alert admin,
  educator payable untouched.
- **processed → reversed**: `payout.reversed` → write `payout_reversal` (+),
  restore payable, alert admin + educator.

**The balance only ever moves on a confirming webhook** (principle 6).

---

## 5. RazorpayX integration

### 5.1 Entities (create-once, reuse)

`Contact (cont_)` → `Fund Account (fa_)` → `Payout (pout_)`. A payout needs a
`fund_account_id`; a fund account needs a contact. We cache both ids on the
educator so each later payout is a single call. (See the earlier "why store the
ids" rationale.)

### 5.2 Service — `services/payout/razorpayx.ts`

Thin **axios** client, Basic auth (`RAZORPAY_KEY_ID:RAZORPAY_KEY_SECRET`); the
SDK's payout coverage is inconsistent.

- `ensureContact(educator)` → create if `razorpayContactId` unset; persist.
- `ensureFundAccount(educator)` → create bank fund account from stored details if
  unset; persist. **Runs Fund Account Validation (penny-drop)** to confirm the
  account is real and the name matches before first payout; stamp
  `bankVerifiedAt`.
- `createPayout({ fundAccountId, amountPaise, referenceId, narration, idempotencyKey })`
  → `POST /v1/payouts` with header `X-Payout-Idempotency: <key>`
  (`account_number=RAZORPAYX_ACCOUNT_NUMBER`, `queue_if_low_balance=true`,
  `mode` chosen by amount). Returns `{ providerPayoutId, status }`.
- `getPayout(providerPayoutId)` / `listPayouts(since)` — for reconciliation.
- `getBalance()` — float-balance monitoring.

### 5.3 Idempotency

Two layers: (a) `X-Payout-Idempotency` = `Payout.idempotencyKey` so a retried
create never doubles at Razorpay; (b) `EducatorLedgerEntry.idempotencyKey` so a
duplicate webhook never double-posts to our books.

---

## 6. Webhooks — `POST /api/v1/webhooks/razorpayx`

- **Verify** `X-Razorpay-Signature` = HMAC-SHA256(rawBody, `RAZORPAYX_WEBHOOK_SECRET`).
  Needs the raw body (mirror the LiveKit webhook's `express.text` route).
- **Persist raw event first** in `WebhookEvent { provider, providerEventId,
  payload, status, receivedAt }`; **dedupe on `providerEventId`** — process each
  event once, tolerate replays.
- **Order-independent:** apply only legal state-machine transitions; a stale/older
  event for an already-terminal payout is acknowledged and ignored.
- **Missing events are caught by reconciliation** (§7), not assumed away.
- Handlers: `payout.processed | payout.failed | payout.rejected | payout.reversed`
  (+ `payout.updated`). Always return 2xx once stored, so Razorpay stops retrying;
  processing failures surface via `WebhookEvent.status=error` + alert.

---

## 7. Reconciliation

Three layers, each a scheduled job writing `ReconciliationBreak` rows on mismatch
and alerting; **none moves money automatically.**

**L1 — Internal (our books are self-consistent).** Nightly, per educator:
`pendingPayout == last balanceAfter == SUM(amount)` and `seq` has no gaps. Break
⇒ freeze that educator's payouts + alert. This catches accrual/webhook bugs.

**L2 — Us ↔ RazorpayX.** For every non-terminal `Payout` (and a trailing window
of terminal ones), `getPayout`/`listPayouts` and diff status + amount + UTR.
Detects: we think `processing`, Razorpay says `failed` (missed webhook); amount
drift; unknown payout. Advances state from the authoritative provider record.

**L3 — RazorpayX ↔ Bank/Float.** Daily, pull the RazorpayX **transactions/
settlement statement** + `getBalance`; assert Σ our processed payouts (− reversals)
== float debits, every processed payout has a UTR, and opening+credits−debits ==
closing balance. Detects money that left the float without a matching payout, or
vice-versa.

**Break handling:** breaks are tickets, not exceptions — resolved by a human via
an `adjustment` entry (dual-controlled), with the break linked to the resolving
entry.

---

## 8. Controls, security, compliance

**Maker–checker.** Payouts ≥ `payout_dual_control_threshold` need a second admin
to approve (`approvedByAdminId ≠ initiatedByAdminId`). Below it, single admin.
Keep RazorpayX's own approval workflow **off** (we own the control) — or on as a
second belt.

**Limits.** Per-payout cap, per-educator daily cap, platform daily total cap;
reject over-limit at authorize time.

**In-flight guard.** At most one non-terminal payout per educator (DB partial
unique / check) so a double-click can't create two.

**Bank details = PII.** Encrypt at rest, mask in UI/logs (`••••1234`), restrict
admin visibility, never log full numbers or webhook payloads with them. Penny-drop
before first payout (§5.2).

**Secrets.** Keys + webhook secret in env/secret manager; support rotation.

**Compliance (flag to finance/legal, don't assume):**
- **TDS** — payments to educators may attract withholding (e.g. 194J/194C). If so,
  the ledger must record gross, TDS, and net as separate entries. **Open question.**
- **GST** — educators who are registered businesses may raise invoices.
- **Statements** — educators get a downloadable earnings + payout statement
  (the ledger makes this trivial).
- **Retention** — ledger + payouts + webhook events retained per audit/tax rules
  (years), immutable.

**Observability.** Metrics: payout success rate, time-to-processed, failed/reversed
counts, RazorpayX float balance (low-balance alarm), pending-payable aging,
reconciliation-break count. Structured logs keyed by `payoutId`.

---

## 9. Schema changes (summary)

- **New:** `EducatorLedgerEntry`, `Payout`, `WebhookEvent`, `ReconciliationBreak`,
  enums `LedgerEntryType`, extend `PayoutStatus`
  (`created, authorized, approved, initiated, processing, processed, failed,
  rejected, reversed, cancelled`).
- **Educator:** `+ razorpayContactId, razorpayFundAccountId, bankVerifiedAt`;
  migrate `pendingPayout` Float → `BigInt` paise (cached balance).
- **Migrate** `PayoutHistory` → `Payout` (backfill), and **backfill the ledger**
  from historical accruals/payouts so `pendingPayout` reconciles from day one
  (one-off script; verify L1 passes before enabling disbursement).
- **Config:** `PAYOUT_PROVIDER=razorpayx|manual` (default `manual`),
  `RAZORPAYX_ACCOUNT_NUMBER`, `RAZORPAYX_WEBHOOK_SECRET`,
  `payout_dual_control_threshold`, per-educator/day + per-txn caps.

---

## 10. Phased rollout

- **M0 — Ledger foundation (no external money).** paise migration; `Payout` +
  `EducatorLedgerEntry`; `accrue.ts` writes an `accrual` entry; backfill; L1
  reconciliation + invariant asserts green. *De-risks the core; ships behind
  `PAYOUT_PROVIDER=manual` so behaviour is unchanged.*
- **M1 — RazorpayX in test mode.** service + penny-drop; idempotent createPayout;
  state machine; `WebhookEvent` ingest + handlers; admin trigger. Full round-trip
  in RazorpayX test mode; L2 reconciliation green.
- **M2 — Controls + reconciliation.** maker–checker, limits, in-flight guard; L2/L3
  jobs + break tickets + alerts; admin & educator payout-status UI (web + mobile,
  per parity); observability dashboards.
- **M3 — Go-live.** activate + fund RazorpayX; small pilot cohort; monitor float,
  success rate, breaks; ramp.
- **M4 — Compliance/statements.** TDS handling if required; educator statements;
  retention policy.

Branch per milestone (own PR), starting `feat/payout-ledger` (M0), per the
issue→branch→PR workflow.

---

## 11. Open questions

1. **RazorpayX activated + funded?** Hard prerequisite (separate from PG). Test
   mode available for M0–M2.
2. **TDS/withholding obligation** on educator payments? Changes the ledger (gross/
   TDS/net). Needs finance/legal.
3. **Dual-control threshold** amount? (₹ that forces a second approver.)
4. **UPI payouts** wanted, or bank-account only?
5. **Who funds the RazorpayX float**, and what low-balance threshold triggers an
   alert / top-up?
6. **Penny-drop** on every bank edit, or first payout only?
