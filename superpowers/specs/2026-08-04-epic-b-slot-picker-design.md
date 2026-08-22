# Epic B — Slot Picker + Minimum Durations Design

**Date:** 2026-08-04
**Status:** Approved (design decisions locked)
**Depends on:** Epic C pricing (rate resolver) — shipped. Epic D (`Student.country`) — shipped.

## Goal

Replace the fixed 60-minute slot list with a date + time picker that offers only
valid, overlap-free start times inside the educator's availability, enforce
per-session-type minimum durations, and validate both on the server so a booking
can never be created outside these rules.

Spec points covered: **#3** (date+time picker + overlap check + suggested slots)
and **#4** (min durations: Teaching 45, Revision 30, Doubt Clearing 30).

## Locked decisions

1. **Picker model** — student picks a date, then a start time on a **15-minute
   grid**. Every offered time must (a) fall inside the educator's availability
   window for that weekday, (b) leave room for the chosen duration before the
   window ends (`start + duration ≤ windowEnd`), and (c) not overlap an existing
   confirmed/pending session. The "suggested slots" are exactly this list for the
   **selected date** (no cross-day lookahead) — one list powers both.
2. **Durations per session type** (minutes):
   - Teaching: `45, 60, 90, 120` (min 45)
   - Revision: `30, 45, 60, 90` (min 30)
   - Doubt Clearing: `30, 45, 60` (min 30)

   Selector defaults to the type's minimum. Server **hard-rejects** any duration
   below the type's minimum.
3. **Suggestions** — only the currently selected date's free start times. Changing
   date or duration refetches.
4. **Session-type-first flow** — the student chooses the **session type first**.
   That choice fixes the type's minimum duration, which then drives everything
   downstream: the duration options offered, whether the educator is even
   bookable for that type (they must have at least one availability window long
   enough to fit the type's minimum), and the free start times. Time selection is
   the **last** step, so a student never picks a time the educator can't honour.

## Flow order (student booking)

1. **Session type** — Teaching / Revision / Doubt Clearing. A type is shown as
   available only if the educator offers a rate for it **and** has at least one
   availability window `≥ SESSION_MIN_MINUTES[type]` (the "can this educator do a
   session this long at all" gate — independent of any specific date).
2. **Duration** — options = `SESSION_DURATION_OPTIONS[type]`, default = the
   type's minimum.
3. **Date** — next N days.
4. **Time** — the 15-minute-grid free start times for that date + duration
   (§ availability rework). Picked last, always valid.

## Architecture

One shared source of truth for durations, one availability endpoint that returns
duration-aware free start times, and server-side validation in `createBooking`.

### Shared (`@mole/shared`)

New `constants/durations.ts`, exported from the package index:

```ts
export const SLOT_STEP_MINUTES = 15;

export const SESSION_MIN_MINUTES: Record<SessionType, number> = {
    teaching: 45,
    revision: 30,
    doubt_clearing: 30,
};

export const SESSION_DURATION_OPTIONS: Record<SessionType, number[]> = {
    teaching: [45, 60, 90, 120],
    revision: [30, 45, 60, 90],
    doubt_clearing: [30, 45, 60],
};
```

Backend, web, and mobile all import these — the duration options in the UI and the
minimum enforced by the server can never drift.

### Backend

**`getPublicAvailability` rework** (`educator.controller.ts`). Query gains
`duration` (minutes, required; validated ≥ 1) alongside `date`. For the selected
date's weekday availability window(s):

- Generate candidate starts on the 15-minute grid from `startTime`.
- Keep a start only if `start + duration ≤ windowEnd`.
- Drop starts whose `[start, start+duration)` interval overlaps any existing
  booking's `[scheduledAt, scheduledAt + durationMinutes)` for that educator
  (status in `confirmed`/`pending_payment`/`awaiting_educator`). This is a real
  interval-overlap test, not the current exact-start-time match.
- Drop past starts (today, with the existing 30-minute buffer).

Returns `{ slots: [{ time24 }] }` — same shape the clients already read, so only
the generation logic changes.

**`getEducatorProfile` gains `bookableSessionTypes`** (`discover.controller.ts`).
For the session-type-first gate, the profile response returns the list of session
types the educator can actually be booked for = the type has an effective rate for
this student (already computed) **and** the educator has at least one active
availability window whose length ≥ `SESSION_MIN_MINUTES[type]`. Computed from the
longest active window (`maxAvailabilityMinutes`) so it's one pass over the
schedule. The client renders only these types in step 1.

**`createBooking` validation** (`services/booking/index.ts`). Before creating:

- **Min duration** — `durationMinutes < SESSION_MIN_MINUTES[sessionType]` →
  throw `DURATION_TOO_SHORT`.
- **Overlap** — the requested `[scheduledAt, +duration)` overlaps an existing
  educator booking (same status set) → throw `SLOT_UNAVAILABLE`.
- **Availability window** — for **scheduled** bookings only, `scheduledAt` must
  fall inside an active availability window with `start+duration ≤ end` →
  else `OUTSIDE_AVAILABILITY`. **Instant** bookings (live-now "Book Now") skip
  this check: a live educator can take a session outside declared hours. The
  booking request gains an `instant: boolean` flag (default false) that the
  controller passes through; overlap + min-duration still apply to instant.

New error codes added to `shared/src/constants/errors.ts` and surfaced by the
booking controller with 409/400 as appropriate.

### Frontend (web + mobile, parity)

- Duration selector options come from `SESSION_DURATION_OPTIONS[sessionType]`;
  default to `SESSION_DURATION_OPTIONS[sessionType][0]` (the minimum). Switching
  session type resets the duration to that type's default.
- The slot fetch passes the chosen `duration`; the returned times are the
  pickable start times (chips). Changing date or duration refetches and clears
  the selected slot.
- On a booking error, show the mapped message (slot taken → "That time was just
  taken, pick another"; too short → guidance). Instant "Book Now" passes
  `instant: true`.

## Testing

- **Shared** — trivial constant module; covered indirectly by backend tests.
- **Backend `rate`-style unit tests** for the availability slot generator (a pure
  function extracted from the endpoint): window + duration → candidate starts;
  overlap filtering; window-end clamp; today buffer.
- **Backend `createBooking` validation tests** (prisma-mock): min-duration reject,
  overlap reject, outside-availability reject (scheduled), instant bypass of the
  window check.
- **Frontend** — no runner yet; typecheck gates only.

## Out of scope

- Live-now instant-accept flow, penalties, WhatsApp/voice accept window — that is
  **Epic A**. Epic B only adds the `instant` flag plumbing so the availability
  check can be skipped; it does not change instant booking behaviour otherwise.
- Rescheduling / editing an existing booking's time.
