# Mole — Web App Test Cases (TC-WEB)

Test-case reference for the **web** app (`code/web`): Next.js 14 App Router + MUI + Zustand + SWR.
Follows the format in [README.md](./README.md). IDs are `TC-WEB-NNN`.

**Architecture notes used across cases**

- **Auth transport:** JWT in `localStorage` (`accessToken`, `refreshToken`) *and* a non-httpOnly `accessToken` cookie (`store/authStore.ts` `saveTokens`, `max-age=86400`, `SameSite=Strict`). The cookie is what `middleware.ts` reads.
- **Middleware guard (`middleware.ts`):** public paths = `/welcome`, `/phone`, `/otp`, `/role` (matched by `startsWith`), plus `/_next`, `/api`, `/favicon.ico`. Every other path requires the `accessToken` cookie, else `redirect → /welcome`.
- **Root routing (`app/page.tsx`):** `loadSession()` then route by role/status — no user → `/welcome`; educator without `hasCompletedIdentity` → `/educator/identity-check`; educator `pending` → `/educator/under-verification`; educator `rejected` → `/educator/rejected`; educator approved → `/educator`; student → `/student`.
- **API client (`services/api.ts`):** `baseURL = <NEXT_PUBLIC_API_URL|http://localhost:3000>/api/v1`, `timeout: 10000`. Request interceptor attaches `Authorization: Bearer <accessToken>`. Response interceptor: on `401` (once, `_retry` guard) POST `/auth/refresh`; on success replays request; on missing/failed refresh clears tokens and `window.location.href = '/welcome'`.
- **Dev helpers:** OTP mock `123456` (non-prod); `POST /payment/:id/mock-confirm`.
- **Colors token set** referred to as `C` (`C.primary`, `C.primaryFixed`, etc.).

---

## Cross-cutting

> **TC-WEB-001: Unauthenticated user redirected to /welcome by middleware**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** No `accessToken` cookie present.
> - **Steps:** Navigate directly to a protected route, e.g. `/student`, `/educator`, `/student/sessions`.
> - **Expected:** Middleware returns `redirect → /welcome`; protected page never renders.

> **TC-WEB-002: Public auth paths reachable without a token**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** No `accessToken` cookie.
> - **Steps:** Visit `/welcome`, `/phone`, `/otp`, `/role` directly.
> - **Expected:** Each renders; no redirect (matched via `startsWith`).

> **TC-WEB-003: Next internals and API paths bypass the guard**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** No token.
> - **Steps:** Request `/_next/*`, `/api/*`, `/favicon.ico`.
> - **Expected:** `NextResponse.next()`; not redirected.

> **TC-WEB-004: 401 triggers one silent token refresh and replays the request**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Valid `refreshToken` in localStorage; API returns 401 once then 200.
> - **Steps:** Make an authenticated API call whose first response is 401.
> - **Expected:** Interceptor POSTs `/auth/refresh`, stores new access+refresh tokens, retries original with new bearer, resolves. Retry attempted only once (`_retry` guard).

> **TC-WEB-005: Refresh failure clears tokens and forces logout**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** `refreshToken` missing or `/auth/refresh` returns error; a request 401s.
> - **Steps:** Trigger a 401 with no/invalid refresh token.
> - **Expected:** `accessToken`+`refreshToken` removed from localStorage; `window.location.href = '/welcome'`; original error re-thrown.

> **TC-WEB-006: Focus-visible ring on keyboard navigation**
> - **Type:** e2e
> - **Priority:** P2
> - **Preconditions:** Any page with interactive controls.
> - **Steps:** Tab through buttons/links/inputs using the keyboard only.
> - **Expected:** Focused element shows a 2px indigo focus ring; ring does not appear on mouse click (focus-visible semantics).

> **TC-WEB-007: Responsive grid reflow at xs/sm**
> - **Type:** e2e
> - **Priority:** P2
> - **Preconditions:** Any multi-column grid (student home, search, dashboard stats).
> - **Steps:** Resize from desktop to mobile widths.
> - **Expected:** Multi-column grids reflow to a single column at `xs`/`sm`; no horizontal overflow.

> **TC-WEB-008: StatusPill color mapping per booking status**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** StatusPill rendered for each status.
> - **Steps:** Render pill for `awaiting_educator`, `pending_payment`, `confirmed`, `completed`, `cancelled`, `disputed`, `failed`.
> - **Expected:** Each status maps to its own color/label; no status renders unstyled/default.

> **TC-WEB-009: API base URL and timeout configured**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** none.
> - **Steps:** Inspect axios instance config.
> - **Expected:** `baseURL` = `<NEXT_PUBLIC_API_URL|http://localhost:3000>/api/v1`; `timeout: 10000`; bearer attached only when `accessToken` exists.

---

## Auth / Public

### Welcome (`/welcome`)

> **TC-WEB-010: Welcome renders and CTA routes to phone entry**
> - **Type:** e2e
> - **Priority:** P1
> - **Preconditions:** none.
> - **Steps:** Load `/welcome`; click **Get Started**.
> - **Expected:** Three feature cards and branding render; navigation to `/phone`.

### Phone (`/phone`)

> **TC-WEB-011: Valid 10-digit number sends OTP and routes to /otp**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** none.
> - **Steps:** Enter `9876543210`; click **Send OTP**.
> - **Expected:** `sendOTP('+919876543210')` → POST `/auth/otp/send`; on success `router.push('/otp?phone=%2B919876543210')` (URL-encoded).

> **TC-WEB-012: Input strips non-digits and caps at 10**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** On `/phone`.
> - **Steps:** Type letters, spaces, and more than 10 digits.
> - **Expected:** Only digits kept; max length 10; `+91` prefix shown as input adornment.

> **TC-WEB-013: Send button disabled until exactly 10 digits**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** On `/phone`.
> - **Steps:** Enter fewer than 10 digits.
> - **Expected:** **Send OTP** disabled; also disabled while `loading`.

> **TC-WEB-014: sendOTP failure surfaces error, stays on page**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** `/auth/otp/send` returns error.
> - **Steps:** Submit a valid number.
> - **Expected:** Error message shown; button returns from "Sending..." to "Send OTP"; no navigation.

> **TC-WEB-015: Loading state during send**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** In-flight send.
> - **Steps:** Submit; observe during request.
> - **Expected:** Button label "Sending..." and disabled; Enter key also submits.

### OTP (`/otp`)

> **TC-WEB-016: useSearchParams wrapped in Suspense**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** none.
> - **Steps:** Load `/otp?phone=...`.
> - **Expected:** Content is under `<Suspense>` with `CircularProgress` fallback; page never throws a CSR-bailout error from `useSearchParams`.

> **TC-WEB-017: Six-box entry auto-advances and auto-submits**
> - **Type:** e2e
> - **Priority:** P0
> - **Preconditions:** On `/otp?phone=%2B91...`.
> - **Steps:** Type 6 digits.
> - **Expected:** Focus auto-advances per digit; when all 6 filled, `verifyOTP(phone, code)` → POST `/auth/otp/verify` auto-fires.

> **TC-WEB-018: Existing user verify routes home**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** verify returns `{ isNewUser: false, accessToken, refreshToken, user }`.
> - **Steps:** Enter valid OTP.
> - **Expected:** `saveTokens` writes localStorage + cookie; `/auth/me` merged into user; `router.push('/')` (root then role-routes).

> **TC-WEB-019: New user verify routes to role selection**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** verify returns `{ isNewUser: true, tempToken }`.
> - **Steps:** Enter valid OTP.
> - **Expected:** `tempToken` stored in store; `router.push('/role')`; no tokens saved yet.

> **TC-WEB-020: Invalid OTP clears fields and refocuses**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** `/auth/otp/verify` returns error.
> - **Steps:** Enter a wrong 6-digit code.
> - **Expected:** Error "Invalid OTP. Please try again."; all boxes cleared; first box refocused.

> **TC-WEB-021: Backspace on empty box moves focus back**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Partial code entered.
> - **Steps:** Press Backspace on an empty box.
> - **Expected:** Focus moves to previous box.

> **TC-WEB-022: Resend countdown and re-arm**
> - **Type:** e2e
> - **Priority:** P1
> - **Preconditions:** Just arrived on `/otp`.
> - **Steps:** Observe countdown; wait to 0; click **Resend OTP**.
> - **Expected:** "Resend OTP in Xs" counts down from 30; disabled until 0; clicking calls `sendOTP(phone)` and restarts the 30s timer.

> **TC-WEB-023: Verify button disabled until 6 digits**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** On `/otp`.
> - **Steps:** Enter fewer than 6 digits.
> - **Expected:** **Verify** disabled; label toggles to "Verifying..." during request.

> **TC-WEB-024: Dev mock OTP accepted**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** Non-prod backend.
> - **Steps:** Enter `123456`.
> - **Expected:** Verify succeeds (new/existing branch as returned).

### Role selection (`/role`)

> **TC-WEB-025: Student registration routes to /student**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** `tempToken` present (new user).
> - **Steps:** Select **Student**, enter name, submit.
> - **Expected:** `register({ role:'student', name })` → POST `/student/profile/setup` with `x-temp-token`; tokens saved; `router.push('/student')`.

> **TC-WEB-026: Educator registration routes to onboarding**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** `tempToken` present.
> - **Steps:** Select **Educator**, enter name, submit.
> - **Expected:** `register({ role:'educator', name })` → POST `/educator/onboarding/step1`; tokens saved; `router.push('/educator/onboarding')`.

> **TC-WEB-027: Validation requires role and non-empty name**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** On `/role`.
> - **Steps:** Submit with no role or blank/whitespace name.
> - **Expected:** Error "Please fill in all fields"; button disabled until both provided; name trimmed.

> **TC-WEB-028: Role tiles toggle selection styling**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** On `/role`.
> - **Steps:** Click Student then Educator.
> - **Expected:** Selected tile shows active border/background; only one selected at a time.

> **TC-WEB-029: Registration failure shows error**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** register endpoint returns error.
> - **Steps:** Submit valid form.
> - **Expected:** Error text shown; button returns from "Creating account..."; no navigation.

---

## Student

### Home (`/student`)

> **TC-WEB-030: Home loads live and suggested rails**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Authenticated student.
> - **Steps:** Load `/student`.
> - **Expected:** SWR GET `/discover/live` (live rail) and GET `/discover/suggested` (suggested grid) fire; results render as educator cards.

> **TC-WEB-031: Skeletons while discovery loads**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Requests in flight.
> - **Steps:** Observe initial render.
> - **Expected:** Live shows `EducatorCardGridSkeleton` (count 4, md 3); suggested shows skeleton (count 6, md 4).

> **TC-WEB-032: Empty suggested state**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** `/discover/suggested` returns `[]`.
> - **Steps:** Load home.
> - **Expected:** EmptyState "No educators yet" / "Try searching for a topic to find a tutor." with action → `/student/search`.

> **TC-WEB-033: Live rail hidden when empty**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** `/discover/live` returns `[]` and not loading.
> - **Steps:** Load home.
> - **Expected:** Live section not rendered (renders only while loading or when `length > 0`).

> **TC-WEB-034: Quick-search chips route to search with query**
> - **Type:** e2e
> - **Priority:** P1
> - **Preconditions:** On home; `/discover/trending-topics` has resolved.
> - **Steps:** Click a chip.
> - **Expected:** `router.push('/student/search?q=<encodeURIComponent(topic.name)>')`.

> **TC-WEB-132: Quick-search chips come from the backend, not a hardcoded list**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Authenticated student.
> - **Steps:** Load `/student` and inspect the chip row against the `/discover/trending-topics` response.
> - **Expected:** SWR fetches `/discover/trending-topics` alongside `/discover/live` and `/discover/suggested`; one chip per returned topic, labelled `topic.name` and keyed by `topic.id`, in the order the API returned them (most-booked first). The old hardcoded Mechanics / Heat Transfer / Electrostatics / Wave Optics array must be gone — chips changing with booking volume is the expected behaviour.

> **TC-WEB-133: Trending chips degrade quietly**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** `/discover/trending-topics` is still in flight, or fails.
> - **Steps:** Render home.
> - **Expected:** `quickSearches` falls back to `[]`, so the chip row is simply empty — no skeleton, no error, and the search field and both rails still work.

> **TC-WEB-035: Search bar focus routes to search page**
> - **Type:** e2e
> - **Priority:** P2
> - **Preconditions:** On home.
> - **Steps:** Focus the search field.
> - **Expected:** Navigate to `/student/search`.

> **TC-WEB-036: "See All" links**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** On home.
> - **Steps:** Click each "See All".
> - **Expected:** Live → `/student/search?live=true`; Suggested → `/student/search`.

> **TC-WEB-037: Educator card routes to profile**
> - **Type:** e2e
> - **Priority:** P1
> - **Preconditions:** Cards rendered.
> - **Steps:** Click a card.
> - **Expected:** `router.push('/student/educator/<id>')`.

### Search (`/student/search`)

> **TC-WEB-038: Suspense wraps search content**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** none.
> - **Steps:** Load `/student/search`.
> - **Expected:** `SearchContent` under `<Suspense>` with `CircularProgress` fallback (uses `useSearchParams`).

> **TC-WEB-039: Debounced query search**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** On search.
> - **Steps:** Type "mech" quickly.
> - **Expected:** GET `/discover/search?q=<debounced>&live=<param>` fires after ~400ms debounce, not per keystroke.

> **TC-WEB-134: Empty Find-an-educator defaults to suggested, not a raw search**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Authenticated student; arrive at `/student/search` with no `q` and no `live` param.
> - **Steps:** Load the page without typing.
> - **Expected:** `isBrowsing` is true, so SWR requests **`/discover/suggested`** — the personalised top-6 — rather than an unfiltered `/discover/search`. The page previously dumped every approved educator here.

> **TC-WEB-135: Typing or filtering switches away from suggested**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** On `/student/search` in the default browsing state.
> - **Steps:** Type a query and wait out the debounce; then clear it; then arrive via `?live=true`.
> - **Expected:** A non-blank debounced query switches the SWR key to `/discover/search?q=…&live=…`; clearing it back to blank returns to `/discover/suggested`; `?live=true` also forces the search key even with an empty query (`isBrowsing` requires both a blank query and `live !== 'true'`).

> **TC-WEB-040: Query param prefills field**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Arrive via `?q=Mechanics`.
> - **Steps:** Load page.
> - **Expected:** Field prefilled "Mechanics"; autofocus; search runs for it.

> **TC-WEB-041: live=true filter propagates**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** Arrive via `?live=true`.
> - **Steps:** Load page.
> - **Expected:** Search request includes `live=true`.

> **TC-WEB-042: Loading skeleton during search**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Request in flight.
> - **Steps:** Observe.
> - **Expected:** `EducatorCardGridSkeleton` (count 6, md 4).

> **TC-WEB-043: Empty search state**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Search returns `[]`.
> - **Steps:** Search a nonsense term.
> - **Expected:** `SearchOffIcon` EmptyState "No educators found" / "Try a different name, subject, or topic."

> **TC-WEB-044: Result card routes to profile**
> - **Type:** e2e
> - **Priority:** P1
> - **Preconditions:** Results shown.
> - **Steps:** Click a result.
> - **Expected:** Navigate to `/student/educator/<id>`.

### Educator profile (`/student/educator/[id]`)

> **TC-WEB-045: Profile loads via discover-by-id**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Valid educator id.
> - **Steps:** Load `/student/educator/<id>`.
> - **Expected:** SWR GET `/discover/<id>`; renders name, avatar (2-letter initials fallback), bio, rating, topics, reviews, and BookingPanel.

> **TC-WEB-046: Loading skeleton layout**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Request in flight.
> - **Steps:** Observe.
> - **Expected:** Circular avatar (72px) skeleton, name/rating/bio text skeletons, and a ~420px rounded booking-panel skeleton.

> **TC-WEB-047: Error state on load failure**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** `/discover/<id>` errors.
> - **Steps:** Load page.
> - **Expected:** "Couldn't load this educator" with `error.response.data.message ?? error.message ?? "Please try again in a moment."`

> **TC-WEB-048: Not-found educator**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Request resolves with no educator.
> - **Steps:** Load page.
> - **Expected:** "Educator not found" / "This profile may no longer be available." with **Back to search** → `/student/search`.

> **TC-WEB-049: Rating formatted to one decimal**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Educator has numeric rating.
> - **Steps:** Inspect header and each review.
> - **Expected:** Rating rendered via `.toFixed(1)` (e.g. `4.0`); review count "(N reviews)"; per-review ratings also one decimal.

> **TC-WEB-050: Topic chips styled**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Educator has topics.
> - **Steps:** Inspect chips.
> - **Expected:** Small chips with `bgcolor C.primaryFixed`, `color C.primary`.

> **TC-WEB-051: Approved reviews list rendering**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** Educator has approved reviews.
> - **Steps:** Scroll reviews.
> - **Expected:** Each review shows student name, `formatDisplayDate(createdAt)`, one-decimal rating; dividers between entries.

> **TC-WEB-052: Empty reviews state**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** No reviews.
> - **Steps:** Load page.
> - **Expected:** `RateReviewOutlinedIcon` EmptyState "No reviews yet" / "Be the first to book and review this educator."

> **TC-WEB-053: Desktop 65/35 sticky booking panel**
> - **Type:** e2e
> - **Priority:** P1
> - **Preconditions:** md+ viewport.
> - **Steps:** Scroll the page.
> - **Expected:** Info column 65% (left), BookingPanel 35% (right) sticky as a Card.

> **TC-WEB-054: Mobile booking drawer**
> - **Type:** e2e
> - **Priority:** P1
> - **Preconditions:** xs viewport.
> - **Steps:** Tap **Book a Session**.
> - **Expected:** Sticky card hidden; BookingPanel opens in a right-anchored bottom/side Drawer; closeable.

### BookingPanel

> **TC-WEB-055: Session-type select shows only rated types**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** Educator has some of `rateTeaching`/`rateRevision`/`rateDoubtClearing`.
> - **Steps:** Open panel.
> - **Expected:** Only session types with a defined rate appear; select shown only when `>1` type available; defaults to the first available.

> **TC-WEB-056: Duration options 30/60/90 with 60 default**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** Panel open.
> - **Steps:** Inspect duration control.
> - **Expected:** Options 30, 60, 90 minutes; default 60.

> **TC-WEB-057: Date selector — next 7 days**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Panel open.
> - **Steps:** Inspect date options.
> - **Expected:** 7 upcoming days formatted "Mon, 12 Jul" (en-IN); default first day; changing date resets `selectedSlot` to null.

> **TC-WEB-058: Availability slots fetched per date**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Date selected.
> - **Steps:** Pick a date.
> - **Expected:** SWR GET `/educator/<id>/availability?date=<date>`; slots converted `utcTimeToLocal` and formatted "2:30 PM".

> **TC-WEB-059: Slot chips select and highlight**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** Slots returned.
> - **Steps:** Tap a slot.
> - **Expected:** Selected chip filled `C.primary` / white / weight 600; others outlined.

> **TC-WEB-060: Slots loading and empty states**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Loading, then empty.
> - **Steps:** Observe while loading; select a date with no slots.
> - **Expected:** 6 skeleton chips while loading; "No slots available on this date." when empty.

> **TC-WEB-061: Computed price**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** Type + duration chosen.
> - **Steps:** Change duration.
> - **Expected:** Price = `Math.round(rate * duration / 60)`; shown only when `> 0`; "Total" + bold primary price.

> **TC-WEB-062: Submit disabled until topic and slot chosen**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** Panel open.
> - **Steps:** Try to submit without topic or slot.
> - **Expected:** **Request Session** disabled until both `topicId` and `selectedSlot` set (also while `loading`).

> **TC-WEB-063: Booking POST and routing**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Topic + slot selected.
> - **Steps:** Submit.
> - **Expected:** POST `/bookings` with `{ educatorId, topicId, scheduledAt (slotToUtcIso), durationMinutes, sessionType, creditsToApply:0 }`; if returned `status==='pending_payment'` → `/student/payment?bookingId=<id>`, else → `/student/sessions`; button shows "Booking...".

> **TC-WEB-064: Booking failure alerts**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** `/bookings` errors.
> - **Steps:** Submit.
> - **Expected:** Alert with `e.response.data.message` or "Booking failed"; no navigation.

### Confirm booking (`/student/confirm-booking`)

> **TC-WEB-065: Suspense + booking fetch**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** `?bookingId=<id>`.
> - **Steps:** Load page.
> - **Expected:** Content under `<Suspense>` (CircularProgress fallback); SWR GET `/student/bookings/<id>` (only when id present); shows topic, educator, `formatDisplayDate`, `formatDisplayTime`, duration, and bold `₹<price>`.

> **TC-WEB-066: Loading and not-found**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Loading, then missing booking.
> - **Steps:** Observe.
> - **Expected:** "Loading..." while fetching; "Booking not found" (error color) when absent.

> **TC-WEB-067: Confirm routes by resulting status**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Booking loaded.
> - **Steps:** Click **Confirm**.
> - **Expected:** POST `/student/bookings/<id>/confirm`; `pending_payment` → `/student/payment?bookingId=<id>`; other → `/student/sessions`; button disabled while loading.

> **TC-WEB-068: Confirm failure routes to booking-failed**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** confirm POST errors.
> - **Steps:** Click Confirm.
> - **Expected:** `router.push('/student/booking-failed')`.

> **TC-WEB-069: Cancel goes back**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** On page.
> - **Steps:** Click **Cancel**.
> - **Expected:** `router.back()`.

### Payment (`/student/payment`)

> **TC-WEB-070: Suspense + create intent**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** `?bookingId=<id>`.
> - **Steps:** Load page.
> - **Expected:** Content under `<Suspense>` (spinner fallback); POST `/payment/<id>/intent`; response `upiUri` enables **Open UPI App**.

> **TC-WEB-071: Status polling every 3s**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Intent created.
> - **Steps:** Wait.
> - **Expected:** GET `/payment/<id>/status` polled every 3s; polling stops on `confirmed` or `failed`.

> **TC-WEB-072: Success routes to booking-confirmed**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Status returns `confirmed`.
> - **Steps:** Poll resolves confirmed.
> - **Expected:** `router.push('/student/booking-confirmed?bookingId=<id>')`.

> **TC-WEB-073: 5-minute countdown to booking-failed**
> - **Type:** e2e
> - **Priority:** P0
> - **Preconditions:** Payment not confirmed.
> - **Steps:** Let timer run to 0:00.
> - **Expected:** MM:SS timer from 5:00; at 0:00 navigate to `/student/booking-failed`.

> **TC-WEB-074: Failed status routes to booking-failed**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Status returns `failed`.
> - **Steps:** Poll resolves failed.
> - **Expected:** Navigate to `/student/booking-failed`.

> **TC-WEB-075: Mock-confirm dev button**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Non-prod.
> - **Steps:** Click **Simulate Payment Success**.
> - **Expected:** POST `/payment/<id>/mock-confirm`; next poll yields confirmed → booking-confirmed.

> **TC-WEB-076: UPI app pills / links**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Intent has `upiUri`.
> - **Steps:** Inspect.
> - **Expected:** GPay/PhonePe/BHIM/Paytm pills; **Open UPI App** deep-links the `upiUri`.

> **TC-WEB-077: Loading spinner before intent**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Intent in flight.
> - **Steps:** Observe.
> - **Expected:** CircularProgress until intent resolves.

### Booking confirmed (`/student/booking-confirmed`)

> **TC-WEB-078: Confirmation renders booking details**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** `?bookingId=<id>`.
> - **Steps:** Load page.
> - **Expected:** Under `<Suspense>`; SWR GET `/student/bookings/<id>`; green CheckCircle, topic, formatted date/time, educator name/avatar.

> **TC-WEB-079: Confirmed nav buttons**
> - **Type:** e2e
> - **Priority:** P2
> - **Preconditions:** On page.
> - **Steps:** Click each button.
> - **Expected:** **View My Sessions** → `/student/sessions`; **Back to Home** → `/student`.

### Booking failed (`/student/booking-failed`)

> **TC-WEB-080: Failure page renders and routes**
> - **Type:** e2e
> - **Priority:** P1
> - **Preconditions:** none.
> - **Steps:** Load; click buttons.
> - **Expected:** Red ErrorOutline, "Booking Failed", "Something went wrong. Your payment has not been charged."; **Back to Home** → `/student`; **Try Again** → `router.back()`.

### Sessions (`/student/sessions`)

> **TC-WEB-081: Upcoming/past tabs query by status set**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Authenticated student.
> - **Steps:** Toggle tabs.
> - **Expected:** GET `/bookings/my?status=<set>` — Upcoming = `awaiting_educator,pending_payment,confirmed`; Past = `completed,cancelled,disputed`.

> **TC-WEB-082: Sessions loading skeleton**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Request in flight.
> - **Steps:** Observe.
> - **Expected:** `ListSkeleton` (3 items, ~110px).

> **TC-WEB-083: Empty per-tab state**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Empty result.
> - **Steps:** Open a tab with none.
> - **Expected:** EmptyState "No upcoming/past sessions" with conditional action link.

> **TC-WEB-084: Conditional action buttons per status**
> - **Type:** e2e
> - **Priority:** P0
> - **Preconditions:** Bookings of varied status.
> - **Steps:** Inspect each card.
> - **Expected:** `confirmed` → **Join Session** → `/student/session/<id>`; `pending_payment` → **Complete Payment** → `/student/payment?bookingId=<id>`; `completed` → **Leave Review** → `/student/review/<id>`; StatusPill shown on every card.

### Live session (`/student/session/[bookingId]`)

> **TC-WEB-085: Session token fetch and provider display**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Confirmed booking.
> - **Steps:** Load `/student/session/<id>`.
> - **Expected:** SWR GET `/session/<id>/token`; dark full-screen area; "Connecting to session..."; provider = `NEXT_PUBLIC_VIDEO_PROVIDER ?? 'zoom'` shown when token received.

> **TC-WEB-086: OTP badge display**
> - **Type:** unit
> - **Priority:** P0
> - **Preconditions:** Token payload includes an OTP.
> - **Steps:** Load page.
> - **Expected:** Top-right chip `OTP: <value>` (the code the educator asks for to end session); hidden when no OTP.

### Review (`/student/review/[bookingId]`)

> **TC-WEB-087: 5-star rating required to submit**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Completed booking.
> - **Steps:** Load; try submit with no stars; then pick 4 stars, add comment, submit.
> - **Expected:** Submit disabled until `rating>0`; POST `/student/bookings/<id>/review` `{ rating, comment }`; review created as `pending`; navigate `/student/sessions`.

> **TC-WEB-088: Comment optional; hover/select stars**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** On review page.
> - **Steps:** Hover stars; leave comment blank; submit.
> - **Expected:** Gold fill on hover/selected, gray otherwise; empty comment allowed; button shows "Submitting...".

> **TC-WEB-089: Skip returns to sessions**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** On review page.
> - **Steps:** Click **Skip**.
> - **Expected:** Navigate `/student/sessions` without POST.

### Wallet (`/student/wallet`)

> **TC-WEB-090: Wallet balance and transactions**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Authenticated student.
> - **Steps:** Load `/student/wallet`.
> - **Expected:** GET `/student/wallet`; balance card; transactions with `formatDisplayDate`, credit green `+₹`, debit red `-₹`.

> **TC-WEB-091: Wallet loading and empty**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Loading, then empty transactions.
> - **Steps:** Observe.
> - **Expected:** `ListSkeleton` (count 4, ~64px); EmptyState "No transactions yet" when none.

### Referral (`/student/referral`)

> **TC-WEB-092: Referral stats and code**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Authenticated student with referral code.
> - **Steps:** Load `/student/referral`.
> - **Expected:** GET `/referral/me/stats` → referralCount + totalBonusEarned; code shown from auth store; "How it works" (3 steps).

> **TC-WEB-093: Stats skeletons while loading**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Stats in flight.
> - **Steps:** Observe.
> - **Expected:** Skeletons for count and bonus values.

> **TC-WEB-094: Copy code**
> - **Type:** e2e
> - **Priority:** P2
> - **Preconditions:** Code present.
> - **Steps:** Click **Copy**.
> - **Expected:** `navigator.clipboard` writes code; button shows "Copied!" for ~2s.

> **TC-WEB-095: Share with fallback**
> - **Type:** e2e
> - **Priority:** P2
> - **Preconditions:** Code present.
> - **Steps:** Click **Share**.
> - **Expected:** `navigator.share` when available; falls back to copy otherwise.

> **TC-WEB-096: Inactive account alert**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** No referral code / inactive.
> - **Steps:** Load page.
> - **Expected:** Alert shown instead of code card.

### Notifications (`/student/notifications`)

> **TC-WEB-097: Student notifications empty state**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Authenticated student.
> - **Steps:** Load page.
> - **Expected:** Static "No notifications yet" with icon; no API call.

### Profile / Navbar (`/student/profile`)

> **TC-WEB-098: Profile shows user and quick links**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Authenticated student.
> - **Steps:** Load `/student/profile`.
> - **Expected:** Avatar initials, name, phone; boxes route to `/student/wallet` and `/student/sessions`.

> **TC-WEB-099: Logout clears cookie + localStorage**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Authenticated student.
> - **Steps:** Open avatar menu → **Logout** (or profile logout).
> - **Expected:** `logout()` POSTs `/auth/logout` (best-effort), `clearTokens()` removes localStorage tokens and expires the `accessToken` cookie, `user=null`; `router.push('/welcome')`.

> **TC-WEB-100: Navbar active-route highlight and menu**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** On any student page.
> - **Steps:** Inspect nav (Home/Search/Sessions/Wallet), open avatar menu.
> - **Expected:** Active item highlighted via `pathname===href || startsWith(href)`; menu has Profile, Referral, divider, Logout; bell → `/student/notifications`; logo → `/student`.

> **TC-WEB-101: Student layout guard**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Session loaded, no user.
> - **Steps:** Land on a `/student/*` route.
> - **Expected:** `!isLoading && !user` → `router.replace('/welcome')`; navbar with 64px top offset when authorized.

---

## Educator

### Onboarding (`/educator/onboarding`)

> **TC-WEB-102: Onboarding submits bio/rate/subjects**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** New educator (post step1).
> - **Steps:** Fill bio, hourly rate, select ≥1 subject; submit.
> - **Expected:** SWR GET `/subjects` populates checkboxes; POST `/educator/onboarding/step2` `{ bio, hourlyRate:Number, subjectIds }`; success → `/educator/identity-check`.

> **TC-WEB-103: Onboarding validation and error**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** On onboarding.
> - **Steps:** Submit missing a field.
> - **Expected:** Button disabled until bio + rate + ≥1 subject; failure shows `e.response.data.message` or "Failed"; button "Saving..." during request.

### Identity check (`/educator/identity-check`)

> **TC-WEB-104: Identity submit and status reload**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Educator without identity.
> - **Steps:** Enter 12-digit Aadhaar + 10-char PAN; submit.
> - **Expected:** POST `/educator/identity` `{ aadhaar, pan (uppercase) }`; `loadSession()` refreshes user; navigate `/educator/under-verification`.

> **TC-WEB-105: Identity input masking and validation**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** On identity-check.
> - **Steps:** Type non-digits in Aadhaar and lowercase PAN.
> - **Expected:** Aadhaar digits-only capped 12; PAN uppercased capped 10; invalid input → "Enter valid Aadhaar (12 digits) and PAN (10 chars)"; failure → server message or "Submission failed".

### Under verification / Rejected

> **TC-WEB-106: Under-verification static screen**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Educator `pending`.
> - **Steps:** Land on `/educator/under-verification`.
> - **Expected:** Clock icon, "Under Review", 1–2 business days message; no actions.

> **TC-WEB-107: Rejected screen logout**
> - **Type:** e2e
> - **Priority:** P1
> - **Preconditions:** Educator `rejected`.
> - **Steps:** Land on `/educator/rejected`; click **Logout**.
> - **Expected:** Block icon, "Application Rejected"; `logout()` then navigate `/welcome`.

> **TC-WEB-108: Root routes educator by status**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Educator user with a given status.
> - **Steps:** Load `/`.
> - **Expected:** no identity → `/educator/identity-check`; pending → `/educator/under-verification`; rejected → `/educator/rejected`; approved → `/educator`.

### Dashboard (`/educator`)

> **TC-WEB-109: Dashboard loads educator summary**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Approved educator.
> - **Steps:** Load `/educator`.
> - **Expected:** GET `/educator/me`; stats Total Sessions, Rating (`Number(rating).toFixed(1)`), Total Earnings (`₹`); today's schedule; recent reviews data.

> **TC-WEB-110: Go-live toggle**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** On dashboard.
> - **Steps:** Toggle go-live.
> - **Expected:** PATCH `/educator/go-live` `{ isLive }`; "LIVE" chip + card styling when live.

> **TC-WEB-111: Today's schedule with Join**
> - **Type:** e2e
> - **Priority:** P1
> - **Preconditions:** Bookings scheduled today.
> - **Steps:** Inspect schedule; click **Join**.
> - **Expected:** Cards show topic, `formatDisplayTime`, duration, student name; **Join** → `/educator/session/<bookingId>`.

> **TC-WEB-112: Empty today's schedule**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** No bookings today.
> - **Steps:** Load dashboard.
> - **Expected:** Empty message for schedule.

### Calendar (`/educator/calendar`)

> **TC-WEB-113: Availability loads per day (UTC→local)**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Approved educator with availability.
> - **Steps:** Load `/educator/calendar`.
> - **Expected:** GET `/educator/me/availability`; 7 day cards; UTC slots shown in local time (`utcTimeToLocal`), 12h format.

> **TC-WEB-114: Add slot with end>start validation**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Add dialog open.
> - **Steps:** Choose start/end and save; try end ≤ start.
> - **Expected:** End dropdown only offers times after start; `start>=end` → alert "End time must be after start time"; valid → POST `/educator/me/availability` (local→UTC); chip appears.

> **TC-WEB-115: Delete slot chip**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** A slot exists.
> - **Steps:** Delete a chip; confirm.
> - **Expected:** `confirm()` prompt; DELETE `/educator/me/availability/<slotId>`; chip shows spinner while deleting then removed.

> **TC-WEB-116: Calendar loading and empty-day**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Loading; a day with no slots.
> - **Steps:** Observe.
> - **Expected:** Full-page CircularProgress while loading; per empty day "No slots — unavailable".

> **TC-WEB-117: Calendar error alert**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** POST/DELETE errors.
> - **Steps:** Trigger failure.
> - **Expected:** Alert on error; state unchanged.

### Earnings (`/educator/earnings`)

> **TC-WEB-118: Earnings stats and payouts**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Approved educator.
> - **Steps:** Load `/educator/earnings`.
> - **Expected:** GET `/educator/earnings`; cards This Month / Total Earned / Pending Payout (`₹`); payout history with `formatDisplayDate` and green amounts.

> **TC-WEB-119: Empty payout history**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** No payouts.
> - **Steps:** Load page.
> - **Expected:** Empty message under payout history.

### Profile (`/educator/profile`)

> **TC-WEB-120: Educator profile uses rateTeaching + topics**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Approved educator.
> - **Steps:** Load `/educator/profile`.
> - **Expected:** GET `/educator/me`; shows avatar, name, rating (`Number().toFixed(1)`), review count, **rateTeaching** (not hourlyRate), bio, **topics** chips (not subjects); **Edit** → `/educator/edit-profile`.

> **TC-WEB-121: Educator approved reviews / empty**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** With and without reviews.
> - **Steps:** Inspect reviews section.
> - **Expected:** Each review: star rating, `formatDisplayDate`, comment if present; empty → StarIcon EmptyState "No reviews yet".

### Edit profile (`/educator/edit-profile`)

> **TC-WEB-122: Edit form loads current values and subjects**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Approved educator.
> - **Steps:** Load `/educator/edit-profile`.
> - **Expected:** GET `/educator/me` + GET `/subjects`; fields bio, city, rateTeaching, rateRevision, rateDoubtClearing; topic checkboxes grouped by subject.

> **TC-WEB-123: Save patches profile**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Edit form open.
> - **Steps:** Change rates + topics; Save.
> - **Expected:** PATCH `/educator/profile` with `{ bio, city, rateTeaching, rateRevision, rateDoubtClearing }` (Number-coerced) and `topicIds` when any selected; `mutate()`; navigate `/educator/profile`; button "Saving..." while saving.

> **TC-WEB-124: Save failure and cancel**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** PATCH errors / user cancels.
> - **Steps:** Trigger error; then Cancel.
> - **Expected:** Alert "Could not save. Try again."; **Cancel** → `router.back()`.

> **TC-WEB-136: Intro Video card explains the re-review consequence**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** Approved educator on `/educator/edit-profile`.
> - **Steps:** View the Intro Video card.
> - **Expected:** Titled "Intro Video" with caption "One short intro (5 min max)…" stating that uploading a new one **puts the profile under review** until an admin re-approves. A single "Upload intro video" button wrapping a hidden `input type=file accept="video/*"`. The educator must not be surprised into losing visibility.

> **TC-WEB-137: Intro upload posts type `intro` and confirms the under-review state**
> - **Type:** e2e
> - **Priority:** P0
> - **Preconditions:** Intro Video card; a valid video under 5 minutes.
> - **Steps:** Pick the file.
> - **Expected:** Duration is read client-side, the file uploads via `uploadVideo(file,'demo-clip')`, then `POST /educator/demo-clips { storageKey, durationSeconds: Math.round(duration), type:'intro' }`; the button reads "Uploading…" and is disabled meanwhile; on success `mutate()` refetches and a success Alert reads "Video uploaded. Your profile is now under review."

> **TC-WEB-138: Teaching Videos card, max 3**
> - **Type:** e2e
> - **Priority:** P1
> - **Preconditions:** Educator with fewer than 3 teaching videos.
> - **Steps:** Upload a teaching video.
> - **Expected:** Card titled "Teaching Videos", caption "Up to 3 videos, 5 minutes max each"; upload posts `type:'demo'`; the success Alert counts this session's uploads and pluralises ("1 teaching video uploaded this session." / "2 teaching videos…").

> **TC-WEB-139: Client-side 5-minute guard fires before any upload**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** A video longer than 300s.
> - **Steps:** Select it in either the intro or the teaching card.
> - **Expected:** `getVideoDuration` reads the duration off a temporary `<video>` element and the upload is rejected **before** the file is sent, with the error Alert "Each video must be 5 minutes or shorter." No storage call, no `POST /educator/demo-clips`. The backend's 422 `VIDEO_TOO_LONG` is the backstop, not the first line of defence.

> **TC-WEB-140: Server limit errors are mapped to friendly copy**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Educator already has 3 teaching videos.
> - **Steps:** Upload a 4th (short) teaching video; then force an unrelated upload failure.
> - **Expected:** 422 `CLIP_LIMIT_REACHED` renders "You can upload a maximum of 3 teaching videos."; `VIDEO_TOO_LONG` renders the 5-minute message; anything else falls back to "Could not upload the video. Please try a smaller file or try again." Errors are per-card, so an intro failure doesn't clear the teaching card's state.

> **TC-WEB-141: File input resets so the same file can be retried**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** An upload just failed.
> - **Steps:** Select the identical file again.
> - **Expected:** The handler sets `e.target.value = ''` after each pick, so re-selecting the same file still fires a `change` event and retries.

### Session (`/educator/session/[bookingId]`)

> **TC-WEB-125: Session token fetch and OTP dialog on load**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Educator session route.
> - **Steps:** Load `/educator/session/<id>`.
> - **Expected:** GET `/sessions/<id>/token`; OTP verification dialog open on load; dark layout; "Waiting for OTP verification...".

> **TC-WEB-126: OTP input constraints**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** OTP dialog open.
> - **Steps:** Type letters/long input.
> - **Expected:** Digits only, max 6, centered large letter-spaced field; verify button disabled while verifying or length `< 4`.

> **TC-WEB-127: Valid End-OTP ends/starts session**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Correct OTP.
> - **Steps:** Enter OTP; submit.
> - **Expected:** POST `/sessions/<id>/end` with `otp`; on success `verified=true`, dialog closes, "Session in progress".

> **TC-WEB-128: INVALID_OTP handling**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Wrong OTP.
> - **Steps:** Submit incorrect code.
> - **Expected:** Helper text "Invalid OTP. Ask the student to share the code shown on their screen."; dialog stays open.

> **TC-WEB-129: End Session fallback routes to dashboard**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** In session.
> - **Steps:** Click **End Session**.
> - **Expected:** POST `/sessions/<id>/end`; navigate `/educator`.

### Educator notifications / layout

> **TC-WEB-130: Educator notifications empty state**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Approved educator.
> - **Steps:** Load `/educator/notifications`.
> - **Expected:** Static "No notifications yet" with `NotificationsNoneIcon`; no API call.

> **TC-WEB-131: Educator layout guard + sidebar**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Session loaded, no user.
> - **Steps:** Land on `/educator/*`.
> - **Expected:** `!isLoading && !user` → `router.replace('/welcome')`; permanent 240px Sidebar (Dashboard/Calendar/Earnings/Profile/Notifications); active item styled `C.primaryFixed`/`C.primary`; logout → `/welcome`; content offset by sidebar width.

## Quiz (QUIZ)

### Educator Setup UI

> **TC-QUIZ-001: Educator opens quiz setup dialog**
> - **Type:** UI Navigation
> - **Priority:** P0
> - **Preconditions:** User is educator; booking details page loaded
> - **Steps:** Click "Set Up Quiz" button on booking detail page
> - **Expected:** Modal dialog opens with 5 empty question blocks; each block has 4 option inputs + correctIndex selector

> **TC-QUIZ-002: Quiz setup validation on submit**
> - **Type:** Form Validation
> - **Priority:** P0
> - **Preconditions:** Quiz setup dialog open
> - **Steps:** Leave question 3 empty; try clicking "Save"
> - **Expected:** Red error highlight on question 3; "Save" button disabled; error toast "All 5 questions required"

> **TC-QUIZ-003: Quiz setup option count validation**
> - **Type:** Form Validation
> - **Priority:** P0
> - **Preconditions:** Quiz setup dialog open with all fields filled
> - **Steps:** Delete one option from question 1 (leaving 3 options); try "Save"
> - **Expected:** Red error highlight on question 1; "Save" disabled; error message "Each question requires 4 options"

> **TC-QUIZ-004: Educator saves valid quiz**
> - **Type:** Happy Path
> - **Priority:** P0
> - **Preconditions:** Quiz setup dialog open; all 5 questions filled with 4 options each + correctIndex set
> - **Steps:** Click "Save"
> - **Expected:** Dialog closes; success toast "Quiz saved"; booking detail page reloads; "Set Up Quiz" button replaced with "View Quiz" (read-only)

> **TC-QUIZ-005: Educator loads pre-saved quiz for editing (before attempt)**
> - **Type:** State Retrieval
> - **Priority:** P0
> - **Preconditions:** Booking has unsaved quiz; educator not yet attempted
> - **Steps:** Open booking detail; click "Edit Quiz"
> - **Expected:** Dialog opens with saved questions pre-filled; all fields editable

> **TC-QUIZ-006: Quiz setup dialog load failure blocks Save**
> - **Type:** Error Handling
> - **Priority:** P1
> - **Preconditions:** Quiz setup dialog open; network error during dialog load
> - **Steps:** Dialog attempts to load existing quiz; network fails
> - **Expected:** Error message displayed; all inputs disabled; "Save" button greyed out

> **TC-QUIZ-007: Quiz becomes read-only after student attempt**
> - **Type:** State Transition
> - **Priority:** P0
> - **Preconditions:** Booking has quiz; student has completed attempt
> - **Steps:** Educator navigates to booking detail
> - **Expected:** Quiz section shows "Quiz Locked" message; "Edit Quiz" button disabled; attempting to click shows tooltip "Quiz cannot be edited after student attempt"

### Student Quiz UI

> **TC-QUIZ-008: Student sees quiz button only if quiz exists and not yet attempted**
> - **Type:** Conditional UI
> - **Priority:** P0
> - **Preconditions:** Booking in review phase (completed); educator has created quiz; student has not attempted
> - **Steps:** Navigate to session review page
> - **Expected:** "Take Quiz" button visible in session summary; button enabled

> **TC-QUIZ-009: Student quiz button hidden if already attempted**
> - **Type:** Conditional UI
> - **Priority:** P0
> - **Preconditions:** Booking in review phase; quiz exists; student already attempted
> - **Steps:** Navigate to session review page
> - **Expected:** "Take Quiz" button not visible; instead shows "Quiz Completed — <score>/5"

> **TC-QUIZ-010: Student quiz button hidden if no quiz exists**
> - **Type:** Conditional UI
> - **Priority:** P1
> - **Preconditions:** Booking in review phase; no quiz created by educator
> - **Steps:** Navigate to session review page
> - **Expected:** No quiz button or section visible

> **TC-QUIZ-011: Student passes quiz (5/5) — confetti + coins UI**
> - **Type:** Happy Path / UX Celebration
> - **Priority:** P0
> - **Preconditions:** Booking review page; quiz with correct answers loaded; student answers all 5 correctly
> - **Steps:** Answer all 5 MCQs correctly; click "Submit"
> - **Expected:** Confetti animation triggers; "Passed!" modal shows "5/5 Correct! +10 coins"; coin balance updates in header; "Done" button dismisses modal

> **TC-QUIZ-012: Student fails quiz (4/5) — score shown, no coins**
> - **Type:** Edge Case / UX Feedback
> - **Priority:** P0
> - **Preconditions:** Booking review page; quiz loaded; student answers 4 correct, 1 incorrect
> - **Steps:** Answer 4 MCQs correctly, 1 incorrectly; click "Submit"
> - **Expected:** Modal shows "Score: 4/5 — Try Again!" (no confetti); no coin UI update; "Done" button available

> **TC-QUIZ-013: Student attempt consumed — button disabled after submit**
> - **Type:** State Transition
> - **Priority:** P0
> - **Preconditions:** Student submitted quiz (passed or failed)
> - **Steps:** Review page reloaded or modal dismissed
> - **Expected:** "Take Quiz" button replaced with "Quiz Completed — <score>/5" (disabled); no retry available

> **TC-QUIZ-014: Quiz submission network error recovery**
> - **Type:** Error Handling
> - **Priority:** P1
> - **Preconditions:** Student completed answers; about to submit
> - **Steps:** Network fails during POST `/bookings/:id/quiz/attempt`
> - **Expected:** Error toast "Failed to submit quiz"; "Submit" button enabled; student can retry
