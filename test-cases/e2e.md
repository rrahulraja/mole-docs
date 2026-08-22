# E2E — Cross-surface journeys

End-to-end journeys that span **student ↔ educator ↔ admin** and multiple apps
(web / mobile / admin / backend). These are the highest-value regression flows.
Format per [README.md](./README.md).

---

## Full booking lifecycle (happy path)

> **TC-E2E-001: Discover → book → pay → attend → end → review → publish**
> - **Type:** e2e
> - **Priority:** P0
> - **Preconditions:** An approved educator with rates + availability; a student with a completed profile.
> - **Steps:**
>   1. Student (web/mobile) opens Home → sees educator under Suggested/Live → opens profile.
>   2. Student books: session type + topic + duration + date + available slot → `POST /bookings` → booking `awaiting_educator`.
>   3. Educator accepts (WhatsApp/link `/:id/respond`) → booking `pending_payment`, `Payment` row created, student notified.
>   4. Student pays: `POST /payment/:id/intent` → (dev) `POST /payment/:id/mock-confirm` → webhook path → booking `confirmed`.
>   5. Student joins session (`POST /session/:id/join` → `studentJoinedAt`); educator joins on **web** (`educatorJoinedAt`).
>   6. Both joined → backend generates 6-digit `sessionEndOtp`, pushed to student.
>   7. Student reads OTP to educator → educator enters it → `POST /sessions/:id/end` → booking `completed`.
>   8. Student submits review → `status = pending` (NOT visible).
>   9. Admin opens Reviews → Approve → review `approved`, educator `overallRating` recomputed.
> - **Expected:** Booking ends at `completed`; review is hidden until admin approval; after approval it appears on the educator profile (`GET /discover/:id` reviews) on both web + mobile; educator earnings reflect the completed session.

> **TC-E2E-002: Cancellation before session refunds credits**
> - **Type:** e2e · **Priority:** P1
> - **Preconditions:** A `confirmed` booking where the student applied wallet credits, >1h before start.
> - **Steps:** Student cancels → `PATCH /bookings/:id/cancel`.
> - **Expected:** Booking `cancelled`; applied credits refunded to wallet (credit transaction); slot freed.

---

## Session extend (mid-call)

> **TC-E2E-010: Student extends an in-progress session**
> - **Type:** e2e · **Priority:** P0
> - **Preconditions:** A `confirmed` booking, both joined, session in progress.
> - **Steps:**
>   1. Student taps Extend 30 → `GET /bookings/:id/extend?minutes=30` → `extensionCost` + `upiUri`.
>   2. Student pays via UPI → `POST /bookings/:id/extend/confirm { extensionMinutes: 30 }`.
> - **Expected:** `durationMinutes` increases by 30; `GET /bookings/:id` reflects the new duration; the end-OTP expiry window extends accordingly; the session countdown updates.

---

## End-OTP guard (security-critical)

> **TC-E2E-020: Educator cannot end session without the student's OTP**
> - **Type:** e2e · **Priority:** P0
> - **Steps:** With both joined, educator calls `POST /sessions/:id/end` with a wrong/absent OTP.
> - **Expected:** `INVALID_OTP` / `OTP_REQUIRED`; booking stays `confirmed`. Only the correct OTP transitions it to `completed`. Expired OTP → `OTP_EXPIRED`.

> **TC-E2E-021: OTP is only generated once both parties join**
> - **Type:** e2e · **Priority:** P1
> - **Steps:** Only the student joins; educator attempts to end.
> - **Expected:** `OTP_NOT_GENERATED_YET`; no OTP exists until `educatorJoinedAt` and `studentJoinedAt` are both set.

---

## Educator onboarding → verification → discoverable

> **TC-E2E-030: New educator becomes bookable only after admin approval**
> - **Type:** e2e · **Priority:** P0
> - **Steps:**
>   1. User signs up as educator: onboarding step1 (profile) → step2 (identity/KYC) → status `pending`.
>   2. Educator app shows the **under-verification** blocking screen; educator does NOT appear in `/discover/*`.
>   3. Admin opens Educators (pending) → reviews KYC → Approve → status `approved`, referral code issued.
>   4. Educator sets rates + availability slots.
> - **Expected:** Only after approval + rates + availability does the educator appear in discovery/search and become bookable. Rejected → **rejected** screen; suspended → cannot go live / not discoverable.

---

## Educator joins from web only (cross-device)

> **TC-E2E-040: Educator session is web-only; student can be on mobile**
> - **Type:** e2e · **Priority:** P0
> - **Steps:** Student joins from mobile; educator opens the session on **mobile**.
> - **Expected:** Mobile educator screen shows the "Join from the web app" notice and makes **no** session token/join calls. Educator joins successfully from **web**. The session still completes end-to-end (OTP end works) with student-on-mobile + educator-on-web.

---

## No-show / dispute → admin resolution

> **TC-E2E-050: Educator no-show auto-disputes and refunds**
> - **Type:** e2e · **Priority:** P0
> - **Preconditions:** `confirmed` booking; educator never joins; scheduled time passes by the no-show threshold.
> - **Steps:** No-show cron runs; later, admin resolves the dispute.
> - **Expected:** Booking → `disputed` (applied credits refunded per cron); admin resolve → `completed` with optional refund; dismiss → `completed`, no refund. Educator delay (5+ min) applies `penaltyPct`.

> **TC-E2E-051: Student opens a dispute within the window**
> - **Type:** e2e · **Priority:** P1
> - **Steps:** After a `completed` session (within 24h), student opens a dispute (reason ≥20 chars); both parties add notes; admin resolves with a credit refund.
> - **Expected:** Booking `disputed` → `completed`; refund credited; dispute-window-expired attempts return `DISPUTE_WINDOW_EXPIRED`.

---

## Review moderation visibility (cross-surface)

> **TC-E2E-060: Pending reviews are invisible; approval publishes to all students**
> - **Type:** e2e · **Priority:** P0
> - **Steps:** Student A reviews educator → pending. Student B views the educator profile (web + mobile). Admin approves. Student B refreshes.
> - **Expected:** Before approval, the review is absent from `/discover/:id` and doesn't affect `reviewCount`/`ratingBreakdown`/`overallRating`. After approval it's visible to every student and the rating recomputes. Reject → stays hidden permanently.

---

## Referral dual-credit

> **TC-E2E-070: Referrer and new student both get wallet credit**
> - **Type:** e2e · **Priority:** P1
> - **Steps:** Referrer shares code → new student signs up with it (profile setup) → check both wallets and `GET /referral/me/stats`.
> - **Expected:** Both parties credited the signup bonus; `referralCount` increments for the referrer; `student.referredBy` set. Invalid/non-student codes grant nothing.

---

## Earnings → payout

> **TC-E2E-080: Completed sessions accrue educator earnings, admin pays out**
> - **Type:** e2e · **Priority:** P1
> - **Steps:** Complete several sessions → check `GET /educator/me/earnings` (net after commission) → admin Payouts, Pending tab, shows the educator → admin processes the payout with a UTR reference.
> - **Expected:** Net earnings = gross × (1 − commission) × (1 − penalty); after the payout the amount moves to History (`PayoutStatus.paid`) and `pendingPayout` drops.

> **TC-E2E-081: Automated run queues, admin settles**
> - **Type:** e2e · **Priority:** P0
> - **Preconditions:** `auto_payout_enabled = true` and today is `auto_payout_day`; an approved educator with accrued earnings above `min_payout_amount`.
> - **Steps:** Run the `auto-payout` cron → open admin Payouts → **Queued** tab → transfer the money out-of-band → Settle the row with the bank UTR.
> - **Expected:** After the cron the educator has a `pending` `PayoutHistory` row and a "Payout queued" notification, but `pendingPayout` is **unchanged** and the row is absent from History — nothing in Mole can move money, so nothing may claim it did. Only the Settle action flips the row to `paid`, stamps `paidAt` + reference, decrements `pendingPayout`, and notifies the educator "Payout processed". Re-running the cron before settling queues nothing new for that educator.

---

## Wallet credit expiry

> **TC-E2E-090: Unused wallet credits expire via cron**
> - **Type:** e2e · **Priority:** P2
> - **Steps:** Grant a credit with an expiry; let the daily expiry cron run without the student using it.
> - **Expected:** Credit marked used/expired; wallet balance decremented; a debit transaction recorded.

---

## Payment failure path

> **TC-E2E-100: Failed/timed-out payment lands on booking-failed**
> - **Type:** e2e · **Priority:** P0
> - **Steps:** Create booking → intent → let the 5-min countdown expire (or webhook FAILURE).
> - **Expected:** Payment `failed`, booking `failed`; client routes to booking-failed; retry path available where allowed (`CANNOT_RETRY` otherwise).
