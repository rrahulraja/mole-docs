# Background Crons

Reference for Mole's scheduled background jobs — what each one does, when it runs,
how it stays safe under concurrency, and what still needs work.

Crons live under **`backend/src/crons/`** and are started once from
`backend/src/index.ts` via `startCrons()`. They run **in-process** with
[`node-cron`](https://www.npmjs.com/package/node-cron) — there is no external
scheduler (no system crontab, no queue). If the backend process isn't running,
no crons run.

### Layout

```
src/crons/
  index.ts        registry — lists every job and registers it
  types.ts        the CronJob interface
  format.ts       istTime() — IST formatting for notification copy
  reminders.ts    pure window/wording helpers for session-reminders
  jobs/           one module per job, each exporting a CronJob
```

Each job module exports a `CronJob` — `{ name, schedule, run }` — that owns its own
schedule and body. `index.ts` only wires them up: it wraps every `run` in
`withCronLock(job.name, …)`, times it, and catches throws (node-cron drops rejected
handlers, so an uncaught error would otherwise vanish). It also **throws at boot on a
duplicate job name**, since the name doubles as the lock name and two jobs sharing one
would silently block each other.

`registry.test.ts` asserts the names are unique and every schedule expression parses,
so a typo fails in CI rather than at 3am.

## How the scheduling works

Each job is registered with a standard 5-field cron expression:

```
┌─ minute (0-59)
│ ┌─ hour (0-23)
│ │ ┌─ day of month (1-31)
│ │ │ ┌─ month (1-12)
│ │ │ │ ┌─ day of week (0-6, Sun=0)
* * * * *
```

Examples used here: `*/2 * * * *` = every 2 minutes · `0 2 * * *` = daily at 02:00 ·
`*/30 * * * *` = every 30 minutes.

## Concurrency safety: `withCronLock`

Every job body is wrapped in `withCronLock(name, fn)` (`backend/src/lib/cron-lock.ts`).
It uses a **row in the `cron_locks` table** as a distributed lock: claiming inserts a
row keyed on the cron name (`createMany({ skipDuplicates })` → count 0 means it's
already held, so the run is **skipped**). The row is deleted in a `finally` when the
run finishes. An `expiresAt` TTL (15 min) lets the next run **reclaim a stale lock**
if a run crashed without releasing.

This is pooling-safe (no session affinity) and works across multiple backend
instances — unlike the previous `pg_advisory_lock`, whose acquire/release could land
on different pooled connections behind pgBouncer/Supabase and leak the lock.

## Idempotency pattern

Because a job can run again before the previous effect is "seen" (or a second
instance runs), state changes use a **conditional `updateMany`** as an atomic claim:

```ts
const claimed = await prisma.booking.updateMany({
  where: { id, status: 'confirmed' },   // only if still in the expected state
  data:  { status: 'completed' },
});
if (claimed.count === 0) continue;      // someone else already handled it
// …only the winner runs the side effects (payout, notifications)
```

The same pattern guards penalties (`penaltyPct: 0 → 10`), one-time calls
(`*CallSentAt: null → now`), and the review prompt (`reviewPromptSent: false → true`).

---

## The jobs

| # | Name | Schedule | Purpose |
|---|------|----------|---------|
| 1 | `auto-complete` | every 5 min | Close finished sessions, accrue payout, notify |
| 2 | `session-reminders` | every 1 min | 1-hour and 5-minute reminders to both parties |
| 3 | `wallet-credit-expiry` | daily 02:00 | Expire unused promo credits |
| 4 | `educator-delay-check` | every 2 min (even) | Late-educator penalty → call → dispute |
| 5 | `student-delay-check` | every 2 min (odd) | Late-student reminder call + push |
| 6 | `review-prompt` | every 30 min | Follow-up nudge to review a past session |
| 7 | `stale-booking-expiry` | every 15 min | Cancel abandoned pre-confirmed bookings |
| 8 | `token-cleanup` | daily 03:00 | Delete expired/used auth tokens |
| 9 | `auto-payout` | daily 04:00 | Opt-in — *queues* educator payouts for settlement |

The two delay checks are offset (`*/2` vs `1-59/2`) so they never contend for the
same minute. Every scan is bounded with a `take` so one job can't monopolise a tick.

> Educator no-shows are handled entirely by `educator-delay-check` (job #4). An
> earlier `no-show-detection` cron disputed at 5 minutes and pre-empted that
> escalation; it was removed.

### 1. `auto-complete` — every 5 min
Finds `confirmed` bookings past `scheduledAt + durationMinutes + 5min` grace and flips
them to `completed`. On completion it:
- **accrues the educator payout** (`accrueEducatorPayout` — net = price × (1−commission%) × (1−penalty%), credited once to `Educator.pendingPayout`),
- **notifies the student** ("session completed — leave a review / take the quiz"),
- **notifies the educator** ("marked complete — earnings added to pending payout").

This is the fallback completion path when a session isn't ended via OTP exchange.

### 2. `session-reminders` — every minute
Sends both the **1-hour** and **5-minute** reminders to the student *and* the educator
(FCM push + in-app notification, session time in **IST**, deep-linked to
`mole://session/<id>`). One job does both so a single tick performs one scan per band
instead of two independent full scans.

Eligibility is **flag-based**, not window-based. Each booking carries
`reminder1hSent` / `reminder5mSent`, and the job selects on the flag plus a wide band:

| Band | Window (`scheduledAt`) | Claims |
|---|---|---|
| 5-minute | `now − 2min` → `now + 6min` | **both** flags |
| 1-hour | `now + 6min` → `now + 60min` | `reminder1hSent` |

The bands are **contiguous** — the 1h band starts exactly where the 5m band ends —
so a booking always falls in exactly one. The 5m band reaches slightly into the past
because a session that *just* started still deserves a "join now" nudge; once it's
5 minutes late it belongs to the delay checks instead. The 5m pass claims **both**
flags so a booking can never receive both reminders.

Why flags matter: with a 1-minute window, a **missed tick** (restart, deploy, a lock
held long) meant the window elapsed and the reminder was **never sent**. With flags
the booking stays eligible, so a missed tick sends **late instead of never**.

Because a late send would otherwise announce the wrong lead time, the title is derived
from the **actual** gap (`reminderTitle()` in `crons/reminders.ts`) — a 1h reminder
recovered when the session is 23 minutes away says *"Session in 23 minutes"*, not
*"Session in 1 hour"*. The window maths and wording are unit-tested in
`crons/reminders.test.ts`.

The flag is claimed **before** sending (so concurrent runs can't both deliver) and
**handed back if the send throws**, so a genuine failure retries on the next tick
rather than being silently swallowed.

### 3. `wallet-credit-expiry` — daily at 02:00
Finds `WalletCredit` rows past `expiresAt` that were never used (`usedAt: null`),
marks them used, decrements the student's `Wallet.balance` by the credit amount, and
writes a `credit_expired:<id>` debit to the ledger.

Only claws back `min(credit.amount, current balance)` so an already-spent credit
can’t drive the balance negative.

### 4. `educator-delay-check` — every 2 min (even minutes)
Two-stage escalation for a late educator (session has a `zoomSessionId`, educator not
joined):
- **5+ minutes late** → atomically apply a **10% penalty** (`penaltyPct: 0 → 10`),
  place an automated **voice reminder call** (`services/voice-call`), and push
  "join now — penalty applied".
- **15+ minutes late** → flip to `disputed`, refund credits, and **notify the student**.

### 5. `student-delay-check` — every 2 min (odd minutes)
When the **educator has joined** but the **student hasn't** 5+ minutes in: places a
one-time voice call to the student and pushes "your educator is waiting". No penalty
(the student is the payer); if they never join, #1 eventually completes the session.

### 6. `review-prompt` — every 30 min  *(follow-up)*
Finds `completed` bookings 2h+ old with **no review** and `reviewPromptSent: false`,
claims each atomically, and sends a one-time "how was your session?" nudge to the
student. The *immediate* review prompt is sent at completion (job #1 / OTP end); this
is the delayed reminder for people who didn't act on it.

### 7. `stale-booking-expiry` — every 15 min
Cancels abandoned pre-confirmed bookings that would otherwise sit forever:
`awaiting_educator` never accepted (>24h) and `pending_payment` never paid (>2h,
holding the slot). Atomically flips them to `cancelled` and notifies the student.
No refund — money/credits are only taken at payment (status → `confirmed`).

### 8. `token-cleanup` — daily at 03:00
Deletes expired/used rows from the auth-token tables: `OtpCode` (expired or used),
`RefreshToken` (expired), and `AdminAuthToken` (expired or used). Pure hygiene —
these tables only grow otherwise.

### 9. `auto-payout` — daily at 04:00  *(opt-in)*
Runs only when `auto_payout_enabled` is true **and** today matches `auto_payout_day`
(0=Sun … 6=Sat). For each `approved` educator whose `pendingPayout` is at least
`min_payout_amount`, it **queues** a `PayoutHistory` row with `status: 'pending'` and
notifies them that a payout is on the way. Idempotent per day via a `note` guard
(`Auto payout run <date>`).

> **Mole cannot move money — there is no bank/UPI integration.** So this job does
> **not** mark anything `paid` and does **not** touch `pendingPayout`. It only builds
> the admin's worklist.

Settlement is a separate, human step: an admin makes the transfer out-of-band, then
settles the row via **`POST /admin/payouts/:id/settle`** with the bank/UPI reference.
*That* is what sets `status: 'paid'`, stamps `paidAt`, decrements `pendingPayout` and
sends the "payout processed" notification. The settle endpoint is guarded
(`status: 'pending'` → `paid`), so two admins clicking at once can't double-decrement.

An educator with an **unsettled** row from an earlier run is skipped — their
`pendingPayout` still includes that amount, so queueing again would double-count it.

Earlier this job wrote `status: 'paid'` and decremented `pendingPayout` directly,
which made the ledger assert money had been transferred when nothing had. A job that
cannot move money must never record money as moved.

---

## Notifications

All push/in-app notifications use `sendPushWithRecord(fcmToken, userId, title, body, data?)`
(`services/notification`), which **always writes a `Notification` row** (so web-only
users see it in-app) and additionally sends FCM if the user has a token. Every cron
call is best-effort (`.catch(console.error)`) so a delivery failure never aborts the
run. Times embedded in bodies are formatted in **IST** (`Asia/Kolkata`).

## Resolved

- ~~Duplicate no-show handling~~ — the `no-show-detection` cron was removed; `educator-delay-check` owns the escalation.
- ~~Advisory-lock pooling risk~~ — replaced with the `cron_locks` table lock (pooling-safe, TTL recovery).
- ~~`wallet-credit-expiry` negative balance~~ — now claws back `min(amount, balance)`; log moved out of the loop.
- ~~`student-delay-check` "Completed" start-log + redundant call-timestamp writes~~ — fixed.
- ~~Missing crons: auto-payout, stale-booking expiry, token cleanup~~ — implemented (jobs #7–#9).
- ~~Reminders were window-based~~ — now flag-based (`reminder1hSent` / `reminder5mSent`), so a missed tick sends late instead of never.
- ~~Cadence churn~~ — the two reminder crons merged into one job, the delay checks staggered onto opposite minutes, `auto-complete` bounded to bookings past their earliest possible end, and every scan given a `take`.
- ~~`auto-payout` marked money paid that never moved~~ — it now only *queues* a `pending` row; an admin settles it after the real transfer.

## Known issues / future work

1. **No bank/UPI transfer provider.** `auto-payout` queues payouts and an admin
   settles them by hand after transferring out-of-band. Wiring a real payout provider
   (RazorpayX / Cashfree payouts) would let settlement be automatic — at which point
   the settle step becomes a callback from the provider rather than an admin action.
2. **In-process scheduling.** Crons run inside the API process, so they stop when it
   stops and compete with request handling. A separate worker process (or a real queue)
   would isolate them. The `cron_locks` table lock already makes multi-instance safe.
3. **No delivery record for reminders.** The flags say a reminder was *attempted*, not
   that it arrived. FCM failures are swallowed as best-effort; there's no retry ladder
   or per-notification delivery status.

## Adding a new cron

1. Create `src/crons/jobs/<name>.ts` exporting a `CronJob`:

   ```ts
   import { CronJob } from '../types';

   export const myJob: CronJob = {
       name: 'my-job',            // also the lock name — must be unique
       schedule: '*/10 * * * *',
       async run() { /* … */ },
   };
   ```

2. Import it in `src/crons/index.ts` and add it to `CRON_JOBS`. That's all the
   wiring — the lock, timing and error handling are applied for you, so `run`
   should **not** call `withCronLock` itself.
3. Guard every state change with a conditional `updateMany` (idempotency).
4. Wrap per-item side effects in `try/catch` so one bad row doesn't abort the run;
   make notifications best-effort (`.catch(console.error)`).
5. Keep the query bounded (`take`) and only select the fields you need.
6. If the job needs a non-trivial decision (time windows, wording, thresholds),
   put it in a pure helper next to `reminders.ts` and unit-test it — that logic is
   otherwise unreachable from a test without a database.
7. Document it here.
