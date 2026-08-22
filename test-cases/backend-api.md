# Mole — Backend API Test Cases

Reference test cases for the Mole backend (`code/backend`, Express + Prisma + TypeScript).
These describe *what* to verify; test code (jest + supertest) comes later.

Format and ID scheme follow [README.md](./README.md). Each case asserts HTTP status,
error `code` string, and/or DB state. Error responses use the shape `{ error: '<CODE>' }`
unless noted (some validation errors return a raw zod `flatten()` under `error` or under
`details`, flagged inline). Ownership failures deliberately differ across domains
(payment/session use **404**, booking/dispute use **403**) — cases assert the actual code.

**Conventions**
- Auth = phone + OTP → JWT (`accessToken` + `refreshToken`); temp token from a new-user OTP verify.
- Admin auth = **passwordless**: email OTP (`/admin/auth/request-otp` → `/admin/auth/verify-otp`) or magic link (`/admin/auth/request-magic` → `/admin/auth/verify-magic`), both issuing a JWT (`authenticateAdmin`, 8h). There is no `/admin/login` and no password column.
- Booking FSM: `awaiting_educator → pending_payment → confirmed → completed`; plus `cancelled`, `failed`, `disputed`.
- Review FSM: `pending → approved | rejected` (only `approved` is public / counts toward rating).
- Educator FSM: `pending → under_review → approved | rejected | suspended` (admin writes are unconditional — no source-state guard). `under_review` is reached automatically when an `approved`/`rejected` educator uploads a video; only `approved` is discoverable.
- Dev helpers: OTP `123456` valid in non-prod; `POST /payment/:id/mock-confirm` confirms without webhook (non-prod only).

---

## Test data & helpers

- **Mock OTP:** in non-prod the SMS provider accepts `123456` for any phone. Unknown phone on `verify` returns `{ isNewUser: true, tempToken }` (auto-registration path — not an error).
- **Mock payment confirm:** `POST /payment/:bookingId/mock-confirm` (non-prod only; `NODE_ENV=production` → 404) performs the same transaction as a webhook SUCCESS: payment→`success`, credits debit if `creditsApplied>0`, booking→`confirmed`, Zoom setup fired async. No ownership check.
- **Test DB:** `.env.test` → `NODE_ENV=test`, `DATABASE_URL=postgresql://postgres:postgres@localhost:5432/mole_test`.
- **Test runner:** `jest --runInBand` with `ts-jest` + `supertest` are installed in `package.json`; **no test files exist yet** — this doc seeds them.
- **Seed scripts:** `prisma/seed.ts` (topics/subjects/config) and `prisma/seed-admin.ts` (bootstrap admin user) — run before integration/e2e suites. `prisma/seed-test.ts` additionally seeds every other educator `approved` with photos, 1 intro + 2 teaching clips, so discovery/search has visible profiles.
- **Signatures:** UPI webhook uses HMAC-SHA256 over raw body, secret `UPI_WEBHOOK_SECRET` (fallback `test_secret`), header `x-webhook-signature`. Zoom webhook uses `v0:{ts}:{body}` HMAC, header `x-zm-signature`, secret `ZOOM_WEBHOOK_SECRET`.
- **Existing unit tests:** `src/crons/reminders.test.ts` covers `reminderTitle()` / `reminderWindows()` — the CRON cases below are written to agree with it.
- **Known bugs to pin (assert current behavior, link to fix tickets):** `getMyEarnings` net formula is wrong (negative net); several admin mutators throw Prisma `P2025` (→500) instead of a clean 404 on missing rows; `webhookHandler`/`mockConfirm` never verify the paid amount; `admin-auth.controller` hard-codes the admin OTP to `123456` (the `crypto.randomInt` line is commented out) — this must not ship to production.
  - *Fixed since the last revision — do not re-assert the old behaviour:* reminder crons now really send push (and were merged into one `session-reminders` job); the 5-min `no-show-detection` cron was deleted so it no longer pre-empts the 15-min escalation.

---

## Auth (`/auth`)

> **TC-AUTH-001: Send OTP — happy path**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** none
> - **Steps:** `POST /auth/otp/send` with `{ phone: '9990001111' }`.
> - **Expected:** 200 `{ message: 'OTP sent' }`; provider invoked with the phone.

> **TC-AUTH-002: Send OTP — phone too short → validation**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** none
> - **Steps:** `POST /auth/otp/send` with `{ phone: '123' }` (< 10 chars).
> - **Expected:** 400 `VALIDATION_ERROR` with `details` (zod flatten).

> **TC-AUTH-003: Send OTP — missing phone → validation**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** none
> - **Steps:** `POST /auth/otp/send` with `{}`.
> - **Expected:** 400 `VALIDATION_ERROR`.

> **TC-AUTH-004: Verify OTP — existing user returns tokens**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** user exists with phone `P`.
> - **Steps:** `POST /auth/otp/verify` `{ phone: P, code: '123456' }`.
> - **Expected:** 200 `{ accessToken, refreshToken, isNewUser: false, user }`; a `refreshToken` row created with `expiresAt ≈ now+30d`.

> **TC-AUTH-005: Verify OTP — new phone returns tempToken**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** no user with phone `P`.
> - **Steps:** `POST /auth/otp/verify` `{ phone: P, code: '123456' }`.
> - **Expected:** 200 `{ isNewUser: true, tempToken }`; NO `accessToken`/`user`; no user row created yet.

> **TC-AUTH-006: Verify OTP — wrong code → invalid**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** none
> - **Steps:** `POST /auth/otp/verify` `{ phone: P, code: '000000' }`.
> - **Expected:** 401 `INVALID_OTP`.

> **TC-AUTH-007: Verify OTP — code not 6 chars → validation**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** none
> - **Steps:** `POST /auth/otp/verify` `{ phone: P, code: '123' }`.
> - **Expected:** 400 `VALIDATION_ERROR` (code must be length 6).

> **TC-AUTH-008: Verify OTP — does not check isActive**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** user with `isActive=false`.
> - **Steps:** verify with valid OTP.
> - **Expected:** 200 with tokens (verify does NOT gate on `isActive`; disabling is enforced at `GET /me`).

> **TC-AUTH-009: Refresh — happy path rotates token**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** a valid non-expired refresh token `R`.
> - **Steps:** `POST /auth/refresh` `{ refreshToken: R }`.
> - **Expected:** 200 `{ accessToken, refreshToken: R2 }`; old row deleted, new row created with the SAME `expiresAt` (not extended).

> **TC-AUTH-010: Refresh — unknown token → invalid**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** none
> - **Steps:** `POST /auth/refresh` `{ refreshToken: 'garbage' }`.
> - **Expected:** 401 `INVALID_REFRESH_TOKEN`.

> **TC-AUTH-011: Refresh — expired token → invalid**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** refresh token row with `expiresAt < now`.
> - **Steps:** `POST /auth/refresh` with it.
> - **Expected:** 401 `INVALID_REFRESH_TOKEN` (filter requires `expiresAt > now`).

> **TC-AUTH-012: Refresh — missing body → validation**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** none
> - **Steps:** `POST /auth/refresh` `{}`.
> - **Expected:** 400 `VALIDATION_ERROR`.

> **TC-AUTH-013: Logout — deletes token, idempotent**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** valid refresh token `R`.
> - **Steps:** `POST /auth/logout` `{ refreshToken: R }`; repeat.
> - **Expected:** 200 `{ message: 'Logged out' }` both times; row gone after first call; second call still 200 (idempotent).

> **TC-AUTH-014: Logout — missing body → validation**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** `POST /auth/logout` `{}`.
> - **Expected:** 400 `VALIDATION_ERROR`.

> **TC-AUTH-015: Update FCM token — happy path**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** authenticated user.
> - **Steps:** `PUT /auth/fcm-token` `{ fcmToken: 'abc' }` with bearer token.
> - **Expected:** 200 `{ message: 'FCM token updated' }`; user.fcmToken persisted.

> **TC-AUTH-016: Update FCM token — no auth → 401**
> - **Type:** integration
> - **Priority:** P0
> - **Steps:** `PUT /auth/fcm-token` `{ fcmToken: 'abc' }` without bearer.
> - **Expected:** 401 (authenticate middleware).

> **TC-AUTH-017: Update FCM token — missing field → validation**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** `PUT /auth/fcm-token` `{}` authenticated.
> - **Expected:** 400 `VALIDATION_ERROR`.

> **TC-AUTH-018: Get me — happy path shape**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** active user.
> - **Steps:** `GET /auth/me` authenticated.
> - **Expected:** 200 with `{ id, name, phone, email, role, referralCode, educatorStatus, hasCompletedIdentity, gradeLevel, examBoards, onboardingSeen }`.

> **TC-AUTH-019: Get me — user missing → 404**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** valid JWT for a userId whose row was deleted.
> - **Steps:** `GET /auth/me`.
> - **Expected:** 404 `USER_NOT_FOUND`.

> **TC-AUTH-020: Get me — disabled account → 403**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** user with `isActive=false`.
> - **Steps:** `GET /auth/me`.
> - **Expected:** 403 `ACCOUNT_DISABLED`.

> **TC-AUTH-021: Get me — no auth → 401**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `GET /auth/me` without bearer.
> - **Expected:** 401.

> **TC-AUTH-022: Accept terms — records version**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** authenticated user; config `terms_version` set (else default `1.0`).
> - **Steps:** `POST /auth/terms/accept`.
> - **Expected:** 200 `{ message: 'Terms accepted', version }`; `termsAcceptedAt`/`termsAcceptedVersion` set.

> **TC-AUTH-023: Terms status — before/after accept**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** authenticated user who has not accepted.
> - **Steps:** `GET /auth/terms/status`; accept; `GET` again.
> - **Expected:** first `{ accepted: false, ... }`; after accept `accepted: true` with `acceptedVersion`/`acceptedAt`.

> **TC-AUTH-024: Terms accept/status — no auth → 401**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** call either without bearer.
> - **Expected:** 401.

---

## Discover (`/discover`) — all routes `authenticate`

> **TC-DISC-001: Live educators — happy path**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** approved educators, some `isLiveNow=true`.
> - **Steps:** `GET /discover/live` authenticated.
> - **Expected:** 200 `{ educators: [...] }`; only `status='approved'` AND `isLiveNow=true`; ordered by rating desc, ≤20; each item has `{ id, name, isLive, rating, rateTeaching, rateRevision, rateDoubtClearing, bio, topics(≤3 approved), photo, introClipUrl }`.

> **TC-DISC-002: Live educators — no auth → 401**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `GET /discover/live` without bearer.
> - **Expected:** 401.

> **TC-DISC-003: Live educators — board filter applied**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** caller is a student with `examBoards=['CBSE']`; educators with/without CBSE.
> - **Steps:** `GET /discover/live`.
> - **Expected:** 200; results restricted to educators matching board (`hasSome`). If board lookup errors it silently returns unfiltered (never 500).

> **TC-DISC-004: Suggested — rating ≥ 4 with board filter, capped at 6**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** ≥10 approved educators with ratings ≥4 for the student's board, plus some <4.
> - **Steps:** `GET /discover/suggested`.
> - **Expected:** 200; only `overallRating >= 4` matching board, `overallRating desc`, **exactly 6 returned** (`SUGGESTED_LIMIT`, was 8).

> **TC-DISC-005: Suggested — fallback shows approved educators before ratings exist**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** a fresh platform — approved educators exist for the student's board but none has reached a 4 rating (ratings may be 0/null).
> - **Steps:** `GET /discover/suggested`.
> - **Expected:** 200 and **not empty**. The fallback keeps the board filter and drops only the rating floor, ordering `[overallRating desc, referralCount desc]`, ≤6. This is the case that used to return an empty Suggested rail on a new install.

> **TC-DISC-006: Search — by topic query `q`**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** approved educators with topic names.
> - **Steps:** `GET /discover/search?q=calculus`.
> - **Expected:** 200; matches on topic name contains (case-insensitive); only approved.

> **TC-DISC-007: Search — subject filter**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** `GET /discover/search?subject=Mathematics`.
> - **Expected:** 200; filtered by topic.subject.name equals (insensitive).

> **TC-DISC-008: Search — live_now=true**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** `GET /discover/search?live_now=true`.
> - **Expected:** 200; only `isLiveNow=true`. Any value other than literal `'true'` does not filter.

> **TC-DISC-009: Search — min_rating is exclusive (gt)**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** educator with rating exactly 4.0.
> - **Steps:** `GET /discover/search?min_rating=4`.
> - **Expected:** 200; the 4.0 educator is EXCLUDED (uses `> `, not `>=`).

> **TC-DISC-010: Search — sort param is a no-op**
> - **Type:** unit
> - **Priority:** P2
> - **Steps:** `GET /discover/search?sort=rating` vs `sort=whatever`.
> - **Expected:** both order by `overallRating desc` (sort currently ignored). Document as known behavior.

> **TC-DISC-011: Search — limit caps results**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** `GET /discover/search?limit=5` with >5 matches.
> - **Expected:** 200; ≤5 returned (default 30 for a real search).

> **TC-DISC-018: Search — default browse is capped at 6, a real search opens up**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** ≥20 approved educators, ≥10 of them teaching a topic matching `calculus`.
> - **Steps:** `GET /discover/search` with no params; then `GET /discover/search?q=calculus`; then `?subject=Mathematics`; then `?live_now=true`; then `?min_rating=4`.
> - **Expected:** the bare browse returns **at most 6** — an empty Find-an-educator page is a curated shortlist, not a dump of every approved educator. Any of `q`, `subject`, `live_now=true` or `min_rating` sets `hasQuery` and raises the cap to 30. An explicit `limit` still overrides both.

> **TC-DISC-019: Trending topics — most-booked in the last 30 days**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** bookings across several topics inside the last 30 days (with a clear ordering by count) plus older bookings on a different topic.
> - **Steps:** `GET /discover/trending-topics`.
> - **Expected:** 200 `{ topics: [{ id, name }] }` grouped by `topicId` over `createdAt >= now-30d`, ordered by booking count desc. Bookings older than 30 days do not influence the ranking. Topics that are no longer `isActive` are dropped from the result rather than returned with a missing name.

> **TC-DISC-020: Trending topics — count from `trending_topics_count`, clamped**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** more than 30 distinct booked topics.
> - **Steps:** call with the config unset; then `trending_topics_count = 5`; then `0`; then `999`.
> - **Expected:** unset → 10; `5` → 5; the value is clamped to `[1, 30]`, so `0` yields 1 and `999` yields 30. The key is on the admin config whitelist, so this is editable from the admin Config page.

> **TC-DISC-021: Trending topics — fallback when there are no recent bookings**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** a fresh platform with active topics but zero bookings in the last 30 days.
> - **Steps:** `GET /discover/trending-topics`.
> - **Expected:** 200 with a non-empty list of active topics (up to the same limit) — the student home's search pills must never render empty just because nothing has been booked yet.

> **TC-DISC-022: Trending topics — requires auth**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** `GET /discover/trending-topics` without a bearer token.
> - **Expected:** 401 — the route sits behind `authenticate` like the rest of `/discover`.

> **TC-DISC-012: Educator profile — happy path**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** approved educator with approved + pending reviews.
> - **Steps:** `GET /discover/:id`.
> - **Expected:** 200 `{ educator: {...reviewCount, ratingBreakdown[5..1], topics(all), demoClips(≤5)...}, reviews }`; `reviews` and `reviewCount` count ONLY `status='approved'`; `ratingBreakdown` covers stars 5→1 with 0-fills.

> **TC-DISC-013: Educator profile — pending reviews NOT visible**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** educator with a `pending` review containing a comment.
> - **Steps:** `GET /discover/:id`.
> - **Expected:** 200; the pending review is absent from `reviews` and excluded from `reviewCount`/`ratingBreakdown`.

> **TC-DISC-014: Educator profile — not found / not approved → 404**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** unknown id, or an educator with `status='pending'`.
> - **Steps:** `GET /discover/:id`.
> - **Expected:** 404 `EDUCATOR_NOT_FOUND` (non-approved treated as not found).

> **TC-DISC-015: Public availability — by date**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** approved educator with active availability rows.
> - **Steps:** `GET /discover/:id/availability?date=2026-07-15`.
> - **Expected:** 200 `{ educator, dates:[7 days with hasSlots], schedule }`.

> **TC-DISC-016: Public availability — educator not found → 404**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** `GET /discover/:unknown/availability`.
> - **Expected:** 404 `EDUCATOR_NOT_FOUND`.

> **TC-DISC-017: Public availability — invalid date string**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** `GET /discover/:id/availability?date=not-a-date`.
> - **Expected:** currently throws (Invalid Date → toISOString) → 500. Assert and file a fix (should be 400).

---

## Booking (`/bookings`)

> **TC-BOOK-001: Create booking — happy path (student)**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** student authed; approved educator with rate set for the sessionType; valid topic; a matching availability slot.
> - **Steps:** `POST /bookings` `{ educatorId, topicId, sessionType:'teaching', durationMinutes:60, scheduledAt:<ISO>, creditsApplied:0 }`.
> - **Expected:** 201 `{ booking }`; booking created with status `awaiting_educator`.

> **TC-BOOK-002: Create booking — no auth → 401**
> - **Type:** integration
> - **Priority:** P0
> - **Steps:** `POST /bookings` without bearer.
> - **Expected:** 401.

> **TC-BOOK-003: Create booking — non-student role → 403**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** educator-role token.
> - **Steps:** `POST /bookings` valid body.
> - **Expected:** 403 (`requireRole('student')`).

> **TC-BOOK-004: Create booking — bad payload → validation**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `POST /bookings` with `educatorId` not a UUID / missing topicId / `sessionType:'foo'` / `durationMinutes:20`.
> - **Expected:** 400 `VALIDATION_ERROR` with `details` (zod flatten).

> **TC-BOOK-005: Create booking — duration bounds (30–120)**
> - **Type:** unit
> - **Priority:** P1
> - **Steps:** try `durationMinutes` = 29, 30, 120, 121.
> - **Expected:** 30 and 120 pass validation; 29 and 121 → 400 `VALIDATION_ERROR`.

> **TC-BOOK-006: Create booking — sessionType enum**
> - **Type:** unit
> - **Priority:** P2
> - **Steps:** try `teaching`, `revision`, `doubt_clearing`, and `group`.
> - **Expected:** first three valid; `group` → 400 `VALIDATION_ERROR`.

> **TC-BOOK-007: Create booking — educator unavailable → 422**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** requested `scheduledAt` has no matching active slot / slot already booked.
> - **Steps:** `POST /bookings`.
> - **Expected:** 422 `EDUCATOR_UNAVAILABLE`; no booking row.

> **TC-BOOK-008: Create booking — rate not set → 422**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** educator has no rate for the requested sessionType.
> - **Steps:** `POST /bookings`.
> - **Expected:** 422 `RATE_NOT_SET`.

> **TC-BOOK-009: Create booking — invalid topic → 422**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** topicId not owned/approved for the educator.
> - **Steps:** `POST /bookings`.
> - **Expected:** 422 `INVALID_TOPIC`.

> **TC-BOOK-010: Get booking — participant happy path**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** booking `B`; caller is its student or educator.
> - **Steps:** `GET /bookings/:id`.
> - **Expected:** 200 `{ booking }` with educator, student, topic, payment includes.

> **TC-BOOK-011: Get booking — not found → 404**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `GET /bookings/<unknown>`.
> - **Expected:** 404 `NOT_FOUND`.

> **TC-BOOK-012: Get booking — non-participant → 403**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** booking `B`; caller is a third-party user.
> - **Steps:** `GET /bookings/:id`.
> - **Expected:** 403 `FORBIDDEN` (checked after NOT_FOUND).

> **TC-BOOK-013: List my bookings (student) — status filter (CSV)**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** student with bookings in mixed statuses.
> - **Steps:** `GET /bookings/my?status=confirmed,completed`.
> - **Expected:** 200 `{ bookings }` filtered to those statuses (CSV → `in`); each has `hasReview` bool, educator `photoUrl`; ≤50, `scheduledAt asc`.

> **TC-BOOK-014: List my bookings — no auth / wrong role**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `GET /bookings/my` unauth; then with educator token.
> - **Expected:** 401 unauth; 403 for educator role.

> **TC-BOOK-015: List educator bookings — status filter (single)**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** educator with bookings.
> - **Steps:** `GET /bookings/educator/my?status=confirmed`.
> - **Expected:** 200; filtered by single status (no CSV support here); ≤50, `scheduledAt desc`.

> **TC-BOOK-016: List educator bookings — student role → 403**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `GET /bookings/educator/my` with student token.
> - **Expected:** 403.

> **TC-BOOK-017: Cancel — awaiting_educator by student**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** booking in `awaiting_educator`, >1h before scheduledAt; caller is student.
> - **Steps:** `PATCH /bookings/:id/cancel`.
> - **Expected:** 200 `{ success: true }`; status → `cancelled`; no refund (was not confirmed).

> **TC-BOOK-018: Cancel — confirmed by student refunds wallet**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** booking `confirmed`, >1h out; caller is the student.
> - **Steps:** `PATCH /bookings/:id/cancel`.
> - **Expected:** 200; status → `cancelled`; wallet credited `amountDue`; walletTransaction `type=credit, reference=cancellation_refund:<id>`.

> **TC-BOOK-019: Cancel — confirmed by educator does NOT refund**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** booking `confirmed`; caller is the educator.
> - **Steps:** `PATCH /bookings/:id/cancel`.
> - **Expected:** 200; status → `cancelled`; NO wallet credit (refund only when canceller is the student).

> **TC-BOOK-020: Cancel — not found → 404**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** cancel unknown id.
> - **Expected:** 404 `NOT_FOUND`.

> **TC-BOOK-021: Cancel — non-participant → 403**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** cancel a booking as a third party.
> - **Expected:** 403 `FORBIDDEN`.

> **TC-BOOK-022: Cancel — terminal status → 422**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** booking `completed` (or cancelled/failed/disputed).
> - **Steps:** cancel.
> - **Expected:** 422 `CANNOT_CANCEL` (allowed only from awaiting_educator/pending_payment/confirmed).

> **TC-BOOK-023: Cancel — within 1 hour → 422**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** cancelable booking with `scheduledAt` < 60 min away.
> - **Steps:** cancel.
> - **Expected:** 422 `TOO_LATE_TO_CANCEL`.

> **TC-BOOK-024: Respond (accept) via JWT link**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** booking `awaiting_educator`; signed token with `{ bookingId, action:'accept' }`.
> - **Steps:** `GET /bookings/:id/respond?token=<jwt>`.
> - **Expected:** 200 HTML ("Session Accepted"); status → `pending_payment`; Payment row created `{ amount:amountDue, status:'pending' }`; student push.

> **TC-BOOK-025: Respond (decline) via JWT link**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** booking `awaiting_educator`; token action `decline`.
> - **Steps:** `GET /bookings/:id/respond?token=<jwt>`.
> - **Expected:** 200 HTML ("Session Declined"); status → `cancelled`.

> **TC-BOOK-026: Respond — token bookingId mismatch → 400**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** token whose `bookingId` != path `:id`.
> - **Steps:** `POST /bookings/:id/respond` with token.
> - **Expected:** 400 `TOKEN_MISMATCH`.

> **TC-BOOK-027: Respond — invalid/expired token → 400**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** respond with a malformed token.
> - **Expected:** 400 `INVALID_TOKEN`.

> **TC-BOOK-028: Respond — no token and no action → 400**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** `POST /bookings/:id/respond` with `{}`.
> - **Expected:** 400 `MISSING_TOKEN_OR_ACTION`.

> **TC-BOOK-029: Respond — booking not found → 404**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** respond to unknown id with a valid-shape token.
> - **Expected:** 404 `NOT_FOUND`.

> **TC-BOOK-030: Respond — wrong status → 422**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** booking already `pending_payment`/`confirmed`.
> - **Steps:** respond accept.
> - **Expected:** 422 `INVALID_STATUS` (only `awaiting_educator` respondable).

> **TC-BOOK-031: Respond (webhook body) — JSON response**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** booking `awaiting_educator`.
> - **Steps:** `POST /bookings/:id/respond` `{ action:'accept' }` (no query token).
> - **Expected:** 200 JSON `{ success:true, status:'pending_payment' }` (HTML branch only when `req.query.token` present).

> **TC-BOOK-032: Extend — cost quote happy path**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** booking `confirmed`; caller is student; rate set for sessionType.
> - **Steps:** `GET /bookings/:id/extend?minutes=30`.
> - **Expected:** 200 `{ extensionCost, upiUri, transactionRef, extensionMinutes:30 }`; `transactionRef` prefixed `MOLE-EXT-`; no status change.

> **TC-BOOK-033: Extend — invalid minutes → 400**
> - **Type:** unit
> - **Priority:** P1
> - **Steps:** `GET /bookings/:id/extend?minutes=45` (and missing param).
> - **Expected:** 400 `INVALID_MINUTES` (allowed 30/60/90).

> **TC-BOOK-034: Extend — not found → 404**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** `GET /bookings/<unknown>/extend?minutes=30`.
> - **Expected:** 404 `NOT_FOUND` (after minutes validation).

> **TC-BOOK-035: Extend — non-student participant → 403**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** caller is the educator of the booking.
> - **Steps:** `GET /bookings/:id/extend?minutes=30`.
> - **Expected:** 403 `FORBIDDEN` (only the student may extend).

> **TC-BOOK-036: Extend — not in session → 422**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** booking not `confirmed`.
> - **Steps:** `GET /bookings/:id/extend?minutes=30`.
> - **Expected:** 422 `NOT_IN_SESSION`.

> **TC-BOOK-037: Extend — rate not set → 422**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** confirmed booking whose sessionType rate is null.
> - **Steps:** extend.
> - **Expected:** 422 `RATE_NOT_SET`.

> **TC-BOOK-038: Confirm extension — updates duration**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** confirmed booking; caller student.
> - **Steps:** `POST /bookings/:id/extend/confirm` `{ extensionMinutes:30 }`.
> - **Expected:** 200 `{ success:true, newDuration }` = old `durationMinutes+30`; `durationMinutes` persisted.

> **TC-BOOK-039: Confirm extension — validation/ownership/status errors**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** confirm with minutes=45 (→400 `INVALID_MINUTES`); as educator (→403 `FORBIDDEN`); unknown id (→404 `NOT_FOUND`); non-confirmed (→422 `NOT_IN_SESSION`).
> - **Expected:** codes as noted. NOTE (bug): confirm does not verify the extension was actually paid — file a ticket.

---

## Payment (`/payment`)

> **TC-PAY-001: Create intent — happy path**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** booking `pending_payment` owned by the caller (student).
> - **Steps:** `POST /payment/:bookingId/intent`.
> - **Expected:** 200 `{ upiUri, amount, bookingId, transactionRef }`; `transactionRef` prefixed `MOLE-`; Payment upserted `status='pending'`, `amount=amountDue`.

> **TC-PAY-002: Create intent — no auth → 401**
> - **Type:** integration
> - **Priority:** P0
> - **Steps:** `POST /payment/:id/intent` without bearer.
> - **Expected:** 401 (router-level `authenticate`).

> **TC-PAY-003: Create intent — not owner / not found → 404**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** booking owned by a different student.
> - **Steps:** create intent.
> - **Expected:** 404 `NOT_FOUND` (ownership failure returns 404, not 403).

> **TC-PAY-004: Create intent — already confirmed → 409**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** booking `confirmed`.
> - **Steps:** create intent.
> - **Expected:** 409 `ALREADY_CONFIRMED`.

> **TC-PAY-005: Create intent — wrong status → 422**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** booking `awaiting_educator` (not pending_payment, not confirmed).
> - **Steps:** create intent.
> - **Expected:** 422 `INVALID_STATUS`.

> **TC-PAY-006: Payment status — happy path**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** booking with payment; caller is student or educator.
> - **Steps:** `GET /payment/:bookingId/status`.
> - **Expected:** 200 `{ status:(pending|success|failed), bookingStatus, scheduledAt, durationMinutes }`.

> **TC-PAY-007: Payment status — not participant / not found → 404**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** status as a third party.
> - **Expected:** 404 `NOT_FOUND` (unauthorized returns 404).

> **TC-PAY-008: Retry — happy path**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** booking `pending_payment` with existing Payment row; caller is owner.
> - **Steps:** `POST /payment/:bookingId/retry`.
> - **Expected:** 200 `{ transactionRef }`; Payment reset to `status='pending'` with new ref.

> **TC-PAY-009: Retry — cannot retry → 422**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** booking not `pending_payment`.
> - **Steps:** retry.
> - **Expected:** 422 `CANNOT_RETRY`.

> **TC-PAY-010: Retry — not owner → 404**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** retry another student's booking.
> - **Expected:** 404 `NOT_FOUND`.

> **TC-PAY-011: Mock-confirm — non-prod confirms**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** `NODE_ENV != production`; booking `pending_payment` with a Payment row.
> - **Steps:** `POST /payment/:bookingId/mock-confirm`.
> - **Expected:** 200 `{ message:'Payment confirmed (mock)', bookingId }`; Payment → `success`; booking → `confirmed`; credits debited if `creditsApplied>0`.

> **TC-PAY-012: Mock-confirm — idempotent**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** payment already `success`.
> - **Steps:** mock-confirm again.
> - **Expected:** 200 `{ message:'Already confirmed' }`; no double-debit.

> **TC-PAY-013: Mock-confirm — production disabled → 404**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** `NODE_ENV=production`.
> - **Steps:** mock-confirm.
> - **Expected:** 404 `NOT_FOUND`.

> **TC-PAY-014: Mock-confirm — no payment row → 404**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** booking without a Payment row (non-prod).
> - **Steps:** mock-confirm.
> - **Expected:** 404 `PAYMENT_NOT_FOUND`.

> **TC-PAY-015: Webhook SUCCESS → booking confirmed**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Payment `pending` with known `transaction_ref`; booking `pending_payment`.
> - **Steps:** `POST /payment/webhook` with valid HMAC `x-webhook-signature`, body `{ transaction_ref, status:'SUCCESS', amount, upi_ref_id }`.
> - **Expected:** 200 `{ message:'Processed' }`; Payment → `success` (`webhookReceivedAt` set); booking → `confirmed`; wallet debited if `creditsApplied>0` (`reference=booking:<id>`).

> **TC-PAY-016: Webhook FAILURE → booking failed**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Payment `pending`.
> - **Steps:** webhook `status:'FAILURE'`, valid signature.
> - **Expected:** 200 `{ message:'Processed' }`; Payment → `failed`; booking → `failed`.

> **TC-PAY-017: Webhook — bad signature → 401**
> - **Type:** integration
> - **Priority:** P0
> - **Steps:** webhook with wrong/absent `x-webhook-signature`.
> - **Expected:** 401 `INVALID_SIGNATURE`; no DB mutation (signature checked before JSON.parse).

> **TC-PAY-018: Webhook — unknown transaction_ref**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** valid signature, `transaction_ref` matching no payment.
> - **Expected:** 200 `{ message:'Unknown transaction' }`; no mutation.

> **TC-PAY-019: Webhook — idempotent (already processed)**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Payment already `success` or `failed`.
> - **Steps:** replay webhook.
> - **Expected:** 200 `{ message:'Already processed' }`; no state change / no double-debit.

> **TC-PAY-020: Webhook — no amount verification (known gap)**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** SUCCESS webhook with `amount` != Payment.amount, valid signature.
> - **Expected:** currently still confirms (amount not compared). Assert and file a hardening ticket.

---

## Session / OTP (`/session`) — all `authenticate`

> **TC-SESS-001: Get token — confirmed booking**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** booking `confirmed`; caller is student or educator.
> - **Steps:** `GET /session/:bookingId/token`.
> - **Expected:** 200 `{ sessionToken, sessionName:<bookingId>, roleType(educator=1|student=0) }`; token expiry 7200s.

> **TC-SESS-002: Get token — not found → 404**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** unknown bookingId.
> - **Expected:** 404 `NOT_FOUND`.

> **TC-SESS-003: Get token — non-participant → 403**
> - **Type:** integration
> - **Priority:** P0
> - **Steps:** get token as a third party.
> - **Expected:** 403 `FORBIDDEN`.

> **TC-SESS-004: Get token — booking not confirmed → 422**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** booking `pending_payment` (or any non-confirmed).
> - **Steps:** get token.
> - **Expected:** 422 `SESSION_NOT_ACTIVE`.

> **TC-SESS-005: Join — records timestamps**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** confirmed booking; only the educator has joined.
> - **Steps:** student `POST /session/:bookingId/join`.
> - **Expected:** 200; `studentJoinedAt` set (idempotent if already set).

> **TC-SESS-006: Join — both joined generates end-OTP**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** confirmed booking; the other party already joined.
> - **Steps:** second party joins.
> - **Expected:** 200; `sessionEndOtp` = 6-digit; `sessionEndOtpExpiresAt = now + (durationMinutes+15) min`; OTP push sent to the student only; OTP returned in the response ONLY when the caller is the student (educator gets `null`).

> **TC-SESS-007: Join — not found → 404**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** join unknown bookingId.
> - **Expected:** 404 `NOT_FOUND`.

> **TC-SESS-008: Join — non-participant → 403**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** join as a third party.
> - **Expected:** 403 `FORBIDDEN`.

> **TC-SESS-009: End — educator with valid OTP → completed**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** confirmed booking; both joined; OTP generated and unexpired.
> - **Steps:** educator `POST /session/:bookingId/end` `{ otp:<correct> }`.
> - **Expected:** 200 `{ message:'Session ended' }`; booking → `completed`.

> **TC-SESS-010: End — student ends without OTP**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** confirmed booking; caller is the student.
> - **Steps:** `POST /session/:bookingId/end` `{}` (no otp).
> - **Expected:** 200; booking → `completed` (OTP gate applies only to the educator).

> **TC-SESS-011: End — educator OTP not generated yet → 422**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** confirmed booking; both parties NOT yet joined (no `sessionEndOtp`); caller educator.
> - **Steps:** educator ends with any otp.
> - **Expected:** 422 `OTP_NOT_GENERATED_YET`.

> **TC-SESS-012: End — educator missing OTP → 422**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** OTP generated; caller educator; body omits `otp`.
> - **Steps:** educator ends `{}`.
> - **Expected:** 422 `OTP_REQUIRED`.

> **TC-SESS-013: End — educator wrong OTP → 422**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** OTP generated; educator supplies wrong otp.
> - **Steps:** end `{ otp:'000000' }`.
> - **Expected:** 422 `INVALID_OTP` (checked before expiry).

> **TC-SESS-014: End — educator expired OTP → 422**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** correct OTP but `sessionEndOtpExpiresAt < now`.
> - **Steps:** educator ends with the correct-but-expired otp.
> - **Expected:** 422 `OTP_EXPIRED`.

> **TC-SESS-015: End — booking not confirmed → 422**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** booking already `completed`/other.
> - **Steps:** end.
> - **Expected:** 422 `SESSION_NOT_ACTIVE`.

> **TC-SESS-016: End — unrelated user → 404 (not 403)**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** authenticated third party.
> - **Steps:** `POST /session/:bookingId/end`.
> - **Expected:** 404 `NOT_FOUND` (endSession returns 404 for non-participants, unlike token/join).

> **TC-SESS-017: End — not found → 404**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** end unknown bookingId.
> - **Expected:** 404 `NOT_FOUND`.

---

## Review (`/review`)

> **TC-REV-001: Submit review — happy path (pending)**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** booking `completed`; caller is its student; no existing review.
> - **Steps:** `POST /review` `{ bookingId, rating:5, comment:'Great' }`.
> - **Expected:** 201 review with `status='pending'`; `overallRating` on the educator is UNCHANGED (recompute deferred to approval).

> **TC-REV-002: Submit review — not visible on discover until approved**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** a pending review from TC-REV-001.
> - **Steps:** `GET /discover/:educatorId`.
> - **Expected:** the pending review is absent from `reviews` / `reviewCount`.

> **TC-REV-003: Submit review — validation**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `POST /review` with missing bookingId / `rating:6` / `rating:0` / `comment` >1000 chars.
> - **Expected:** 400 with raw zod flatten under `error` (no `VALIDATION_ERROR` string). Rating must be int 1–5.

> **TC-REV-004: Submit review — abusive comment → 422**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** completed booking.
> - **Steps:** submit with an abusive `comment`.
> - **Expected:** 422 `ABUSIVE_CONTENT` (checked before booking lookup); no review created.

> **TC-REV-005: Submit review — booking not found → 404**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** submit with unknown bookingId.
> - **Expected:** 404 `BOOKING_NOT_FOUND`.

> **TC-REV-006: Submit review — not the student → 403**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** booking whose student is someone else.
> - **Steps:** submit.
> - **Expected:** 403 `FORBIDDEN`.

> **TC-REV-007: Submit review — booking not completed → 422**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** booking `pending_payment` (in prod). Note: `confirmed` is allowed only when `NODE_ENV != production`; `completed`/`disputed` always allowed.
> - **Steps:** submit.
> - **Expected:** 422 `BOOKING_NOT_COMPLETED`.

> **TC-REV-008: Submit review — duplicate → 409**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** booking already has a review.
> - **Steps:** submit again.
> - **Expected:** 409 `REVIEW_ALREADY_EXISTS`.

> **TC-REV-009: Submit review — no auth → 401**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `POST /review` without bearer.
> - **Expected:** 401.

> **TC-REV-010: Get educator reviews — approved only, paginated**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** educator with approved + pending + rejected reviews.
> - **Steps:** `GET /review/educator/:educatorId?limit=10`.
> - **Expected:** 200 `{ items, nextCursor }`; only `status='approved'` items, `createdAt desc`; cursor pagination.

> **TC-REV-011: Report educator — happy path**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** authenticated user; educator id.
> - **Steps:** `POST /review/report` `{ educatorId, reason:'no_show', details:'...' }`.
> - **Expected:** 201 `{ message: 'Report submitted...' }`; `educatorReport` row created; admins notified.

> **TC-REV-012: Report — reason enum validation**
> - **Type:** unit
> - **Priority:** P1
> - **Steps:** report with `reason:'spam'` (not in enum) / missing educatorId / details >2000.
> - **Expected:** 400 (raw zod flatten). Valid reasons: `inappropriate_behavior`, `harassment`, `fraud`, `no_show`, `other`.

> **TC-REV-013: Report — abusive details → 422**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** report with abusive `details`.
> - **Expected:** 422 `ABUSIVE_CONTENT` (no `message` field, unlike review).

> **TC-REV-014: Report — limit reached → 429**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** reporter already has 3 reports for the same educator.
> - **Steps:** submit a 4th.
> - **Expected:** 429 `REPORT_LIMIT_REACHED` (limit is 3 per reporter per educator).

---

## Review Moderation (`/admin/reviews`)

> **TC-MOD-001: List reviews — filter by status**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** admin token; reviews in mixed statuses.
> - **Steps:** `GET /admin/reviews?status=pending`.
> - **Expected:** 200 `{ reviews:[...] }` with `studentName`/`educatorName`/`topicName` defaults; only pending. Default status is `pending`; an invalid status silently returns ALL (no 400).

> **TC-MOD-002: Approve review → recompute rating**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** a pending review with rating; admin token.
> - **Steps:** `PUT /admin/reviews/:id/review` `{ action:'approve' }`.
> - **Expected:** 200 `{ success:true }`; review → `approved`, `moderatedAt` set; educator `overallRating` recomputed as avg over `approved` reviews; audit `approve_review`.

> **TC-MOD-003: Approved review now visible on discover**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** review approved in TC-MOD-002.
> - **Steps:** `GET /discover/:educatorId`.
> - **Expected:** review appears in `reviews`; `reviewCount` incremented; rating reflects it.

> **TC-MOD-004: Reject review → not counted**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** pending review.
> - **Steps:** `PUT /admin/reviews/:id/review` `{ action:'reject' }`.
> - **Expected:** 200; review → `rejected`; NOT visible on discover; rating unaffected by it.

> **TC-MOD-005: Moderate — invalid action → 400**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `{ action:'delete' }`.
> - **Expected:** 400 `INVALID_ACTION`.

> **TC-MOD-006: Moderate — review not found → 404**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** unknown review id.
> - **Expected:** 404 `NOT_FOUND`.

> **TC-MOD-007: Moderation — no admin auth → 401**
> - **Type:** integration
> - **Priority:** P0
> - **Steps:** list/moderate without admin token.
> - **Expected:** 401 (`authenticateAdmin`).

---

## Educator (`/educator`)

> **TC-EDU-001: Onboarding step1 — happy path**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** valid temp token (from new-user OTP verify); no user for that phone.
> - **Steps:** `POST /educator/onboarding/step1` with full step1 body.
> - **Expected:** 201 `{ accessToken, refreshToken, user }`; user role `educator` + educator row `status='pending'` created.

> **TC-EDU-002: Step1 — user already exists → 409**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** a user already exists for the temp token's phone.
> - **Steps:** `POST /educator/onboarding/step1`.
> - **Expected:** 409 `USER_ALREADY_EXISTS` (checked before body validation).

> **TC-EDU-003: Step1 — validation**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** step1 with `bio` <10 chars / `subjectsInterested` empty or >2 / `qualifications.year` out of range.
> - **Expected:** 400 `VALIDATION_ERROR` with `details`.

> **TC-EDU-004: Step1 — no temp token → 401**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** step1 without temp token.
> - **Expected:** 401 (`authenticateTempToken`).

> **TC-EDU-005: Update step1 (PATCH) — partial**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** educator token.
> - **Steps:** `PATCH /educator/onboarding/step1` `{ bio:'updated bio ...' }`.
> - **Expected:** 200 `{ ok:true }`; validation applies to present fields only.

> **TC-EDU-006: Step2 identity — happy path**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** authenticated educator.
> - **Steps:** `POST /educator/onboarding/step2` `{ govtIdUrl, selfieUrl, educationProofUrl }`.
> - **Expected:** 200 `{ message:'Identity details saved' }`.

> **TC-EDU-007: Step2 — validation**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** step2 missing `govtIdUrl`/`selfieUrl`/`educationProofUrl`.
> - **Expected:** 400 `VALIDATION_ERROR`.

> **TC-EDU-008: Get educator me — happy path**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** educator with bookings/reviews.
> - **Steps:** `GET /educator/me`.
> - **Expected:** 200 `{ educator, todaySchedule, recentReviews }`; `recentReviews` approved-only (≤5); `todaySchedule` today's confirmed/completed; bank account masked.

> **TC-EDU-009: Get educator me — not found → 404**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** token whose educator row is missing.
> - **Steps:** `GET /educator/me`.
> - **Expected:** 404 `EDUCATOR_NOT_FOUND`.

> **TC-EDU-010: Get educator me — student role → 403**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `GET /educator/me` with student token.
> - **Expected:** 403.

> **TC-EDU-011: Go-live toggle**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** educator token.
> - **Steps:** `PATCH /educator/go-live` `{ isLive:true }`; then `false`.
> - **Expected:** 200 `{ isLive:true }` then `{ isLive:false }`; `isLiveNow` persisted.

> **TC-EDU-012: Go-live — non-boolean → 400**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** `{ isLive:'yes' }`.
> - **Expected:** 400 `VALIDATION_ERROR` (no `details` on this handler).

> **TC-EDU-013: Update profile — happy path**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `PATCH /educator/profile` `{ bio:'new bio ...', rateTeaching:500, topicIds:[uuid] }`.
> - **Expected:** 200 `{ ok:true }`; when `topicIds` provided, educatorTopics recreated with `status='pending_review'`.

> **TC-EDU-014: Update profile — validation (bio ≤500)**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** `PATCH /educator/profile` with `bio` >500 or `rateTeaching` negative.
> - **Expected:** 400 `VALIDATION_ERROR`.

> **TC-EDU-015: Setup bank — happy path**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `POST /educator/setup-bank` `{ bankAccountName, bankAccountNumber(≥8), bankIfscCode(11 chars), bankAccountType:'Savings' }`.
> - **Expected:** 200 `{ message:'Payout details saved' }`.

> **TC-EDU-016: Setup bank — validation (IFSC length, type enum)**
> - **Type:** unit
> - **Priority:** P2
> - **Steps:** IFSC not 11 chars / `bankAccountType:'Checking'` / account number <8.
> - **Expected:** 400 `VALIDATION_ERROR`.

> **TC-EDU-017: Get bank details — happy path & role guard**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** `GET /educator/me/bank-details` (educator → 200 fields or `{}`); with non-educator role → 403 `FORBIDDEN`.
> - **Expected:** codes as noted.

> **TC-EDU-018: Update bank details — IFSC regex**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** `PATCH /educator/me/bank-details` with `bankIfscCode:'ABC1234'` (fails `/^[A-Z]{4}0[A-Z0-9]{6}$/`).
> - **Expected:** 400 (raw zod flatten). Wrong role first → 403 `FORBIDDEN`.

> **TC-EDU-019: Delete bank details**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** `DELETE /educator/bank`.
> - **Expected:** 200 `{ ok:true }`; all four bank fields nulled.

The video rules are: **one** intro video, **at most three** teaching (`demo`) videos, and
**five minutes maximum per video**. The backend is the enforcement point; web and mobile
both pre-check duration client-side so the user gets a friendly error before the upload.

> **TC-EDU-020: Add demo clip — happy path & re-review**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** an `approved` educator.
> - **Steps:** `POST /educator/demo-clips` `{ storageKey, durationSeconds:120, type:'demo' }`.
> - **Expected:** 201 `{ clip }` with `type:'demo'`; educator status → **`under_review`** (not `pending`), which hides them from discovery until an admin re-approves. A `rejected` educator also moves to `under_review`; a still-onboarding `pending` educator stays `pending`; `suspended` is untouched.

> **TC-EDU-021: Add demo clip — missing fields → 400**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** post without `storageKey`; then without `durationSeconds`; then with `durationSeconds: 0`.
> - **Expected:** 400 `MISSING_FIELDS` for the first two. `durationSeconds: 0` is **accepted** — some clients can't read duration up front, so zero is allowed through and the client-side guard is what catches long files there.

> **TC-EDU-022: Add teaching video — 4th is rejected → 422**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** educator already has 3 `demo` clips.
> - **Steps:** add a 4th `demo` clip.
> - **Expected:** 422 `CLIP_LIMIT_REACHED` with message "You can upload a maximum of 3 teaching videos."; no clip row created; educator status unchanged. The limit counts **per type**, so 3 teaching videos plus an intro is a valid state.

> **TC-EDU-026: Intro video is replaced, not rejected**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** educator with exactly one `intro` clip.
> - **Steps:** `POST /educator/demo-clips` `{ storageKey, durationSeconds:60, type:'intro' }`.
> - **Expected:** 201; the previous intro row is deleted first, so the educator ends with exactly **one** intro clip (the new one). Never 422 — the intro is a slot, not a quota. The educator still moves to `under_review`.

> **TC-EDU-027: Video longer than 5 minutes → 422**
> - **Type:** integration
> - **Priority:** P0
> - **Steps:** `POST /educator/demo-clips` `{ storageKey, durationSeconds:301, type:'demo' }`; repeat with `type:'intro'`; then post exactly `300`.
> - **Expected:** 422 `VIDEO_TOO_LONG` with message "Each video must be 5 minutes or shorter." for both types at 301s; **no** clip row is created and the educator's status is **not** changed (the length check runs before anything is written). Exactly 300s is accepted — the bound is inclusive.

> **TC-EDU-028: Unknown clip type falls back to `demo`**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** post with `type:'promo'`, and with `type` omitted.
> - **Expected:** 201 in both cases with the clip stored as `type:'demo'` (valid types are `demo`, `intro`, `other`) — so it counts against the 3-video teaching limit, not the intro slot.

> **TC-EDU-023: Delete demo clip — ownership**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** `DELETE /educator/demo-clips/:id` for own clip (200 `{ success:true }`); for another educator's clip / unknown → 404 `NOT_FOUND`.
> - **Expected:** codes as noted.

> **TC-EDU-024: Public educator profile (`GET /:id`)**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** approved educator.
> - **Steps:** `GET /educator/:id`.
> - **Expected:** 200 with `verified`, subjects→topics, rates, approved demoClips, `sessionCount`; only topics in `approved`/`pending_review`.

> **TC-EDU-025: Public educator profile — not approved → 404**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `GET /educator/:id` for a pending/unknown educator.
> - **Expected:** 404 `NOT_FOUND`.

---

## Availability (`/educator/me/availability`, `/educator/:id/availability`)

> **TC-AVAIL-001: Get my availability — grouped by day**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** educator with active slots.
> - **Steps:** `GET /educator/me/availability`.
> - **Expected:** 200 map `{ [dayOfWeek]: [{ id, startTime, endTime }] }`; only `isActive=true`.

> **TC-AVAIL-002: Add slot — happy path**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `POST /educator/me/availability` `{ dayOfWeek:1, startTime:'09:00', endTime:'11:00' }`.
> - **Expected:** 201 with the created slot (local↔UTC conversion applied).

> **TC-AVAIL-003: Add slot — bad format → 400**
> - **Type:** unit
> - **Priority:** P2
> - **Steps:** `startTime:'9am'` / `dayOfWeek:7`.
> - **Expected:** 400 (raw zod flatten; `dayOfWeek` 0–6, time `HH:MM`).

> **TC-AVAIL-004: Add slot — end ≤ start → 422**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `{ dayOfWeek:1, startTime:'11:00', endTime:'09:00' }`.
> - **Expected:** 422 `END_BEFORE_START` (endTime must be strictly after startTime).

> **TC-AVAIL-005: Add slot — overlap → 409**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** existing active slot 09:00–11:00 on day 1.
> - **Steps:** add 10:00–12:00 on day 1.
> - **Expected:** 409 `SLOT_OVERLAP`.

> **TC-AVAIL-006: Delete slot — ownership (soft delete)**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** `DELETE /educator/me/availability/:slotId` for own slot (200 `{ message:'Slot removed' }`, `isActive=false`); another's / unknown → 404 `NOT_FOUND`.
> - **Expected:** codes as noted.

> **TC-AVAIL-007: Availability — no auth / wrong role**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** any `/me/availability` route without bearer / with student token.
> - **Expected:** 401 unauth; 403 wrong role.

> **TC-AVAIL-008: Public availability by date — happy path**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** approved educator with slots on the target weekday.
> - **Steps:** `GET /educator/:id/availability?date=2026-07-15`.
> - **Expected:** 200 `{ slots:[...] }`; 60-min slots between start/end; booked slots (`confirmed`/`pending_payment`) filtered; today's past slots (≤ now+30min UTC buffer) excluded; `{ slots: [] }` when none.

> **TC-AVAIL-009: Public availability — missing date → 400**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** `GET /educator/:id/availability` without `date`.
> - **Expected:** 400 `{ error:'date query param required' }` (plain string, not a code).

---

## Admin (`/admin`) — `authenticateAdmin`

Admin sign-in is passwordless. Four unauthenticated routes sit under `/admin/auth/*`;
everything else is behind `authenticateAdmin`.

> **TC-ADM-001: Admin email OTP — request then verify**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** active admin (seed-admin) with a known email.
> - **Steps:** `POST /admin/auth/request-otp` `{ email }`; read the emitted code; `POST /admin/auth/verify-otp` `{ email, code }`.
> - **Expected:** request 200 with the generic body `{ message:'If an admin account exists for that email, a sign-in message has been sent.' }`, one `adminAuthToken` row `type='otp'` storing only the **sha256 hash** (never the code), `expiresAt = now+10min`; verify 200 `{ token, admin:{ id, name, email } }`, JWT `expiresIn:8h`, and the token row's `usedAt` stamped.

> **TC-ADM-002: Admin OTP — email normalisation and re-request invalidation**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** admin with email `ops@mole.test`.
> - **Steps:** request an OTP with `'  OPS@MOLE.TEST '`; request a second OTP for the same admin; verify with the **first** code.
> - **Expected:** the address is lower-cased/trimmed so the admin is found; the second request `deleteMany`s the prior unused `otp` rows, leaving exactly one; the first code now → 401 `INVALID_CODE`.

> **TC-ADM-003: Admin OTP — bad / expired / reused code → 401**
> - **Type:** integration
> - **Priority:** P0
> - **Steps:** verify with a wrong code; verify with a code whose `expiresAt < now`; verify twice with the same valid code.
> - **Expected:** 401 `INVALID_CODE`; 401 `CODE_EXPIRED`; second use → 401 `INVALID_CODE` (the lookup filters `usedAt: null`).

> **TC-ADM-034: Admin OTP — unknown or deactivated email never leaks**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** one address with no admin, one address belonging to an `isActive=false` admin.
> - **Steps:** `POST /admin/auth/request-otp` for each; then `POST /admin/auth/verify-otp` with any code.
> - **Expected:** request returns **200 with the same generic message** in both cases (no 404, no timing/shape difference) and creates no token row and sends no email; verify → 401 `INVALID_CODE`.

> **TC-ADM-035: Admin magic link — request then verify**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** active admin; `ADMIN_APP_URL` configured.
> - **Steps:** `POST /admin/auth/request-magic` `{ email }`; take the raw token from the emailed `…/login?token=<raw>`; `POST /admin/auth/verify-magic` `{ token }`.
> - **Expected:** request 200 generic body; one `adminAuthToken` row `type='magic'`, 32 random bytes hashed with sha256, `expiresAt = now+15min`, prior unused magic rows deleted; verify 200 `{ token, admin }` and `usedAt` stamped.

> **TC-ADM-036: Admin magic link — invalid / expired / reused / deactivated → 401**
> - **Type:** integration
> - **Priority:** P0
> - **Steps:** verify a made-up token; an expired one; the same token twice; a valid token whose admin was deactivated between request and verify.
> - **Expected:** 401 `INVALID_TOKEN`; 401 `TOKEN_EXPIRED`; second use 401 `INVALID_TOKEN`; deactivated admin 401 `INVALID_TOKEN` (checked *after* the token matches, so `usedAt` is not stamped).

> **TC-ADM-037: Admin auth — missing fields → 400**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `request-otp` `{}`; `verify-otp` `{ email }` only; `verify-magic` `{}`.
> - **Expected:** 400 `{ error:'email required' }`; 400 `{ error:'email and code required' }`; 400 `{ error:'token required' }`.

> **TC-ADM-038: `/admin/login` no longer exists**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `POST /admin/login` `{ username, password }`.
> - **Expected:** 404 — the password route was removed with the `username`/`password` columns. Any client still calling it is a regression.

> **TC-ADM-004: Any admin route — no token → 401**
> - **Type:** integration
> - **Priority:** P0
> - **Steps:** `GET /admin/educators` without token.
> - **Expected:** 401.

> **TC-ADM-005: List educators — filter + pagination**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `GET /admin/educators?status=pending&q=john&page=1`.
> - **Expected:** 200 `{ educators, total, page, pages }` (limit 50; `q` matches name/email/phone).

> **TC-ADM-006: Get educator — presigned URLs / not found**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `GET /admin/educators/:id` known (200 with presigned selfie/govtId/proof) and unknown (404 `NOT_FOUND`).
> - **Expected:** codes as noted; presign failure falls back to raw URLs (still 200).

> **TC-ADM-007: Approve educator → approved + referralCode**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** pending educator.
> - **Steps:** `PUT /admin/educators/:id/approve`.
> - **Expected:** 200 `{ success:true, referralCode }`; status → `approved`; approval notification; audit `approve_educator`.

> **TC-ADM-008: Approve — not found → 404**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** approve unknown id.
> - **Expected:** 404 `NOT_FOUND`.

> **TC-ADM-009: Reject educator → rejected + reason**
> - **Type:** integration
> - **Priority:** P0
> - **Steps:** `PUT /admin/educators/:id/reject` `{ reason:'...' }`.
> - **Expected:** 200 `{ success:true }`; status → `rejected`, `rejectionReason` set; audit.

> **TC-ADM-010: Reject — missing reason → 400**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** reject with `{}`.
> - **Expected:** 400 `{ error:'reason required' }`.

> **TC-ADM-011: Suspend educator**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `PUT /admin/educators/:id/suspend`.
> - **Expected:** 200; status → `suspended`, `isLiveNow=false`; audit.

> **TC-ADM-012: Reactivate educator → approved**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** suspended educator.
> - **Steps:** `PUT /admin/educators/:id/reactivate`.
> - **Expected:** 200; status → `approved` (unconditional; no FSM source guard). NOTE: rejected/pending also jump to approved — file a ticket if unintended.

> **TC-ADM-013: Educator mutators — missing row → 500 (known gap)**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** reject/suspend/reactivate an unknown id.
> - **Expected:** currently Prisma `P2025` → 500 (no existence guard). Assert and file a fix (should be 404).

> **TC-ADM-014: Review clip — approve/reject**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `PUT /admin/clips/:clipId/review` `{ action:'approve' }` / `reject`.
> - **Expected:** 200; clip status updated; audit. `INVALID_ACTION` (400) on bad action; `NOT_FOUND` (404) on unknown clip.

> **TC-ADM-015: List educator topics + review**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** `GET /admin/educator-topics?status=pending_review`; `PUT /admin/educator-topics/:id` `{ action:'approve' }`.
> - **Expected:** 200; topic status → approved/rejected. Bad action → 400 `INVALID_ACTION`; unknown → 404 `NOT_FOUND`.

> **TC-ADM-016: List students + deactivate**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `GET /admin/students?q=...`; `PUT /admin/students/:id/deactivate`.
> - **Expected:** list 200 `{ students, total, page, pages }`; deactivate 200 `{ success:true }`, `isActive=false`; unknown student → 404 `NOT_FOUND`.

> **TC-ADM-017: Grant wallet credit — happy path**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** existing student.
> - **Steps:** `POST /admin/wallet/grant` `{ studentId, amount:100, note }`.
> - **Expected:** 201 `{ success:true }`; wallet balance +100; walletTransaction `type=credit, reference=admin_grant:<adminId>`; audit.

> **TC-ADM-018: Grant — missing fields → 400 / student not found → 404**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** grant `{ studentId }` (no amount) → 400; grant for unknown studentId → 404.
> - **Expected:** 400 `{ error:'studentId and amount required' }` (note `amount:0` also fails); 404 `STUDENT_NOT_FOUND`.

> **TC-ADM-019: Topics/subjects CRUD**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** `GET /admin/topics`; `POST /admin/subjects` `{ name }`; `POST /admin/topics` `{ subjectId, name }`; `PUT /admin/topics/:id` `{ isActive:false }`; `DELETE /admin/topics/:id`.
> - **Expected:** 200 list; 201 subject/topic; 200 update/delete (soft delete `isActive=false`). Missing `name`/`subjectId` → 400.

> **TC-ADM-020: List bookings + pending payments**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `GET /admin/bookings?status=confirmed&from=&to=&page=1`; `GET /admin/bookings/pending-payment`.
> - **Expected:** 200 `{ bookings, total, page, pages }`; pending-payment returns `pending_payment` bookings older than 15 min.

> **TC-ADM-021: Resolve payment — confirm/fail**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** booking `pending_payment`.
> - **Steps:** `PUT /admin/payments/:bookingId/resolve` `{ status:'confirmed' }`.
> - **Expected:** 200 `{ success:true }`; booking → `confirmed`, payment → `success` (or both `failed` for `status:'failed'`); audit.

> **TC-ADM-022: Resolve payment — invalid status → 400**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `{ status:'refunded' }`.
> - **Expected:** 400 `{ error:'status must be confirmed or failed' }`.

> **TC-ADM-023: List disputes + resolve (with refund)**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** disputed booking.
> - **Steps:** `GET /admin/disputes`; `PUT /admin/disputes/:id/resolve` `{ resolutionNote, refund:true, refundAmount:100, refundType:'credits' }`.
> - **Expected:** list 200; resolve 200; booking → `completed`; wallet +100 (`reference=dispute_refund:<bookingId>`); audit.

> **TC-ADM-024: Resolve dispute — validation & state**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** resolve without `resolutionNote` (→400); resolve unknown id (→404 `NOT_FOUND`); resolve a non-disputed booking (→422 `NOT_IN_DISPUTE`).
> - **Expected:** codes as noted.

> **TC-ADM-025: Dismiss dispute**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** disputed booking.
> - **Steps:** `PUT /admin/disputes/:id/dismiss`.
> - **Expected:** 200; booking → `completed`, no refund; unknown → 404 `NOT_FOUND`; non-disputed → 422 `NOT_IN_DISPUTE`.

> **TC-ADM-026: Payout endpoints are mounted under admin auth**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** call `GET /admin/payouts/pending`, `GET /admin/payouts/history`, `POST /admin/payouts`, `POST /admin/payouts/:id/settle` without an admin token.
> - **Expected:** 401 on each. Behavioural cases live in the [Payout](#payout-adminpayouts) section below.

> **TC-ADM-027: Verify educator documents**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** educator with an `educationProofUrl` and ≥1 `demoClip` in `pending`.
> - **Steps:** `PUT /admin/educators/:id/verify-documents`; then repeat for an unknown id.
> - **Expected:** 200 `{ success:true, educationProofVerifiedAt }`; in one transaction the educator's `educationProofVerifiedAt` is stamped **and every one of that educator's demoClips flips to `approved`**; audit `verify_documents`. Unknown id → 404 `NOT_FOUND`. Note it does **not** change `status` — approval stays a separate action.

> **TC-ADM-028: Config — get / update / unknown key**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `GET /admin/config`; `PUT /admin/config/platform_commission_pct` `{ value:15 }`; `PUT /admin/config/not_a_key` `{ value:1 }`.
> - **Expected:** get 200 `{ configs }`; valid key 200 `{ success:true }` (upserted as `String(value)`, audited); unknown key 400 `{ error:'unknown config key' }`.

> **TC-ADM-029: Analytics — stats / sessions / revenue**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `GET /admin/analytics/stats`; `/analytics/sessions`; `/analytics/revenue`.
> - **Expected:** 200 each. stats has totals + `revenueThisMonth`; sessions has `completionRate`/`noShowRate`(disputed/total)/`byType`/`sessionsPerDay`; revenue has `grossRevenue`/`commissionEarned` (commission default 20% if config missing).

> **TC-ADM-030: Admins — list / create / self-deactivate guard**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `GET /admin/admins`; `POST /admin/admins` `{ name, email }`; `DELETE /admin/admins/:selfId`.
> - **Expected:** list 200 `{ admins }` (`id, email, name, isActive, createdAt, createdBy.name`, `createdAt asc`) — **no password field is ever returned or stored**; create 201 `{ admin:{ id, email, name } }` with `createdById` set to the acting admin, audited `create_admin`; the new admin has no credential and signs in via email OTP / magic link; deactivating own id → 400 `{ error:'Cannot deactivate yourself' }`.

> **TC-ADM-031: Create admin — email taken → 409 / bad input → 400**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** create with an existing email (in different casing/whitespace); create with `{ name }` only; create with `{ name, email:'not-an-email' }`.
> - **Expected:** 409 `EMAIL_TAKEN` (email is lower-cased + trimmed before the uniqueness check, so `Ops@Mole.test ` collides with `ops@mole.test`); 400 `{ error:'name and email required' }`; 400 `{ error:'invalid email' }`.

> **TC-ADM-039: Admin notification feed derived from live counts**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** ≥1 `pending` educator, ≥1 `under_review` educator, ≥1 `pending` review, ≥1 `disputed` booking, ≥1 educator with `pendingPayout > 0`.
> - **Steps:** `GET /admin/notifications`.
> - **Expected:** 200 `{ items, total }`. There is **no stored notifications table** — items are computed per request. Items carry `{ key, count, title, link }` with keys `pending_educators` → `/educators?status=pending`, `under_review_educators` → `/educators?status=under_review`, `pending_reviews` → `/reviews`, `open_disputes` → `/disputes`, `payouts_due` → `/payouts`; titles pluralise on `count===1` ("1 educator awaiting approval" vs "2 educators…"); `total` is the **sum of counts**, not the item count.

> **TC-ADM-040: Admin notification feed — zero-count buckets are omitted**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** a clean queue (nothing pending, no disputes, no payouts due).
> - **Steps:** `GET /admin/notifications`.
> - **Expected:** 200 `{ items: [], total: 0 }` — buckets with `count === 0` are filtered out rather than returned as empty rows.

> **TC-ADM-032: Audit log**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** `GET /admin/audit-log`.
> - **Expected:** 200 `{ logs }` (≤200, `createdAt desc`, with admin name). Prior mutating actions produced rows.

> **TC-ADM-033: Admin get recording**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** `GET /admin/recordings/:bookingId` for a `ready` recording (200 `{ url, s3Key, durationSecs, recordedAt }`, presigned 1h) and a non-ready/unknown one (404 `NOT_FOUND`).
> - **Expected:** codes as noted.

---

## Payout (`/admin/payouts`) — `authenticateAdmin`

Mole has **no bank or UPI integration**. Nothing in the codebase can move money. The
money-shaped states are therefore split in two:

- **queued** — a `PayoutHistory` row with `status:'pending'`, `paidAt:null`, no reference.
  Written by the `auto-payout` cron. `Educator.pendingPayout` is **untouched**.
- **paid** — `status:'paid'`, `paidAt` stamped, `reference` recorded, and only *then*
  `pendingPayout` decremented. Only an admin who has actually transferred the money can
  produce this state, via `POST /admin/payouts/:id/settle` (settling a queued row) or
  `POST /admin/payouts` (a direct, unqueued payout).

The central invariant for these cases: **`pendingPayout` may only ever be decremented by
an explicit admin action, exactly once per payout row.**

> **TC-PAYOUT-001: Pending balances list**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** approved educator with `pendingPayout>0`; an approved educator with `pendingPayout=0`; a non-approved educator with `pendingPayout>0`.
> - **Steps:** `GET /admin/payouts/pending`.
> - **Expected:** 200 `{ educators }` containing only `status:'approved'` educators with `pendingPayout > 0`, ordered `pendingPayout desc`, each with `user:{ name, email }`.

> **TC-PAYOUT-002: Payout history ordering includes queued rows**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** a mix of `pending` (queued, `paidAt=null`) and `paid` rows.
> - **Steps:** `GET /admin/payouts/history`.
> - **Expected:** 200 `{ payouts }`, ≤100, ordered **`createdAt desc`** — deliberately not `paidAt`, because queued rows have a null `paidAt` and would sort unpredictably. Each row includes `educator.user.name`. Queued and paid rows both appear; callers separate them by `status`.

> **TC-PAYOUT-003: Direct payout — happy path**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** educator with `pendingPayout = 5000`.
> - **Steps:** `POST /admin/payouts` `{ educatorId, amount:5000, reference:'UTR123456' }`.
> - **Expected:** 201 `{ success:true }`; in **one transaction** a `PayoutHistory` row `status:'paid'`, `type:'session_earnings'`, `paidAt` set, `reference` recorded, and `pendingPayout` → 0; audit `trigger_payout`.

> **TC-PAYOUT-004: Direct payout — validation**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `{ educatorId }` only; then `{ educatorId, amount:0, reference:'X' }`; then `{ educatorId, amount:'abc', reference:'X' }`; then an unknown `educatorId`.
> - **Expected:** 400 `{ error:'educatorId, amount, reference required' }`; 400 `{ error:'amount must be a positive number' }` for both the zero and the non-finite amount; 404 `EDUCATOR_NOT_FOUND`.

> **TC-PAYOUT-005: Direct payout — amount above balance → 422**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** educator with `pendingPayout = 1000`.
> - **Steps:** `POST /admin/payouts` `{ educatorId, amount:1500, reference:'UTR1' }`.
> - **Expected:** 422 `AMOUNT_EXCEEDS_PENDING` with a `message` naming both amounts; **no** `PayoutHistory` row; `pendingPayout` still 1000 (the balance can never be driven negative).

> **TC-PAYOUT-006: Settle a queued payout**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** educator with `pendingPayout = 3000` and a queued row `{ status:'pending', amount:3000, paidAt:null }`.
> - **Steps:** `POST /admin/payouts/:id/settle` `{ reference:'UTR999', note:'settled by ops' }`.
> - **Expected:** 200 `{ success:true }`; the row → `status:'paid'`, `reference:'UTR999'`, `note:'settled by ops'`, `paidAt` stamped; educator `pendingPayout` → 0; a "Payout processed" push + `Notification` row for the educator (deep link `mole://educator/earnings`); audit `settle_payout`.

> **TC-PAYOUT-007: Settle — reference is mandatory**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** a queued row.
> - **Steps:** `POST /admin/payouts/:id/settle` with `{}`, then with `{ note:'x' }`.
> - **Expected:** 400 `{ error:'reference required' }` both times; the row stays `pending`; `pendingPayout` unchanged. A settled payout without a bank reference is untraceable, so this must never succeed.

> **TC-PAYOUT-008: Settle — omitted note preserves the queue note**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** a queued row created by the cron with `note = 'Auto payout run 2026-07-20'`.
> - **Steps:** settle with `{ reference:'UTR1' }` and no `note`.
> - **Expected:** 200; `note` still reads `'Auto payout run 2026-07-20'` (it falls back to the existing value rather than nulling it), so the audit trail back to the run survives.

> **TC-PAYOUT-009: Settle — already settled → 409**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** a row already `status:'paid'`.
> - **Steps:** `POST /admin/payouts/:id/settle` `{ reference:'UTR2' }`.
> - **Expected:** 409 `ALREADY_SETTLED`; `paidAt`/`reference` unchanged; `pendingPayout` **not** decremented a second time.

> **TC-PAYOUT-010: Settle — concurrent settles decrement exactly once**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** educator with `pendingPayout = 2000` and one queued row of 2000.
> - **Steps:** fire two `POST /admin/payouts/:id/settle` requests concurrently (two admins clicking Settle at the same time).
> - **Expected:** exactly one 200 and one 409 `ALREADY_SETTLED`; `pendingPayout` ends at 0, never −2000. The guard is the `updateMany where { id, status:'pending' }` claim — a `claimed.count === 0` result must short-circuit **before** the decrement.

> **TC-PAYOUT-011: Settle — unknown ids → 404**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** settle a non-existent payout id; then settle a valid row whose educator record was deleted.
> - **Expected:** 404 `PAYOUT_NOT_FOUND`; 404 `EDUCATOR_NOT_FOUND`.

> **TC-PAYOUT-012: Settle — queued amount now exceeds the balance → 422**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** a queued row of 3000, but the educator's `pendingPayout` has since dropped to 1000 (e.g. a direct payout was processed in between).
> - **Steps:** settle the row.
> - **Expected:** 422 `AMOUNT_EXCEEDS_PENDING`; the row stays `pending`; `pendingPayout` still 1000. The stale row must be corrected by hand rather than over-drawing the balance.

---

## Dispute (`/disputes`) — all `authenticate`

> **TC-DISP-001: Open dispute — happy path**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** booking `confirmed` or `completed` (within 24h of session end); caller is the student.
> - **Steps:** `POST /disputes` `{ bookingId, reason:'twenty-plus character reason here' }`.
> - **Expected:** 201 `{ message:'Dispute opened' }`; booking → `disputed`; a disputeNote created with `body=reason`, `authorId=caller`.

> **TC-DISP-002: Open — reason < 20 chars → 400**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `POST /disputes` `{ bookingId, reason:'too short' }`.
> - **Expected:** 400 (raw zod flatten; reason min 20).

> **TC-DISP-003: Open — booking not found → 404**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** unknown bookingId with a valid reason.
> - **Expected:** 404 `NOT_FOUND` (code is `NOT_FOUND`, not `BOOKING_NOT_FOUND`).

> **TC-DISP-004: Open — not the student → 403**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** caller is the educator (or third party).
> - **Steps:** open dispute.
> - **Expected:** 403 `FORBIDDEN` (only the student may open).

> **TC-DISP-005: Open — wrong status → 422**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** booking `pending_payment`/`cancelled`.
> - **Steps:** open dispute.
> - **Expected:** 422 `CANNOT_DISPUTE_AT_THIS_STATUS` (allowed only from confirmed/completed).

> **TC-DISP-006: Open — window expired → 422**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** booking `completed`, session end (`scheduledAt+durationMinutes`) more than 24h ago.
> - **Steps:** open dispute.
> - **Expected:** 422 `DISPUTE_WINDOW_EXPIRED`. (Window check applies only when status is `completed`.)

> **TC-DISP-007: Get dispute notes — party / admin**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** disputed booking.
> - **Steps:** `GET /disputes/:bookingId/notes` as student, educator, and admin.
> - **Expected:** 200 array (asc, with author name/role) for all three.

> **TC-DISP-008: Get notes — non-party → 403 / not found → 404**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** notes as a third party (403 `FORBIDDEN`); unknown bookingId (404 `NOT_FOUND`).
> - **Expected:** codes as noted.

> **TC-DISP-009: Add dispute note — happy path**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** disputed booking; caller is a party or admin.
> - **Steps:** `POST /disputes/:bookingId/notes` `{ body:'some note' }`.
> - **Expected:** 201 with created note.

> **TC-DISP-010: Add note — not in dispute → 422 (before length check)**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** booking not `disputed`.
> - **Steps:** add a note.
> - **Expected:** 422 `NOT_IN_DISPUTE` (status check precedes length check).

> **TC-DISP-011: Add note — too short → 400**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** disputed booking.
> - **Steps:** add `{ body:'hi' }` (<5 chars).
> - **Expected:** 400 `NOTE_TOO_SHORT`.

> **TC-DISP-012: Add note — non-party → 403 / not found → 404**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** add as a third party; add to unknown bookingId.
> - **Expected:** 403 `FORBIDDEN`; 404 `NOT_FOUND`.

---

## Earning (`/earning`) — `authenticate`

> **TC-EARN-001: My earnings — gross/net + commission**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** educator with completed bookings; config `platform_commission_pct` set.
> - **Steps:** `GET /earning/me/earnings`.
> - **Expected:** 200 `{ allTime:{ gross, net, sessionCompleted }, thisMonth:{ gross, net }, commissionPct }`. NOTE (known bug): current net formula yields negative values (`gross*(1-pct)/100`); assert current output and link the fix ticket.

> **TC-EARN-002: My earnings — no auth → 401**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `GET /earning/me/earnings` without bearer.
> - **Expected:** 401.

> **TC-EARN-003: My earnings — commission default when config missing**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** no `platform_commission_pct` config row.
> - **Steps:** `GET /earning/me/earnings`.
> - **Expected:** 200; `commissionPct=20` (default).

> **TC-EARN-004: My payouts — cursor pagination**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** `GET /earning/me/payouts?limit=20`.
> - **Expected:** 200 `{ items, nextCursor }`, `paidAt desc`.

> **TC-EARN-005: My payouts — invalid cursor**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** `GET /earning/me/payouts?cursor=nonexistent`.
> - **Expected:** currently 500 (Prisma throws on bad cursor). Assert and file a fix.

---

## Referral (`/referral`) — `authenticate`

> **TC-REF-001: My referral stats**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** user with a referralCode who referred others.
> - **Steps:** `GET /referral/me/stats`.
> - **Expected:** 200 `{ referralCode, referralCount, totalBonusEarned }`; count = students with `referredBy=me`; bonus = sum of walletTransactions with `reference` starting `referral_by:`.

> **TC-REF-002: Referral stats — no auth → 401**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `GET /referral/me/stats` without bearer.
> - **Expected:** 401.

> **TC-REF-003: Dual-credit on signup with referral code**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** referrer is a student with a valid referralCode; config `referral_signup_bonus` (default 50).
> - **Steps:** `POST /student/profile/setup` (temp token) with `{ role:'student', referralCode:<referrer code> }`.
> - **Expected:** 200 tokens; new student `referredBy=referrer.id`; referrer wallet credited (`reference=referral_by:<newUserId>`); new user wallet credited (`reference=referred_by:<referrerId>`).

> **TC-REF-004: Signup — unknown/invalid referral code silently ignored**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** profile setup with `referralCode:'ZZZZZZ'`.
> - **Expected:** 200; no credits granted; `referredBy` unset.

> **TC-REF-005: Signup — referrer is an educator → no credit**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** referralCode belongs to an educator (no student profile).
> - **Steps:** profile setup with that code.
> - **Expected:** 200; no dual-credit (referrer must be a student).

---

## Wallet (`/student/wallet`)

> **TC-WALLET-001: Get wallet — happy path**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** student with a wallet.
> - **Steps:** `GET /student/wallet`.
> - **Expected:** 200 `{ wallet:{ balance } }`; `balance:0` if no wallet row (no 404).

> **TC-WALLET-002: Get wallet — no auth / wrong role**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `GET /student/wallet` unauth; with educator token.
> - **Expected:** 401 unauth; 403 wrong role.

> **TC-WALLET-003: Wallet transactions — latest 50**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** `GET /student/wallet/transactions`.
> - **Expected:** 200 `{ transactions }` (≤50, `createdAt desc`); empty array if none.

> **TC-WALLET-004: Admin grant reflected in wallet**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** admin grants credit (TC-ADM-017).
> - **Steps:** `GET /student/wallet` for that student.
> - **Expected:** balance includes the grant; matching credit transaction visible.

---

## Notification (`/notification`) — `authenticate`

> **TC-NOTIF-001: List notifications — unread count + pagination**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** user with read + unread notifications.
> - **Steps:** `GET /notification?limit=20`.
> - **Expected:** 200 `{ items, nextCursor, unreadCount }`, `createdAt desc`.

> **TC-NOTIF-002: List — no auth → 401**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `GET /notification` without bearer.
> - **Expected:** 401.

> **TC-NOTIF-003: Mark one read**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** unread notification owned by the caller.
> - **Steps:** `PATCH /notification/:notificationId/read`.
> - **Expected:** 200 `{ message:'Marked read' }`; `readAt` set.

> **TC-NOTIF-004: Mark read — other user's / unknown id is a silent no-op**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** mark read a notification that belongs to someone else or does not exist.
> - **Expected:** 200 (updateMany matches 0 rows; no 404). Assert the target's `readAt` unchanged.

> **TC-NOTIF-005: Mark all read**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** `PATCH /notification/read-all`.
> - **Expected:** 200 `{ message:'All notifications marked read' }`; all caller's unread → read; still 200 when already all read.

---

## Recording (`/recordings`)

> **TC-REC-001: Zoom webhook — URL validation challenge**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `POST /recordings/zoom-webhook` with valid `x-zm-signature` and `event:'endpoint.url_validation'` `{ payload:{ plainToken } }`.
> - **Expected:** 200 `{ plainToken, encryptedToken:<HMAC hex> }`.

> **TC-REC-002: Zoom webhook — recording.completed queues processing**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** valid signature; payload has `session_name`=bookingId and an MP4 in `recording_files`.
> - **Steps:** `POST /recordings/zoom-webhook`.
> - **Expected:** 200 `{ message:'processing' }`; async `processZoomRecording` scheduled.

> **TC-REC-003: Zoom webhook — bad signature → 401**
> - **Type:** integration
> - **Priority:** P0
> - **Steps:** webhook with a wrong `x-zm-signature`.
> - **Expected:** 401 `INVALID_SIGNATURE`.

> **TC-REC-004: Zoom webhook — non-recording events ignored**
> - **Type:** integration
> - **Priority:** P2
> - **Steps:** valid signature, `event:'meeting.started'` / missing payload / no MP4.
> - **Expected:** 200 `{ message:'ignored' | 'no payload' | 'no mp4' }` respectively.

> **TC-REC-005: Get recording presigned URL — participant**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** a `ready` recording for a booking the caller participated in.
> - **Steps:** `GET /recordings/:bookingId`.
> - **Expected:** 200 `{ url, expiresIn:3600 }` (1h presigned).

> **TC-REC-006: Get recording — not found / expired → 404**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** no recording, not authorized, or beyond the 24h availability window.
> - **Steps:** `GET /recordings/:bookingId`.
> - **Expected:** 404 `{ error:'NOT_FOUND_OR_EXPIRED' }`.

> **TC-REC-007: Get recording — no auth → 401**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** `GET /recordings/:bookingId` without bearer.
> - **Expected:** 401.

---

## Crons (`src/crons/index.ts`)

Registered jobs and their schedules, as of `src/crons/index.ts`:

| Job | Schedule |
|---|---|
| `auto-complete` | `*/5 * * * *` |
| `session-reminders` | `* * * * *` |
| `educator-delay-check` | `*/2 * * * *` (even minutes) |
| `student-delay-check` | `1-59/2 * * * *` (odd minutes) |
| `review-prompt` | `*/30 * * * *` |
| `stale-booking-expiry` | `*/15 * * * *` |
| `wallet-credit-expiry` | `0 2 * * *` |
| `token-cleanup` | `0 3 * * *` |
| `auto-payout` | `0 4 * * *` |

There is **no** `no-show-detection` job any more (it disputed at 5 min and pre-empted the
escalation in `educator-delay-check`), and no `session-reminder-1hr` / `session-reminder-5min`
— both reminders were merged into `session-reminders`. Every scan is bounded by a `take`
limit so a backlog can't pull the whole table into memory.

> **TC-CRON-001: Removed jobs are not registered**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** `startCrons()` invoked with `node-cron` stubbed to record registrations.
> - **Steps:** collect the lock names / schedules registered.
> - **Expected:** the set matches the table above exactly. `no-show-detection`, `session-reminder-1hr` and `session-reminder-5min` are absent. A test still referencing those names is stale, not failing.

> **TC-CRON-002: Delay checks are scheduled on opposite minutes**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** as above.
> - **Steps:** read the two cron expressions.
> - **Expected:** `educator-delay-check` is `*/2 * * * *` (even minutes) and `student-delay-check` is `1-59/2 * * * *` (odd minutes) — they never fire on the same minute, so they don't contend for the DB or each other's lock.

> **TC-CRON-003: Auto-complete (scheduledAt+duration+5min) → completed + payout accrual + pushes**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** booking `confirmed`; `now > scheduledAt + durationMinutes + 5min`.
> - **Steps:** run `auto-complete` (schedule `*/5 * * * *`).
> - **Expected:** booking → `completed`; `accrueEducatorPayout(id)` runs once (educator `pendingPayout` credited, `payoutAccrued` set); student gets "Session completed" (deep link `mole://review/<id>`) and educator gets "Session completed" (deep link `mole://educator/sessions`); no wallet side effects; the `updateMany where { id, status:'confirmed' }` guard prevents a double-flip and a double accrual on a concurrent run.

> **TC-CRON-013: Auto-complete query is bounded to plausible candidates**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** many `confirmed` bookings, most of them scheduled in the future or in the last minute.
> - **Steps:** run `auto-complete` and capture the Prisma query.
> - **Expected:** the filter includes `scheduledAt: { not: null, lt: now - 5min }` (the earliest possible end, since the shortest session plus grace is 5 min) with `orderBy scheduledAt asc` and `take: 500` — it does **not** scan every confirmed booking ever. The exact per-booking end (`scheduledAt + durationMinutes + 5min`) is still checked in JS, so nothing inside the bound is completed early.

> **TC-CRON-004: Auto-complete — not yet past window → unchanged**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** confirmed booking still within its window.
> - **Steps:** run cron.
> - **Expected:** stays `confirmed`.

> **TC-CRON-005: Educator delay — 5+ min penalty**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** `confirmed`, `zoomSessionId` set, `scheduledAt < now-5min`, `educatorJoinedAt=null`, `penaltyPct=0`, `educatorCallSentAt=null`.
> - **Steps:** run `educator-delay-check` (schedule `*/2 * * * *`), phase A.
> - **Expected:** `penaltyPct=10`, `educatorCallSentAt` set; reminder call + push to educator; runs once (guard on `penaltyPct=0`).

> **TC-CRON-006: Educator delay — 15+ min no-show → disputed + refund**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** `confirmed`, `scheduledAt < now-15min`, `educatorJoinedAt=null`, `creditsApplied>0`.
> - **Steps:** run `educator-delay-check` phase B.
> - **Expected:** booking → `disputed`; wallet refund (`reference=noshow_refund:<id>`) only when `creditsApplied > 0`; the student gets an "Educator didn't join" push (deep link `mole://booking/<id>`) whose body mentions the refund only when credits were actually returned; `updateMany where { status:'confirmed' }` makes it idempotent. The old conflict with a separate 5-min `no-show-detection` job is gone — that job was deleted, so the 5-min penalty from phase A now reliably lands first and sticks.

> **TC-CRON-007: Student delay — 5+ min, educator present → call only**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** `confirmed`, `zoomSessionId` set, `scheduledAt < now-5min`, `educatorJoinedAt` set, `studentJoinedAt=null`, `studentCallSentAt=null`.
> - **Steps:** run `student-delay-check` (schedule `1-59/2 * * * *`).
> - **Expected:** `studentCallSentAt` set; reminder call + push to student; NO status change / no penalty; runs once per booking.

### Session reminders (`session-reminders`, `* * * * *`)

One job serves both reminders. Eligibility is driven by the **`Booking.reminder1hSent` /
`Booking.reminder5mSent` flags**, not by a narrow "is it exactly now+60min" window — so a
tick the scheduler missed sends the reminder *late* rather than never. The window maths and
wording live in `src/crons/reminders.ts` and are already unit-tested in `reminders.test.ts`;
the cases below extend that to the DB-facing behaviour.

> **TC-CRON-008: 1h reminder — happy path, enriched copy**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** `confirmed` booking ~60 min out, `reminder1hSent=false`, a topic, and named student + educator.
> - **Steps:** run `session-reminders`.
> - **Expected:** both parties get a push + `Notification` row titled **"Session in 1 hour"**, deep link `mole://session/<id>`. The student's body names the *educator* and the educator's body names the *student*, both including the topic and the time rendered in **IST** — e.g. "Your Mechanics session with Aarav Sharma is at 20 Jul 2026, 4:00 pm. Get ready to join from the Mole app." `reminder1hSent` → true, `reminder5mSent` still false.

> **TC-CRON-009: 5m reminder claims BOTH flags**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** `confirmed` booking ~3 min out with `reminder1hSent=false`, `reminder5mSent=false` (e.g. booked inside the hour).
> - **Steps:** run `session-reminders`.
> - **Expected:** one push per party titled "Session starting in 3 minutes" with the "join now" tail; **both** `reminder1hSent` and `reminder5mSent` are set to true in the same claim. This is what stops a late-booked session from receiving a "Session in 1 hour" reminder afterwards — a booking must never get both reminders.

> **TC-CRON-014: Bands are contiguous — a booking is in exactly one pass**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** `reminderWindows(now)`.
> - **Steps:** classify sessions at +90, +30, +3, −1 and −5 minutes.
> - **Expected:** matches `reminders.test.ts` — `oneHour.after === fiveMin.upTo` (no gap); +30 is 1h-only; +3 is 5m-only; +90 is beyond the hour horizon and picked up by neither; −1 still qualifies for the 5m nudge (2 min grace) while −5 does not, because a session five minutes late is the delay-check's problem, not a reminder's.

> **TC-CRON-015: Missed tick — reminder is delivered late, not dropped**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** `confirmed` booking 23 min out with `reminder1hSent=false` (the process was down when it passed the 60-min mark).
> - **Steps:** run `session-reminders`.
> - **Expected:** the booking is still selected — the 1h pass filters on the flag plus the band `(now+6min, now+60min]`, not on a one-minute window — and the reminder is sent with the **true** lead time: title "Session in 23 minutes", never "Session in 1 hour". This is the exact case pinned by the `reminderTitle` unit test.

> **TC-CRON-016: Reminder titles state the real gap**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** `reminderTitle(scheduledAt, now)`.
> - **Steps:** evaluate at +60, +23, +5, +1, 0 and −1 minutes.
> - **Expected:** "Session in 1 hour" (≥55 min); "Session in 23 minutes"; "Session starting in 5 minutes" (≤6 min switches to *starting*); "Session starting in 1 minute" (singular unit); and at 0 or already past, still "Session starting in 1 minute" — it never says "in 0 minutes".

> **TC-CRON-017: Flag is claimed atomically — concurrent runs send once**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** one eligible booking; the cron lock bypassed so two passes really overlap.
> - **Steps:** run two `sendReminder` passes concurrently over the same booking.
> - **Expected:** the flag is claimed with `updateMany where { id, reminder1hSent:false }` **before** any push is sent; exactly one pass gets `claimed.count === 1` and sends, the other gets `0` and returns immediately. Exactly one push per party, one `Notification` row per party.

> **TC-CRON-018: Failed send reverts the claim so the next tick retries**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** eligible booking; `sendPushWithRecord` stubbed to throw.
> - **Steps:** run `session-reminders`; assert flags; unstub; run again.
> - **Expected:** first run claims the flag, the send throws, and the catch **hands the claim back** (`reminder1hSent` → false; for a 5m send both flags → false); the error is logged per-booking and does not abort the loop over the other bookings. The second run re-selects the booking and delivers it. A claimed-but-unsent reminder must never be silently dropped.

> **TC-CRON-019: Already-reminded and non-confirmed bookings are skipped**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** a booking 60 min out with `reminder1hSent=true`; a `cancelled` booking 60 min out; a `pending_payment` booking 3 min out.
> - **Steps:** run `session-reminders` twice.
> - **Expected:** no pushes for any of them (both passes filter `status:'confirmed'` plus the relevant flag); re-running the job for an already-reminded booking is a no-op — reminders are exactly-once per flag, per booking.

> **TC-CRON-020: Reminder copy falls back gracefully on missing relations**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** booking whose topic or counterpart user name is null.
> - **Steps:** run `session-reminders`.
> - **Expected:** body reads "your topic" / "your educator" / "your student" instead of `undefined`; a party with a null `fcmToken` still gets an in-app `Notification` row (push is skipped, record is not).

> **TC-CRON-010: Wallet credit expiry (daily 2am)**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** `walletCredit` with `expiresAt < now`, `usedAt=null`.
> - **Steps:** run `wallet-credit-expiry` (schedule `0 2 * * *`).
> - **Expected:** credit `usedAt` set; wallet balance decremented by `credit.amount`; walletTransaction `type=debit, reference=credit_expired:<id>`.

> **TC-CRON-011: Wallet credit expiry — future/used credits untouched**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** credits with `expiresAt > now` or `usedAt` already set.
> - **Steps:** run cron.
> - **Expected:** no balance change; credits unchanged.

> **TC-CRON-012: Cron locking prevents concurrent runs**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** any cron wrapped in `withCronLock`.
> - **Steps:** invoke the same cron twice concurrently.
> - **Expected:** only one execution proceeds; the other is skipped by the lock.

> **TC-CRON-021: Wallet credit expiry never drives the balance negative**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** an expired unused `walletCredit` of 500, but the wallet balance is only 200 (the credit was partly spent).
> - **Steps:** run `wallet-credit-expiry`.
> - **Expected:** credit `usedAt` stamped; the clawback is `min(credit.amount, balance)` = 200, so the balance lands on 0, not −300; one `walletTransaction` `type=debit, amount=200, reference=credit_expired:<id>`. A wallet already at 0 gets no transaction at all.

> **TC-CRON-022: Review prompt — one-time follow-up 2h after a session**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** `completed` booking with `scheduledAt < now-2h`, no `review`, `reviewPromptSent=false`.
> - **Steps:** run `review-prompt` (`*/30 * * * *`) twice.
> - **Expected:** first run claims `reviewPromptSent` atomically and sends "How was your session?" naming the topic (deep link `mole://review/<id>`); the second run sends nothing. A booking that already has a review, or is under 2h old, is never selected.

> **TC-CRON-023: Stale booking expiry — unaccepted and unpaid**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** an `awaiting_educator` booking created >24h ago and a `pending_payment` booking created >2h ago; plus fresher ones of each.
> - **Steps:** run `stale-booking-expiry` (`*/15 * * * *`).
> - **Expected:** both stale bookings → `cancelled` and their slots freed; the fresh ones untouched. **No refund is issued** — money and credits are only taken at payment, i.e. at the transition to `confirmed`, so nothing was ever debited. The student push differs per source status: the `awaiting_educator` one says the educator didn't respond, the `pending_payment` one says the unpaid booking expired; both deep-link `mole://search`. The `updateMany where { id, status: <original> }` claim makes it idempotent.

> **TC-CRON-024: Token cleanup removes spent and expired auth tokens**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** expired and used `otpCode` rows, expired `refreshToken` rows, and `adminAuthToken` rows that are expired or already used; plus a live unused one of each.
> - **Steps:** run `token-cleanup` (`0 3 * * *`).
> - **Expected:** deletes `otpCode` where expired **or** `used`; `refreshToken` where expired (used-but-unexpired refresh tokens survive by design); `adminAuthToken` where expired **or** `usedAt != null`. The live unused rows of each type remain, so an in-flight sign-in isn't broken.

### Auto-payout (`auto-payout`, `0 4 * * *`)

This job **cannot pay anyone** — there is no bank/UPI integration. It only queues a worklist.
Any test asserting that it marks a payout paid or decrements `pendingPayout` is asserting the
old, wrong behaviour.

> **TC-CRON-025: Auto-payout queues only — never marks paid, never decrements**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** `auto_payout_enabled=true`, today's weekday equals `auto_payout_day`, an `approved` educator with `pendingPayout = 4200.75` and no outstanding payout row.
> - **Steps:** run `auto-payout`.
> - **Expected:** one `PayoutHistory` row `{ status:'pending', type:'session_earnings', amount: 4200 (floored), note:'Auto payout run <YYYY-MM-DD>' }` with **`paidAt` null and no `reference`**; the educator's `pendingPayout` is **unchanged at 4200.75**; the educator gets a "Payout queued" push worded as queued, not paid ("We'll confirm once the transfer is complete."). Money is only moved to `paid` by an admin settling the row — see TC-PAYOUT-006.

> **TC-CRON-026: Auto-payout — gated on config and weekday**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** eligible educators exist.
> - **Steps:** run with `auto_payout_enabled` unset/false; then with it true but the weekday ≠ `auto_payout_day` (0=Sun…6=Sat, default 1).
> - **Expected:** returns immediately in both cases; no rows created, no pushes. The enabled check is strict (`value !== true`).

> **TC-CRON-027: Auto-payout — skips educators with an unsettled row**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** an approved educator with `pendingPayout = 5000` who already has a `status:'pending'` `PayoutHistory` row of 5000 from a previous run.
> - **Steps:** run `auto-payout` on a later payout day.
> - **Expected:** the educator is skipped and logged; still exactly one queued row. Queueing again would double-count, because `pendingPayout` still includes the earlier row's amount — the skip is what keeps the queue reconcilable against the balance.

> **TC-CRON-028: Auto-payout — same-day idempotency and minimum threshold**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** `min_payout_amount = 500`; educators with `pendingPayout` of 5000, 300 and 0.
> - **Steps:** run `auto-payout` twice on the same day.
> - **Expected:** only the 5000 educator is queued — the query filters `pendingPayout >= max(min_payout_amount, 1)`, and a floored amount of 0 is skipped. The second run finds a row already carrying today's `note` and returns without queueing anything again.

---

## Cross-cutting / regression

> **TC-XCUT-001: Booking FSM — full happy lifecycle**
> - **Type:** e2e
> - **Priority:** P0
> - **Preconditions:** student + approved educator with slots/rate.
> - **Steps:** create booking → educator respond accept → payment intent → mock-confirm → get session token → both join → educator end with OTP.
> - **Expected:** status walks `awaiting_educator → pending_payment → confirmed → completed`; Payment `success`; end-OTP generated on both-joined; final booking `completed`.

> **TC-XCUT-002: Booking FSM — decline path**
> - **Type:** e2e
> - **Priority:** P1
> - **Steps:** create → educator declines.
> - **Expected:** `awaiting_educator → cancelled`.

> **TC-XCUT-003: Booking FSM — payment failure path**
> - **Type:** e2e
> - **Priority:** P1
> - **Steps:** create → accept → webhook FAILURE.
> - **Expected:** `pending_payment → failed` (booking + payment both `failed`).

> **TC-XCUT-004: Booking FSM — no-show → disputed → admin resolve**
> - **Type:** e2e
> - **Priority:** P0
> - **Steps:** confirmed booking → educator never joins → no-show cron → admin resolve with refund.
> - **Expected:** `confirmed → disputed → completed`; wallet refunded.

> **TC-XCUT-005: Review visibility gate end-to-end**
> - **Type:** e2e
> - **Priority:** P0
> - **Steps:** complete session → student submits review (pending) → check discover (absent) → admin approves → check discover (present, rating updated).
> - **Expected:** pending review invisible; approval makes it visible and recomputes `overallRating`.

> **TC-XCUT-006: Ownership-code convention regression**
> - **Type:** integration
> - **Priority:** P1
> - **Steps:** hit non-participant paths across domains.
> - **Expected:** payment + session-token/join + createIntent/retry use **404** for ownership; booking get/cancel/extend + dispute use **403 FORBIDDEN**; endSession uses **404**. Lock these in so refactors don't silently change them.

## Quiz (QUIZ)

### Quiz Setup & Validation

> **TC-QUIZ-001: Create quiz with invalid question count**
> - **Type:** Validation
> - **Priority:** P0
> - **Preconditions:** User is educator; booking exists
> - **Steps:** POST `/bookings/:id/quiz` with 4 questions (not 5)
> - **Expected:** 400 Bad Request; error message indicates exactly 5 questions required

> **TC-QUIZ-002: Create quiz with invalid option count**
> - **Type:** Validation
> - **Priority:** P0
> - **Preconditions:** User is educator; booking exists with valid 5-question structure
> - **Steps:** POST `/bookings/:id/quiz` where one question has 3 options (not 4)
> - **Expected:** 400 Bad Request; error message indicates each question requires 4 options

> **TC-QUIZ-003: Create quiz with out-of-range correctIndex**
> - **Type:** Validation
> - **Priority:** P0
> - **Preconditions:** User is educator; booking exists with 5 questions × 4 options each
> - **Steps:** POST `/bookings/:id/quiz` where correctIndex = 4 (valid range 0–3)
> - **Expected:** 400 Bad Request; error message indicates correctIndex must be 0–3

> **TC-QUIZ-004: Educator can create quiz on own booking**
> - **Type:** Happy Path
> - **Priority:** P0
> - **Preconditions:** User is educator; booking created by this educator; session not yet completed
> - **Steps:** POST `/bookings/:id/quiz` with valid 5×4 questions, all correctIndex in range [0,3]
> - **Expected:** 201 Created; quiz persisted; response includes `{ id, questions: [{question, options}], created }`

> **TC-QUIZ-005: Educator cannot create quiz on others' booking**
> - **Type:** Authorization
> - **Priority:** P0
> - **Preconditions:** User A is educator; booking created by educator B
> - **Steps:** POST `/bookings/:id/quiz` with valid quiz
> - **Expected:** 403 Forbidden; error message indicates not authorized

### Quiz Locking

> **TC-QUIZ-006: Quiz is locked after first student attempt**
> - **Type:** State Transition
> - **Priority:** P0
> - **Preconditions:** Booking completed; student has attempted quiz (5/5 or 4/5)
> - **Steps:** Educator tries POST `/bookings/:id/quiz` to edit quiz
> - **Expected:** 409 Conflict; error code `QUIZ_LOCKED`; body indicates quiz already attempted

### Student Quiz Retrieval

> **TC-QUIZ-007: Student GET strips correctIndex from quiz**
> - **Type:** Data Masking
> - **Priority:** P0
> - **Preconditions:** Booking completed; quiz created by educator
> - **Steps:** Student GET `/bookings/:id/quiz`
> - **Expected:** 200 OK; response includes `{ questions: [{question, options}] }` (no correctIndex); includes `{ exists: true, rewardCoins: 10, alreadyAttempted: false }`

> **TC-QUIZ-008: Student GET includes alreadyAttempted flag after attempt**
> - **Type:** State Tracking
> - **Priority:** P0
> - **Preconditions:** Booking completed; student has already attempted quiz
> - **Steps:** Student GET `/bookings/:id/quiz`
> - **Expected:** 200 OK; `alreadyAttempted: true`; `attempt: { correctCount, passed, coinsAwarded }`

> **TC-QUIZ-009: Student GET with no quiz returns exists: false**
> - **Type:** Happy Path
> - **Priority:** P1
> - **Preconditions:** Booking exists but no quiz created
> - **Steps:** Student GET `/bookings/:id/quiz`
> - **Expected:** 200 OK; `{ exists: false }`

### Attempt Prerequisites

> **TC-QUIZ-010: Student cannot attempt quiz before booking completed**
> - **Type:** State Guard
> - **Priority:** P0
> - **Preconditions:** Booking exists with quiz; session status is not `completed`
> - **Steps:** POST `/bookings/:id/quiz/attempt` with 5 answers
> - **Expected:** 409 Conflict; error code `BOOKING_NOT_COMPLETED`

### Attempt Submission & Grading

> **TC-QUIZ-011: Student submits 5/5 correct answers**
> - **Type:** Happy Path
> - **Priority:** P0
> - **Preconditions:** Booking `completed`; quiz exists; no prior attempt
> - **Steps:** POST `/bookings/:id/quiz/attempt` with `{ answers: [0, 1, 2, 3, 0] }` matching all correctIndex values
> - **Expected:** 200 OK; response `{ correctCount: 5, passed: true, coinsAwarded: 10 }`; user wallet credited 10 coins; `WalletTransaction(credit, reference: "quiz_reward:<booking-id>")` created

> **TC-QUIZ-012: Student submits 4/5 correct answers**
> - **Type:** Edge Case
> - **Priority:** P0
> - **Preconditions:** Booking `completed`; quiz exists; no prior attempt
> - **Steps:** POST `/bookings/:id/quiz/attempt` with 4 correct, 1 incorrect answer
> - **Expected:** 200 OK; response `{ correctCount: 4, passed: false, coinsAwarded: 0 }`; wallet NOT credited; NO transaction created

> **TC-QUIZ-013: Answer array validation on attempt**
> - **Type:** Validation
> - **Priority:** P0
> - **Preconditions:** Booking `completed`; quiz exists
> - **Steps:** POST `/bookings/:id/quiz/attempt` with 4 answers (not 5)
> - **Expected:** 400 Bad Request; error indicates exactly 5 answers required

> **TC-QUIZ-014: Answer index out of range validation**
> - **Type:** Validation
> - **Priority:** P0
> - **Preconditions:** Booking `completed`; quiz exists
> - **Steps:** POST `/bookings/:id/quiz/attempt` with one answer = 4 (valid range 0–3)
> - **Expected:** 400 Bad Request; error indicates each answer must be 0–3

### Attempt Uniqueness & Conflict

> **TC-QUIZ-015: Second attempt rejected with ALREADY_ATTEMPTED**
> - **Type:** State Guard
> - **Priority:** P0
> - **Preconditions:** Booking `completed`; student already has one attempt
> - **Steps:** POST `/bookings/:id/quiz/attempt` with 5 answers
> - **Expected:** 409 Conflict; error code `ALREADY_ATTEMPTED`; no wallet change; no transaction

> **TC-QUIZ-016: Double-credit prevented by unique constraint**
> - **Type:** Data Integrity
> - **Priority:** P0
> - **Preconditions:** Booking `completed`; quiz exists with valid answers; attempt is being processed
> - **Steps:** Two concurrent requests POST `/bookings/:id/quiz/attempt` both with correct answers (simulated race condition)
> - **Expected:** First request 200 OK with coins credited; second request 409 Conflict with `ALREADY_ATTEMPTED` or P2002 → 409; wallet shows only 10 coins added (not 20)

