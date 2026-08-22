# Epic B — Slot Picker + Minimum Durations Implementation Plan

> **For agentic workers:** implement task-by-task. Steps use checkbox (`- [ ]`) syntax.

**Goal:** Session-type-first booking: pick type → duration (type-aware, min-enforced) →
date → a 15-minute-grid, overlap-free start time inside the educator's availability,
validated on the server.

**Architecture:** Shared duration constants; a duration-aware, overlap-aware
availability slot generator; `createBooking` server validation (min-duration,
overlap, availability window — window skipped for instant); profile exposes
`bookableSessionTypes`. Web + mobile booking UIs reorder to type-first and fetch
duration-aware slots.

**Tech Stack:** Express + Prisma · Next.js 14 + MUI · Expo/RN · `@mole/shared`.

## Global Constraints

- Base branch **main**; each unit issue→branch→PR→merge→Project board #2 Done.
- Web+mobile parity for every student booking-UI change.
- Commit as `Rrahul Raja <rahulraja.ngp@gmail.com>`; strip Claude trailers.
- Never commit `.env*`, `admin/vite.config.ts`, `docs/deployment/vps-deploy.md`.
- Money/duration = integer minutes; `totalPrice = round(rate × minutes / 60)`.
- Session types + values from `@mole/shared` `SESSION_TYPE`; durations from the new
  `SESSION_MIN_MINUTES` / `SESSION_DURATION_OPTIONS`.

---

## Unit 1 — Shared duration constants + backend booking validation

**Files:**
- Create: `shared/src/constants/durations.ts`
- Modify: `shared/src/index.ts` (export)
- Modify: `shared/src/constants/errors.ts` (new codes)
- Modify: `backend/src/services/booking/index.ts` (`createBooking` validation + `instant`)
- Modify: `backend/src/controllers/booking.controller.ts` (pass `instant`, map new errors)
- Create: `backend/src/services/booking/validate.ts` (pure overlap + window helpers)
- Create: `backend/src/services/booking/validate.test.ts`

**Produces:** `SESSION_MIN_MINUTES`, `SESSION_DURATION_OPTIONS`, `SLOT_STEP_MINUTES`;
`overlaps()`, `withinAvailability()`; `createBooking` rejects `DURATION_TOO_SHORT` /
`SLOT_UNAVAILABLE` / `OUTSIDE_AVAILABILITY`.

- [ ] Add `durations.ts` (constants from spec) + export from index; add error codes
  `DURATION_TOO_SHORT`, `SLOT_UNAVAILABLE`, `OUTSIDE_AVAILABILITY`.
- [ ] `validate.ts`: `overlaps(startA,durA,startB,durB)` interval test;
  `withinAvailability(start, duration, windows)` (start inside a window and
  `start+dur ≤ end`). Pure, minutes-based.
- [ ] `validate.test.ts`: overlap true/false edges (touching endpoints don't
  overlap), window fit/overflow, outside-window. Run → PASS.
- [ ] `createBooking`: accept `instant?: boolean`; fetch educator active
  availability + same-day existing bookings (`confirmed`/`pending_payment`/
  `awaiting_educator`); enforce min-duration always, overlap always, window when
  `!instant`. Throw the new codes.
- [ ] `booking.controller.ts`: read `instant` from body, pass through; map new
  errors to 400 (duration) / 409 (overlap, outside-availability).
- [ ] Backend `typecheck` + jest; commit `feat(backend): booking min-duration + overlap + availability validation`.

## Unit 2 — Duration-aware, overlap-aware availability endpoint

**Files:**
- Create: `backend/src/services/booking/slots.ts` (pure `freeStartTimes(...)`)
- Create: `backend/src/services/booking/slots.test.ts`
- Modify: `backend/src/controllers/educator.controller.ts` (`getPublicAvailability`)

**Consumes:** Unit 1 `overlaps`, `SLOT_STEP_MINUTES`.

- [ ] `slots.ts`: `freeStartTimes({ windows, duration, bookings, now, isToday, bufferMin })`
  → `string[]` of `HH:MM` on the 15-min grid, window-fit, overlap-free, past-filtered.
- [ ] `slots.test.ts`: grid step, window-end clamp for long durations, overlap
  removal against a mid-window booking, today buffer. Run → PASS.
- [ ] `getPublicAvailability`: require `date` + `duration`; gather that weekday's
  active windows + existing bookings; return `{ slots: freeStartTimes(...).map(t => ({ time24: t })) }`.
- [ ] Backend `typecheck` + jest; commit `feat(backend): duration-aware overlap-free availability slots`.

## Unit 3 — Profile `bookableSessionTypes` + web booking UI

**Files:**
- Modify: `backend/src/controllers/discover.controller.ts` (`getEducatorProfile` → `bookableSessionTypes` + `maxAvailabilityMinutes`)
- Modify: `web/components/student/BookingPanel.tsx`
- Modify: `web/app/student/educator/[id]/page.tsx` (if it gates types)

**Consumes:** Units 1–2; `SESSION_DURATION_OPTIONS`, `SESSION_MIN_MINUTES`.

- [ ] `getEducatorProfile`: compute longest active availability window; return
  `bookableSessionTypes` = types with an effective rate **and** `maxWindow ≥ min[type]`.
- [ ] `BookingPanel`: session type first (only `bookableSessionTypes`); duration
  options from `SESSION_DURATION_OPTIONS[type]`, default min; switching type resets
  duration; slot fetch passes `duration`; changing date/duration clears slot;
  `instant` passes through; map new error codes to messages.
- [ ] Web `typecheck` (changed files clean); commit `feat(web): session-type-first booking with duration-aware slot picker`.

## Unit 4 — Mobile booking UI parity

**Files:**
- Modify: `mobile/app/(student)/educator/[id].tsx` (booking sheet)
- Modify: `mobile/app/(student)/book/[id].tsx` (if separate scheduled flow)
- Modify: `mobile/app/(student)/confirm-booking.tsx` (duration/price if needed)

**Consumes:** Units 1–3 (same endpoints + `bookableSessionTypes`).

- [ ] Mobile booking sheet: type-first (only `bookableSessionTypes`); duration from
  `SESSION_DURATION_OPTIONS[type]`, default min, reset on type change; slot fetch
  passes `duration`; clear slot on date/duration change; `instant` for Book Now;
  map new error codes.
- [ ] Mobile `tsc` (changed files clean); commit `feat(mobile): session-type-first booking with duration-aware slot picker`.

---

## Self-Review
- Coverage: durations+validation (U1), slot generation (U2), type gate + web UI (U3),
  mobile UI (U4) — all spec sections mapped.
- Types: `SessionType` + duration constants consistent across units; `instant` flag
  threaded controller→service.
- Parity: U3 web + U4 mobile mirror the same flow and endpoints.
