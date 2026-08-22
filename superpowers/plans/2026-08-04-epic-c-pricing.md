# Epic C — Pricing Implementation Plan

> **For agentic workers:** implement task-by-task. Steps use checkbox (`- [ ]`) syntax.

**Goal:** Single rate/hr MVP with flag-gated segregated per-type rates, plus per-educator
admin-gated international charges that a non-India student is actually billed.

**Architecture:** One `resolveRate` picks domestic-vs-intl (by student country + educator
toggle) then single-vs-segregated (by global flag). Educator gains `ratePerHour`, an intl
toggle, and single+segregated intl rate columns. Frontends render the rate shape from the
flag and show students their effective price.

**Tech Stack:** Express + Prisma · Next.js 14 + MUI · Expo/RN · React+Vite+Tailwind (admin) · `@mole/shared`.

## Global Constraints

- Base branch **main**; each unit issue→branch→PR→merge→Project board #2 Done.
- Web+mobile parity for every student/educator UI change; admin changes in `admin/`.
- Prisma: migrate + `pnpm generate:prisma` before typecheck.
- Commit as `Rrahul Raja <rahulraja.ngp@gmail.com>`; strip Claude trailers.
- Never commit `.env*`, `admin/vite.config.ts`, `docs/deployment/vps-deploy.md`.
- Depends on Epic D: `Student.country` (ISO-3166 alpha-2, null=domestic).
- Money = INR ints; `totalPrice = round(rate × minutes / 60)`.

---

## Unit 1 — Schema + rate resolver + booking wiring

**Files:**
- Modify: `backend/prisma/schema.prisma` (Educator) + migration
- Create: `backend/src/services/booking/rate.ts`
- Create: `backend/src/services/booking/rate.test.ts`
- Modify: `backend/src/services/booking/index.ts` (createBooking)
- Modify: `backend/src/controllers/booking.controller.ts` (rateFor + extension)
- Modify: seed/config for `segregated_rates_enabled`

**Produces:** `resolveRate`, `segregatedEnabled`; bookings priced through the resolver.

- [ ] Add `ratePerHour`, `internationalPricingEnabled`, `intlRatePerHour`, `intlRateTeaching/Revision/DoubtClearing` to `Educator`; migrate + generate.
- [ ] Seed/upsert `PlatformConfig` `segregated_rates_enabled = "false"`.
- [ ] Write `rate.test.ts` (7 cases from spec); run → FAIL.
- [ ] Implement `resolveRate` + `segregatedEnabled` in `rate.ts`; run test → PASS.
- [ ] `createBooking`: fetch `student.country` + `segregatedEnabled()`, price via `resolveRate`.
- [ ] Extension `rateFor`: fetch booking student country + flag, use `resolveRate`.
- [ ] Backend `typecheck` + full jest; commit `feat(backend): rate resolver (single/segregated + international) + booking wiring`.

## Unit 2 — Educator rate UI (single/segregated), web + mobile

**Files:**
- Modify: `backend/src/controllers/educator.controller.ts` (accept `ratePerHour` + return flags/rates in `me`; expose `segregatedEnabled`)
- Modify: mobile `app/(educator)/edit-profile.tsx` (+ onboarding rate step if present)
- Modify: web `app/educator/edit-profile/page.tsx` + `app/educator/profile/page.tsx`
- Reuse/create: a small rate-fields block per app

**Consumes:** Unit 1 fields + flag.

- [ ] Backend: educator onboarding/profile update accept `ratePerHour` (+ keep `rate*`); `me` returns `ratePerHour`, `rate*`, `segregatedEnabled`.
- [ ] Mobile educator rate UI: read flag → one "Rate/hr" or three fields; prefill; save.
- [ ] Web educator rate UI: same shape.
- [ ] Mobile tsc + web tsc (changed files clean); commit `feat(web,mobile): educator single/segregated rate input`.

## Unit 3 — Admin international toggle + educator intl rate fields

**Files:**
- Modify: `backend/src/controllers/admin.controller.ts` + `routes/v1/admin.ts` (`PATCH /admin/educators/:id/international-pricing`)
- Modify: `backend/src/controllers/educator.controller.ts` (accept `intlRatePerHour`/`intlRate*` only when enabled; return them + `internationalPricingEnabled` in `me`)
- Modify: `admin/src/pages/…` educator detail/list — toggle
- Modify: mobile + web educator edit-profile — intl rate fields (shown only when enabled)

**Consumes:** Unit 1 fields, Unit 2 UI.

- [ ] Backend admin endpoint sets `internationalPricingEnabled`; educator update accepts intl rates only when enabled (ignore otherwise).
- [ ] `me` returns intl fields + toggle.
- [ ] Admin UI per-educator International-pricing toggle.
- [ ] Mobile + web: intl rate fields (single/segregated mirror), visible only when enabled, with an admin-must-enable hint otherwise.
- [ ] Typechecks; commit `feat(admin,web,mobile): per-educator international pricing toggle + intl rate fields`.

## Unit 4 — Student-facing effective price

**Files:**
- Modify: `backend/src/controllers/discover.controller.ts` (card/profile return effective rate for requesting student via resolver) + educator public `getProfile`
- Modify: mobile + web educator card + profile price display

**Consumes:** Unit 1 resolver, Units 2–3 data.

- [ ] Backend: discover `formatEducator` + public profile compute effective rate using the requesting student's country + flag; return single price or per-type breakdown + `segregatedEnabled`.
- [ ] Mobile + web educator card/profile render the effective price (per-type when segregated).
- [ ] Typechecks + manual quote check (domestic vs foreign); commit `feat(web,mobile): show student their effective (domestic/international) price`.

---

## Self-Review
- Coverage: schema+resolver (U1), educator single/segregated UI (U2), admin toggle+intl fields (U3), student price display (U4) — all spec sections mapped.
- Types: `resolveRate`/`SessionType`/`RateContext` consistent across units.
- Parity: U2/U3/U4 list web + mobile (+ admin in U3).
