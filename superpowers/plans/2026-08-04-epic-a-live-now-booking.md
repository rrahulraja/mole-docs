# Epic A — Live-Now Instant Booking Implementation Plan

> **For agentic workers:** implement task-by-task. Steps use checkbox (`- [ ]`) syntax.

**Goal:** Book Now (live-only, 5-min accept window, multi-channel notify, penalty,
blinking awaiting state, race-safe) + Book for later (auto-book, no accept).

**Architecture:** `createBooking` branches on `instant`: scheduled → `pending_payment`
directly; instant → `awaiting_educator` + `instantExpiresAt` + 3-channel notify.
Accept reuses the existing `awaiting_educator → pending_payment` path. Decline and a
1-min expiry cron apply an idempotent %-penalty to `pendingPayout`. Clients gain the
two buttons and a polling, blinking awaiting screen.

**Tech Stack:** Express + Prisma · node-cron · Next.js 14 + MUI · Expo/RN · `@mole/shared`.

## Global Constraints

- Base branch **main**; each unit issue→branch→PR→merge→Project board #2 Done.
- Web+mobile parity for every student booking-UI change.
- Prisma: migrate (diff+deploy workaround) + `pnpm generate:prisma` before typecheck.
- Commit as `Rrahul Raja <rahulraja.ngp@gmail.com>`; strip Claude trailers.
- Never commit `.env*`, `admin/vite.config.ts`, `docs/deployment/vps-deploy.md`.
- Reuse existing services: `notification`, `whatsapp`, `voice-call`, `payout`; cron
  via `withCronLock` + the `CronJob` registry.

---

## Unit 1 — Schema + booking split (scheduled auto-book / instant window) + notify

**Files:**
- Modify: `backend/prisma/schema.prisma` (Booking) + migration + `seed.ts` (2 configs)
- Modify: `backend/src/services/booking/index.ts` (`createBooking` branch + notify)
- Create: `backend/src/services/booking/create.test.ts`

**Produces:** instant bookings carry `isInstant` + `instantExpiresAt`; scheduled
bookings land in `pending_payment`.

- [ ] Add `isInstant`, `instantExpiresAt`, `instantPenaltyApplied` to `Booking`;
  migrate + generate.
- [ ] Seed/upsert `instant_accept_window_min = 5`, `instant_no_accept_penalty_pct = 10`.
- [ ] `createBooking`: for `instant`, set `status: 'awaiting_educator'`, `isInstant: true`,
  `instantExpiresAt = now + window`; notify educator via push + WhatsApp (window noted)
  + `makeReminderCall` (best-effort, `voice_calls_enabled`-gated). For scheduled, set
  `status: 'pending_payment'` and send an informational push (no accept links).
- [ ] `create.test.ts` (prisma-mock): scheduled → `pending_payment`; instant →
  `awaiting_educator` with a non-null `instantExpiresAt`.
- [ ] Backend `typecheck` + jest; commit `feat(backend): instant vs scheduled booking creation + live-now notifications`.

## Unit 2 — Accept/decline/expiry + penalty + cron

**Files:**
- Create: `backend/src/services/payout/instant-penalty.ts`
- Create: `backend/src/services/payout/instant-penalty.test.ts`
- Modify: `backend/src/controllers/booking.controller.ts` (`applyEducatorResponse` decline → penalty)
- Create: `backend/src/crons/jobs/instant-accept-expiry.ts`
- Modify: `backend/src/crons/registry.ts` (register the job)
- Create: `backend/src/crons/jobs/instant-accept-expiry.test.ts`

**Consumes:** Unit 1 fields + configs.

- [ ] `applyInstantNoAcceptPenalty(bookingId)`: atomic claim `instantPenaltyApplied
  false→true`; `penalty = round(totalPrice × pct/100)`; `pendingPayout = max(0, −penalty)`;
  notify educator; no-op when pct = 0 or already applied.
- [ ] `instant-penalty.test.ts` (prisma-mock): idempotent claim, math, floor, pct=0 no-op.
- [ ] `applyEducatorResponse('decline')` on an instant booking: apply the penalty
  before/after the status flip to `cancelled` (status-guarded).
- [ ] `instant-accept-expiry` cron (`* * * * *`): claim instant `awaiting_educator`
  rows with `instantExpiresAt < now` via status-guarded `updateMany → cancelled`;
  for each claimed, apply the penalty + notify student ("didn't respond in time").
- [ ] Register the cron; `instant-accept-expiry.test.ts`: only past-window instant
  awaiting rows are claimed; idempotent.
- [ ] Backend `typecheck` + jest; commit `feat(backend): instant no-accept penalty + 5-min accept-window expiry cron`.

## Unit 3 — Web: Book Now / Book for later + awaiting-educator state

**Files:**
- Modify: `web/components/student/EducatorCard.tsx` (two actions)
- Modify: `web/components/student/BookingPanel.tsx` / `web/app/student/educator/[id]/page.tsx`
- Create/Modify: web awaiting-educator view (blinking + poll)

**Consumes:** Units 1–2 endpoints; `isLive`.

- [ ] Card + profile: **Book Now** (enabled only when `isLive`, disabled + hint
  otherwise, posts `instant: true`) and **Book for later** (scheduled picker).
- [ ] Instant confirm → blinking "Awaiting educator…" with a countdown that polls
  `GET /bookings/:id`; `pending_payment` → payment, `cancelled` → "didn't respond,
  pick another".
- [ ] Scheduled → straight to payment (already `pending_payment`).
- [ ] Web `typecheck` (changed files clean); commit `feat(web): Book Now vs Book for later + blinking awaiting-educator state`.

## Unit 4 — Mobile parity

**Files:**
- Modify: `mobile/components/EducatorCard.tsx` (two actions)
- Modify: `mobile/app/(student)/educator/[id].tsx` (Book Now gating)
- Modify: `mobile/app/(student)/confirm-booking.tsx` (awaiting poll + countdown + pulse)

**Consumes:** Units 1–3.

- [ ] Card + profile: Book Now (live-only) + Book for later.
- [ ] `confirm-booking` awaiting state: poll `GET /bookings/:id`, 5-min countdown,
  pulse animation; `pending_payment` → payment, `cancelled` → rebook message.
- [ ] Mobile `tsc` (changed files clean); commit `feat(mobile): Book Now vs Book for later + blinking awaiting-educator state`.

---

## Self-Review
- Coverage: schema+split+notify (U1), accept/decline/expiry+penalty (U2), web UI (U3),
  mobile UI (U4) — all spec sections mapped.
- Types: `isInstant`/`instantExpiresAt`/`instantPenaltyApplied` consistent; penalty
  helper mirrors `accrueEducatorPayout`'s idempotent-claim shape.
- Parity: U3 web + U4 mobile mirror the two buttons + awaiting state.
- Race: accept / decline / expiry / student-cancel all status-guarded atomic updates.
