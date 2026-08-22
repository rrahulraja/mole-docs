# Epic C — Pricing (single-rate MVP + international charges) — Design Spec

**Date:** 2026-08-04
**Status:** Approved (design)
**Part of:** Booking/marketplace revamp (Epics D → C → B → A). C is second; depends on
Epic D's `Student.country` for the India-vs-not decision.

## Goal

1. **Single rate/hr for MVP** — educator enters one rate that applies to all session
   types. Segregated per-type rates (teaching/revision/doubt) are kept in code and
   **flag-gated** (global `segregated_rates_enabled`, default off) for later.
2. **International charges** — per educator, **admin-enabled**. Once enabled, the
   educator sets their international rate(s); a student whose location is outside India
   is charged those rates instead of the domestic ones.

## Locked decisions

- Segregated flag = **global** `PlatformConfig.segregated_rates_enabled` (default off).
- International rate = **INR** amount(s) set by the educator.
- Non-India student is **actually charged** the international rate (not display-only).
- Admin enabling is a **gate**: educator can set/see international rate fields only when
  `Educator.internationalPricingEnabled` is on for them.
- International rates are **symmetric** with domestic: single `intlRatePerHour` when the
  segregated flag is off, per-type intl rates when it is on.

## Data model (`Educator`)

Existing (keep): `rateTeaching`, `rateRevision`, `rateDoubtClearing` (Int?).

Add:
```prisma
ratePerHour                 Int?    @map("rate_per_hour")               // single MVP domestic rate
internationalPricingEnabled Boolean @default(false) @map("international_pricing_enabled")
intlRatePerHour             Int?    @map("intl_rate_per_hour")          // single intl rate
intlRateTeaching            Int?    @map("intl_rate_teaching")
intlRateRevision            Int?    @map("intl_rate_revision")
intlRateDoubtClearing       Int?    @map("intl_rate_doubt_clearing")
```
Migration + `pnpm generate:prisma` before typecheck.

`PlatformConfig` row `segregated_rates_enabled` = `"false"` seeded (read as `value === 'true'`).

## Rate resolver — `backend/src/services/booking/rate.ts`

Single source of truth for "what does this student pay this educator for this session
type". Used by `createBooking` and the extension `rateFor`.

```ts
export type SessionType = 'teaching' | 'revision' | 'doubt_clearing';

export interface RateContext {
  segregated: boolean;          // PlatformConfig segregated_rates_enabled
  studentCountry?: string | null; // Student.country (ISO-3166 alpha-2) or null
}

export function resolveRate(
  educator: {
    ratePerHour: number | null;
    rateTeaching: number | null; rateRevision: number | null; rateDoubtClearing: number | null;
    internationalPricingEnabled: boolean;
    intlRatePerHour: number | null;
    intlRateTeaching: number | null; intlRateRevision: number | null; intlRateDoubtClearing: number | null;
  },
  sessionType: SessionType,
  ctx: RateContext,
): number | null {
  const isForeign = !!ctx.studentCountry && ctx.studentCountry.toUpperCase() !== 'IN';
  const perType = (t: number | null, r: number | null, d: number | null) =>
    sessionType === 'teaching' ? t : sessionType === 'revision' ? r : d;

  const intl = ctx.segregated
    ? perType(educator.intlRateTeaching, educator.intlRateRevision, educator.intlRateDoubtClearing)
    : educator.intlRatePerHour;
  const domestic = ctx.segregated
    ? perType(educator.rateTeaching, educator.rateRevision, educator.rateDoubtClearing)
    : educator.ratePerHour;

  // International only when the student is foreign, admin enabled it, AND a rate exists;
  // otherwise fall back to domestic (never block a booking for a missing intl rate).
  if (isForeign && educator.internationalPricingEnabled && intl != null) return intl;
  return domestic;
}

export async function segregatedEnabled(): Promise<boolean> {
  const c = await prisma.platformConfig.findUnique({ where: { key: 'segregated_rates_enabled' } });
  return c?.value === 'true';
}
```

`createBooking` (`services/booking/index.ts`): fetch `student.country` +
`segregatedEnabled()`, call `resolveRate(educator, sessionType, { segregated, studentCountry })`,
throw `RATE_NOT_SET` if null. `totalPrice = round(rate × minutes / 60)` unchanged.
Extension `rateFor` (`booking.controller.ts`) uses the same resolver (fetch the booking's
student country + flag).

## Backend endpoints

- **Educator onboarding + profile update** accept `ratePerHour`, the three `rate*`, and
  (only when `internationalPricingEnabled`) `intlRatePerHour` + the three `intlRate*`.
  Intl fields sent while the toggle is off are **ignored** (not an error).
- **Educator `me`**: return `ratePerHour`, `rate*`, `internationalPricingEnabled`,
  `intlRatePerHour`, `intlRate*`, and the global `segregatedEnabled` flag so the rate
  form can render the right shape.
- **Public educator profile / discover cards**: return the **effective** rate for the
  requesting student (resolver with that student's country) — so a foreign student sees
  intl pricing and a domestic one sees domestic. Also return `segregatedEnabled` so the
  UI can show one price or a per-type breakdown.
- **Config**: expose `segregated_rates_enabled` via the existing `/config` surface the
  frontends already read (e.g. onboarding/config endpoint), so rate forms know the shape.
- **Admin**: `PATCH /admin/educators/:id/international-pricing` `{ enabled: boolean }` →
  sets `internationalPricingEnabled`. (Follows existing admin educator-mutation pattern.)

## Frontend (web + mobile parity)

- **Educator rate input** (onboarding + edit-profile, web + mobile):
  - `segregated` off → one **Rate per hour** field (`ratePerHour`).
  - `segregated` on → three fields (teaching/revision/doubt).
  - If `internationalPricingEnabled` → mirror the above with **International rate** fields
    (`intlRatePerHour` or the three `intlRate*`). Hidden entirely when the toggle is off,
    with a hint that admin must enable international pricing.
- **Student-facing** (educator card + profile, web + mobile): show the price the current
  student would pay (already resolved server-side). When `segregated` on, show the
  per-type breakdown; otherwise a single "₹X/hr".
- **Admin console** (`admin/`, Tailwind): per-educator **International pricing** toggle on
  the educator detail/list, calling the admin endpoint.

## Error handling / edge cases

- Missing rate for the resolved branch → `RATE_NOT_SET` (422), same as today.
- Intl enabled but educator hasn't set an intl rate → domestic rate used (no block).
- `Student.country` null (location denied in Epic D) → treated as India → domestic.
- Segregated flag flipped on with educators who only set `ratePerHour` → their per-type
  rates are null → `RATE_NOT_SET` until they fill them. (Acceptable: flag is an
  intentional platform-wide switch; a migration/backfill can copy `ratePerHour` into the
  three columns when flipping. Out of scope for MVP-off default.)

## Testing

- **Unit (`rate.test.ts`)**: domestic single, domestic segregated per-type, foreign+enabled
  returns intl, foreign+enabled+missing-intl falls back to domestic, foreign+disabled uses
  domestic, null country uses domestic, segregated intl per-type.
- **Manual**: educator sets single rate → domestic student quote correct; admin enables
  intl → educator sets intl rate → foreign student (country≠IN) quote uses intl.

## Rollout (units — issue→branch(main)→PR→merge→board)

1. Schema + `segregated_rates_enabled` config + `resolveRate`/`segregatedEnabled` +
   `createBooking`/`rateFor` wiring + unit tests.
2. Educator rate UI (onboarding + edit-profile), web + mobile — single/segregated shape.
3. Admin per-educator international toggle + educator international rate fields (web + mobile).
4. Student-facing effective-price display (card + profile), web + mobile.
