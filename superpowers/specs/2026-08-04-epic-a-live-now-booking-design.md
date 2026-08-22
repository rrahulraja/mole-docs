# Epic A — Live-Now Instant Booking Design

**Date:** 2026-08-04
**Status:** Approved (design decisions locked)
**Depends on:** Epic B (slot picker, `instant` flag plumbing) — shipped. Epic C
(pricing/`totalPrice`) — shipped.

## Goal

Two booking paths from search: **Book Now** (enabled only when the educator is
live) triggers an instant-accept flow with a 5-minute window, multi-channel
educator notification, a blinking "awaiting educator" state for the student, race
handling, and a configurable penalty if the educator doesn't accept; **Book for
later** auto-books with no educator accept step.

Spec points covered: **#1** (Book Now / Book for later buttons) and **#2**
(live-now instant booking + notifications + window + penalty + race + blinking
state; later = auto-book).

## Locked decisions

1. **Book-for-later = auto-book, no accept.** A scheduled booking skips the
   educator accept step and is created directly in `pending_payment`; the student
   pays and it becomes `confirmed`. The educator gets an informational
   notification, not an accept/decline prompt.
2. **Book Now = accept-first, then pay.** An instant booking is created in
   `awaiting_educator` with a 5-minute accept window. On educator **accept** it
   moves to `pending_payment` (existing path) → student pays → `confirmed` →
   session. No money is held during the window, so a no-accept needs no refund.
3. **No-accept penalty = configurable %, no offline flip.** If the educator
   ignores the window (expiry) or declines an instant request, the educator stays
   live (NOT set offline). A penalty of `instant_no_accept_penalty_pct`% of the
   booking's `totalPrice` is deducted from `Educator.pendingPayout` (floored at 0),
   applied **once** (idempotent guard). No session, so no other money moves.
4. **Window expiry = 1-minute cron.** A node-cron job (guarded by `withCronLock`)
   sweeps every minute and expires instant `awaiting_educator` bookings past their
   `instantExpiresAt` (effective window 5–6 min).

## Data model

`Booking` gains (migration, backfilled defaults):

```prisma
isInstant             Boolean   @default(false) @map("is_instant")
instantExpiresAt      DateTime?                  @map("instant_expires_at")
instantPenaltyApplied Boolean   @default(false) @map("instant_penalty_applied")
```

New `PlatformConfig` keys (seed):

- `instant_accept_window_min` = `5`
- `instant_no_accept_penalty_pct` = `10` (x; admin-editable)

## Flow

### Book for later (scheduled)

`createBooking({ instant: false })` → validation (Epic B) → create in
`pending_payment` (was `awaiting_educator`) → informational push to educator
("You have a new session on <date>") → client routes to payment.

### Book Now (instant)

`createBooking({ instant: true })` → validation (overlap + min-duration; the
availability-window check is already skipped for instant) → create in
`awaiting_educator` with `isInstant = true`, `instantExpiresAt = now +
instant_accept_window_min`. Notify the educator on **three** channels:

- **App push** (`sendPushWithRecord`) with the respond deep-link (existing).
- **WhatsApp** accept/decline links (existing `sendWhatsAppMessage`), noting the
  5-minute window.
- **Voice call** (`makeReminderCall`) — gated by the existing `voice_calls_enabled`
  config; best-effort, never blocks the booking.

Client shows the blinking **"Awaiting educator…"** state and polls booking status.

**Accept (within window):** `applyEducatorResponse('accept')` — unchanged path →
`pending_payment`; the student's polling sees it and routes to payment.

**Decline:** apply the penalty (see below), set `cancelled`, notify the student to
rebook.

**Expiry (cron):** window passed, still `awaiting_educator` + `isInstant` → apply
the penalty, set `cancelled`, notify the student ("The educator didn't respond in
time") and the educator (penalty applied).

### Penalty helper

`applyInstantNoAcceptPenalty(bookingId)` in `services/payout/` (near
`accrueEducatorPayout`): atomically claim via `instantPenaltyApplied = false →
true` (idempotent, like `payoutAccrued`); compute `penalty = round(totalPrice ×
instant_no_accept_penalty_pct / 100)`; `Educator.pendingPayout = max(0, pending −
penalty)`; notify the educator. No-op when the pct is 0.

### Race handling

- **Late accept** (educator taps accept after expiry): the existing guard already
  rejects when `booking.status !== 'awaiting_educator'`; the cron will have moved
  it to `cancelled`, so accept returns `INVALID_STATUS` → the respond UI shows
  "This request expired." The cron claims the row with a status-guarded
  `updateMany`, so accept-vs-expiry can't both win.
- **Student moved on** (student cancels during the window): the existing cancel
  path allows `awaiting_educator`; a later educator accept then sees `cancelled`
  and no-ops. No penalty when the student cancelled.
- **Accept vs decline vs expiry** all funnel through status-guarded atomic
  updates, so exactly one transition takes effect.

## Frontend

### Search / educator card (web + mobile) — point #1

Two actions per educator: **Book Now** (enabled only when `isLive`; disabled with a
hint otherwise) and **Book for later**. Book Now opens the instant confirm →
awaiting state; Book for later opens the session-type-first picker (Epic B).

### Awaiting-educator state (web + mobile) — point #2

A blinking/pulsing "Awaiting educator…" screen with the 5-minute countdown that
polls `GET /bookings/:id` every few seconds and transitions:

- `pending_payment` → go to payment (accepted).
- `cancelled` → "The educator didn't respond — pick another educator or try
  later." (expired/declined).

Mobile already has an `awaiting_educator` state in `confirm-booking.tsx`; it gains
polling + countdown + the pulse animation. Web adds the equivalent.

## Testing

- **Backend unit** (prisma-mock): `applyInstantNoAcceptPenalty` — idempotent claim,
  pct math, pendingPayout floor, pct=0 no-op. `createBooking` — scheduled →
  `pending_payment`, instant → `awaiting_educator` + `instantExpiresAt`.
- **Backend unit**: the instant-expiry cron claims only past-window instant
  awaiting rows and is idempotent.
- **Frontend**: typecheck only (no runner).

## Out of scope

- Changing the scheduled cancel / refund windows, payout settlement, or the
  reassignment/no-show crons.
- Educator-side "I'm live" toggle changes (already exists: `isLiveNow`).
- SMS (non-WhatsApp) instant notifications.
