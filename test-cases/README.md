# Mole — Test Case Reference

Reference test cases across the whole platform, to be used as the source for future
**unit**, **integration**, and **e2e** tests. These describe *what* to verify; the actual
test code comes later.

## Documents

| File | Scope |
|---|---|
| [backend-api.md](./backend-api.md) | Every API domain: auth, discover, booking, payment, session (OTP/extend), review + moderation, educator, availability, admin, dispute, payout, referral, wallet, notification, recording, crons |
| [web.md](./web.md) | Web app (Next.js) — student + educator journeys, UI states, guards |
| [mobile.md](./mobile.md) | Mobile app (Expo/RN) — student + educator journeys, UI states |
| [admin.md](./admin.md) | Admin app (React/Vite) — verification, moderation, bookings, disputes, payouts, config |
| [e2e.md](./e2e.md) | Cross-surface end-to-end journeys (student ↔ educator ↔ admin) |

## Test case format

Each case uses this shape:

> **TC-`<AREA>`-`<NNN>`: `<short title>`**
> - **Type:** unit / integration / e2e
> - **Priority:** P0 (critical path) / P1 (important) / P2 (edge/nice-to-have)
> - **Preconditions:** state that must exist first
> - **Steps:** the actions to perform
> - **Expected:** the observable, assertable result (status code, DB state, UI state, navigation)

## ID scheme

`TC-<AREA>-<NNN>` — area prefixes:

`AUTH`, `DISC` (discover), `BOOK`, `PAY`, `SESS` (session/OTP/extend), `REV` (review),
`MOD` (review moderation), `EDU` (educator), `AVAIL` (availability), `ADM` (admin),
`DISP` (dispute), `PAYOUT`, `REF` (referral), `WALLET`, `NOTIF`, `REC` (recording),
`CRON`, `WEB`, `MOB`, `E2E`.

## Priority guidance

- **P0** — money, auth, booking lifecycle, session end/OTP, review visibility (moderation). A bug here blocks release.
- **P1** — discovery, availability, profiles, notifications, admin ops.
- **P2** — empty/loading states, formatting, cosmetic, rare edge cases.

## Conventions used in the cases

- **Roles:** student, educator, admin. Auth = phone + OTP (JWT); admin is **passwordless** — email OTP or emailed magic link, both issuing an 8h JWT (`admin_token`). Admins have no username and no password column.
- **Booking status:** `awaiting_educator → pending_payment → confirmed → completed`; plus `cancelled`, `failed`, `disputed`.
- **Review status:** `pending → approved | rejected` (only `approved` is public).
- **Educator status:** `pending → under_review → approved | rejected | suspended`. `under_review` is an *already-onboarded* educator who re-uploaded a video and needs re-approval; only `approved` is discoverable.
- **Dev helpers:** OTP mock accepts `123456` (non-prod); `POST /payment/:id/mock-confirm` confirms payment without a webhook (non-prod).
- **Video provider:** behind `VIDEO_PROVIDER` (`zoom` default). Session end-OTP / extend / review flows are provider-agnostic.
