# Mole — Mobile App Test Cases (`TC-MOB-NNN`)

Test-case reference for the Mole **mobile** app (Expo / React Native, expo-router).
Covers the student and educator journeys, root session/redirect logic, UI states
(loading / empty / error / validation), role gating, navigation, and mobile-specific
concerns (press feedback, tap targets, safe-area, offline/permission).

Format per [README.md](./README.md): **Type** (unit/integration/e2e), **Priority**
(P0/P1/P2), **Preconditions**, **Steps**, **Expected**.

## Conventions used in these cases

- **Roles:** student, educator. Auth = phone (+91, 10 digits) + 6-digit OTP.
- **Booking status:** `awaiting_educator → pending_payment → confirmed → completed`; plus `cancelled`, `failed`, `disputed`, `expired`.
- **Educator status:** `pending → approved | rejected | suspended`; identity gate via `hasCompletedIdentity`.
- **Dev helpers:** OTP mock accepts `123456`; `POST /payment/:id/mock-confirm` and `POST /bookings/:id/mock-confirm` confirm payment without a webhook (`__DEV__` only).
- **Video provider:** Zoom SDK is a native module — **not available in Expo Go** ("Dev Build Required"). Educators **never** join from mobile.
- **`PressableScale`** primitive: animates children to scale **0.98 over 150ms** on press-in, back to **1** on release (`useNativeDriver: true`).
- **`StatusBadge`** canonical mapping (bg / text / label):
  - `awaiting_educator` → indigoTint / warning / "Awaiting Educator"
  - `pending_payment` → indigoTint / warning / "Pay Now"
  - `pending` → indigoTint / warning / "Pending"
  - `confirmed` → indigoTint / primary / "Confirmed"
  - `completed` → surfaceContainer / success / "Completed"
  - `cancelled` / `failed` / `disputed` → errorContainer / error
  - `expired` → surfaceContainer / textSecondary / "Expired"
- **`EmptyState`** renders an icon-in-circle + title + optional subtitle for all empty lists.
- **Tap targets:** interactive controls must have ≥44–48px hit area (via size or `hitSlop`).

---

## Root layout & session (`app/_layout.tsx`, `store/authStore`)

> **TC-MOB-001: Branded splash renders while fonts/session load**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** Cold app start; fonts not yet loaded OR `isLoading === true`.
> - **Steps:** Launch app.
> - **Expected:** `BrandedSplash` renders (gradient `#34495E→#2E4057→#28384A`, centered `mole_mark.png` 120×120) with fade-in + spring-scale animation; native splash hidden only after `fontsLoaded && !isLoading`.

> **TC-MOB-002: `loadSession` runs once on mount**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** App launched.
> - **Steps:** Observe mount effect.
> - **Expected:** `loadSession()` called once; reads `accessToken` from SecureStore; if absent → `isLoading=false`, `user=null`; if present → `GET /auth/me` and sets `user`. On `/auth/me` failure, `isLoading=false` without throwing.

> **TC-MOB-003: Not logged in redirects to `(auth)/welcome`**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** No stored token; `user=null`; not currently in `(auth)` group.
> - **Steps:** Finish session load.
> - **Expected:** `router.replace('/(auth)/welcome')`. If already in `(auth)` group, no redirect.

> **TC-MOB-004: FCM token registered when logged in**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** `user` truthy; Firebase messaging module available; notifications permission granted.
> - **Steps:** Auth check effect runs with a user.
> - **Expected:** `registerFcmToken()` requests permission; if AUTHORIZED/PROVISIONAL, fetches token and calls `PUT /auth/fcm-token` with `{ fcmToken }`.

> **TC-MOB-005: FCM registration no-ops without native module / permission**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Running in Expo Go (messaging require fails) OR permission denied.
> - **Steps:** Trigger auth check with a user.
> - **Expected:** No crash; no `PUT /auth/fcm-token` call; failure is caught and warns only.

> **TC-MOB-006: Student with `onboardingSeen=false` → onboarding tutorial**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** `user.role='student'`, `onboardingSeen=false`, not in `(student)`.
> - **Steps:** Auth check runs.
> - **Expected:** `router.replace('/(student)/onboarding-tutorial')`.

> **TC-MOB-007: Student with `onboardingSeen=true` → student home**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** `user.role='student'`, `onboardingSeen=true`, not in `(student)`.
> - **Steps:** Auth check runs.
> - **Expected:** `router.replace('/(student)')`.

> **TC-MOB-008: Educator missing identity → identity-check**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** `user.role='educator'`, `hasCompletedIdentity=false`.
> - **Steps:** Auth check runs.
> - **Expected:** `router.replace('/(educator)/identity-check')` (evaluated first, before status).

> **TC-MOB-009: Educator status `pending` → under-verification**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** `hasCompletedIdentity=true`, `educatorStatus='pending'`.
> - **Steps:** Auth check runs.
> - **Expected:** `router.replace('/(educator)/under-verification')`.

> **TC-MOB-010: Educator status `approved` → educator home**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** `hasCompletedIdentity=true`, `educatorStatus='approved'`.
> - **Steps:** Auth check runs.
> - **Expected:** `router.replace('/(educator)')`.

> **TC-MOB-011: Educator status `rejected` → rejected screen**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** `hasCompletedIdentity=true`, `educatorStatus='rejected'`.
> - **Steps:** Auth check runs.
> - **Expected:** `router.replace('/(educator)/rejected')`.

> **TC-MOB-012: Student cannot land on educator group and vice-versa**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Logged-in student.
> - **Steps:** Deep-link to an `(educator)` route.
> - **Expected:** Role-based redirect returns the user to their own group; no cross-role screen renders.

> **TC-MOB-013: Version update check on mount**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** App launched.
> - **Steps:** Observe mount.
> - **Expected:** `checkForUpdate()` called once independently of auth.

> **TC-MOB-014: Logout clears tokens and returns to welcome**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Logged in.
> - **Steps:** Call `authStore.logout()`.
> - **Expected:** `POST /auth/logout` with refreshToken (best-effort); SecureStore access+refresh tokens deleted; `onboarding_complete` removed from AsyncStorage; `user=null` → root redirect to `(auth)/welcome`.

---

## Auth flow — `(auth)` group

> **TC-MOB-020: Welcome screen renders hero + Get Started**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** In `(auth)/welcome`.
> - **Steps:** Open screen.
> - **Expected:** Hero image + gradient, "mole" wordmark, tagline, three feature pills, "Get Started" CTA, terms text. Safe-area top+bottom insets applied.

> **TC-MOB-021: Get Started navigates to phone**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Welcome screen.
> - **Steps:** Tap "Get Started".
> - **Expected:** `router.push('/(auth)/phone')`.

> **TC-MOB-022: Phone input accepts max 10 digits, +91 prefix**
> - **Type:** unit
> - **Priority:** P0
> - **Preconditions:** Phone screen.
> - **Steps:** Type into number field.
> - **Expected:** Fixed 🇮🇳 +91 prefix; `keyboardType=phone-pad`; `maxLength=10`; autofocus. Checkmark icon appears at exactly 10 digits.

> **TC-MOB-023: Send OTP disabled until 10 digits**
> - **Type:** unit
> - **Priority:** P0
> - **Preconditions:** Phone screen, `<10` digits.
> - **Steps:** Inspect "Send OTP" button.
> - **Expected:** Button disabled (`btnDisabled` style) while `phone.length !== 10` or loading; `handleSend` early-returns if `<10`.

> **TC-MOB-024: Send OTP success navigates to otp with phone param**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** 10-digit number entered.
> - **Steps:** Tap "Send OTP".
> - **Expected:** `sendOTP('+91' + phone)` → `POST /auth/otp/send`; button shows "Sending…"; on success `router.push('/(auth)/otp', { phone: '+91…' })`.

> **TC-MOB-025: Send OTP failure shows error**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** `POST /auth/otp/send` errors.
> - **Steps:** Tap "Send OTP".
> - **Expected:** `showError('Could not send OTP. Please try again.')`; loading reset; no navigation.

> **TC-MOB-026: OTP — 6 separate boxes with auto-advance/backspace**
> - **Type:** unit
> - **Priority:** P0
> - **Preconditions:** OTP screen.
> - **Steps:** Type digits; press backspace on empty box.
> - **Expected:** Six single-char boxes; typing advances focus to next; backspace on empty box moves focus back; box style toggles active/error.

> **TC-MOB-027: OTP auto-verifies when all 6 filled**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** 5 digits entered.
> - **Steps:** Enter 6th digit.
> - **Expected:** `handleVerify` fires automatically → `verifyOTP(phone, code)` → `POST /auth/otp/verify`; button shows "Verifying…".

> **TC-MOB-028: OTP new user → role screen**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Verify returns `isNewUser=true` with `tempToken`.
> - **Steps:** Complete OTP.
> - **Expected:** `tempToken` stored in memory only (not SecureStore); `router.replace('/(auth)/role')`.

> **TC-MOB-029: OTP existing user → session + role redirect**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Verify returns `isNewUser=false` with tokens + user.
> - **Steps:** Complete OTP.
> - **Expected:** access+refresh tokens saved to SecureStore; `GET /auth/me` merged into user; root layout redirects by role/status.

> **TC-MOB-030: OTP incorrect code error + reset**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Verify rejects.
> - **Steps:** Enter wrong code.
> - **Expected:** Error "Incorrect code. Please try again."; boxes cleared; focus returns to first box.

> **TC-MOB-031: OTP resend countdown 60s then enabled**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** OTP screen just opened.
> - **Steps:** Wait through countdown.
> - **Expected:** "Resend in 0:NN" ticks down from 60; at 0 shows "Resend OTP" button; tapping it resets countdown to 60.

> **TC-MOB-032: OTP masked phone + edit number**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** OTP screen with phone param.
> - **Steps:** Read subtitle; tap "Edit number".
> - **Expected:** Phone shown masked (`+91 NN••••••NN`); "Edit number" → `router.back()` to phone screen.

> **TC-MOB-033: Role select defaults to student and is single-choice**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** Role screen.
> - **Steps:** Tap each role card.
> - **Expected:** "I want to learn" preselected; tapping toggles selection border/check; only one selected at a time.

> **TC-MOB-034: Role continue routes to correct onboarding**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Role screen, role chosen.
> - **Steps:** Tap "Continue".
> - **Expected:** student → `router.replace('/(student)/onboarding')`; educator → `router.replace('/(educator)/onboarding')`.

---

## Student — Home (`app/(student)/index.tsx`)

> **TC-MOB-040: Home fetches live + suggested + trending topics**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Student home.
> - **Steps:** Open screen.
> - **Expected:** `GET /discover/live`, `GET /discover/suggested` and `GET /discover/trending-topics` fire; "Live Now" rendered as a horizontal rail, "Suggested for You" as a vertical list, and the search pills built from `trendingData.topics` (keyed by `topic.id`, labelled `topic.name`).

> **TC-MOB-215: Search pills are backend-driven and degrade to empty**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Student home.
> - **Steps:** Compare the pill row against the `/discover/trending-topics` response; then re-render with that request failing or in flight.
> - **Expected:** One pill per returned topic in API order (most-booked in the last 30 days first) — no hardcoded topic list remains. On failure/in-flight the list falls back to `[]` and the row renders empty rather than erroring. Parity with web (TC-WEB-132/133).

> **TC-MOB-216: Suggested rail is capped at 6**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** Many approved, highly-rated educators.
> - **Steps:** Open home and count the "Suggested for You" entries.
> - **Expected:** At most 6 — the cap is enforced server-side by `SUGGESTED_LIMIT`, so the screen renders whatever it receives without its own slice.

> **TC-MOB-041: Home loading shows skeleton/placeholder**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Discover requests in flight.
> - **Steps:** Open home.
> - **Expected:** Loading placeholder (skeleton/spinner) shown until data resolves; no crash on undefined arrays.

> **TC-MOB-042: Home empty sections hidden**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Both discover arrays empty.
> - **Steps:** Open home.
> - **Expected:** "Live Now" and "Suggested" sections hidden when their arrays are empty; search bar + chips still shown.

> **TC-MOB-043: Educator cell content**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** Discover returns educators.
> - **Steps:** Inspect an educator cell.
> - **Expected:** Cell shows avatar (photo or initials), name, rating "★ X.X" + review count, city, starting price. Live educators show a green live dot.

> **TC-MOB-044: Search bar entry navigation**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Home.
> - **Steps:** Tap "What do you want to learn?" bar.
> - **Expected:** `router.push('/(student)/search')`. Bar is a `PressableScale` (scales to 0.98).

> **TC-MOB-045: Quick chips prefill search query**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Home with topic chips (Mechanics, Heat Transfer, …).
> - **Steps:** Tap a chip.
> - **Expected:** `router.push('/(student)/search?q=<chip>')`.

> **TC-MOB-046: "See all" links**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** Home with both sections.
> - **Steps:** Tap "See all" on each.
> - **Expected:** Live → `/(student)/search?live=true`; Suggested → `/(student)/search`.

> **TC-MOB-047: Educator cell press → profile**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Home with educators.
> - **Steps:** Tap a cell.
> - **Expected:** `router.push('/(student)/educator/{id}')`; press feedback via `PressableScale`.

> **TC-MOB-048: Home respects top safe-area inset**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Notched device.
> - **Steps:** Open home.
> - **Expected:** Header content offset by `insets.top`; no clipping under the notch.

---

## Student — Search (`app/(student)/search.tsx`)

> **TC-MOB-050: Search calls discover/search with q**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Search screen.
> - **Steps:** Type a query.
> - **Expected:** `GET /discover/search?q=<query>` fires; input autofocuses, `returnKeyType=search`.

> **TC-MOB-051: Live Now filter adds live_now=true**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Search screen.
> - **Steps:** Toggle "Live Now" chip.
> - **Expected:** Query gains `&live_now=true`; chip shows green dot + active style; re-tap removes it.

> **TC-MOB-052: Rating 4+ filter adds min_rating=4**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Search screen.
> - **Steps:** Toggle "Rating 4+" chip.
> - **Expected:** Query gains `&min_rating=4`; chip active style; re-tap removes it. Filters compose with `q`.

> **TC-MOB-053: Empty results state**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Query with no matches.
> - **Steps:** Search a nonsense term.
> - **Expected:** `EmptyState` shown (e.g. "No educators found for '<query>'"). The empty-query case is now the suggested list (TC-MOB-217), not a placeholder.

> **TC-MOB-217: Empty search defaults to suggested educators**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Open the search screen with no query and no filters applied.
> - **Steps:** Observe the request and the list.
> - **Expected:** The SWR key is **`/discover/suggested`**, so the default view is the personalised top-6 — previously this screen showed nothing at all until the student typed. Typing a query, or applying "Live Now" / "Rating 4+", switches the key to `/discover/search?q=…`; clearing both switches back. Parity with web (TC-WEB-134/135).

> **TC-MOB-054: Search result → profile + press feedback**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Results present.
> - **Steps:** Tap a result.
> - **Expected:** `router.push('/(student)/educator/{id}')`; `PressableScale` on cards.

> **TC-MOB-055: Search back button**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Search screen.
> - **Steps:** Tap back.
> - **Expected:** `router.back()`; back control ≥44px hit area.

---

## Student — Educator profile (`app/(student)/educator/[id].tsx`)

> **TC-MOB-060: Profile fetch**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Valid educator id.
> - **Steps:** Open profile.
> - **Expected:** `GET /discover/{id}` returns `{ educator, reviews }`; hero renders name, avatar, rating; nothing crashes while `educator` is null (hero renders null).

> **TC-MOB-061: Topics expand/collapse**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Educator with >3 topics.
> - **Steps:** Tap "+ N more".
> - **Expected:** First 3 topics visible initially; tapping expands to all; collapse control returns to 3.

> **TC-MOB-062: Rate cards for three session types**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** Educator with rates.
> - **Steps:** View rate section.
> - **Expected:** Teaching / Revision / Doubt Clearing rate cards shown with per-hour prices.

> **TC-MOB-063: Rating breakdown bars**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Reviews present.
> - **Steps:** View rating section.
> - **Expected:** Star-distribution bars render with overall rating + count.

> **TC-MOB-064: Only approved reviews listed**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** `/discover/{id}` returns only approved reviews (server filtered).
> - **Steps:** View reviews list.
> - **Expected:** Each review shows rating + comment + date; pending/rejected reviews never appear.

> **TC-MOB-065: Booking bottom sheet — topic/type/duration**
> - **Type:** unit
> - **Priority:** P0
> - **Preconditions:** Profile loaded.
> - **Steps:** Tap "Book a Session".
> - **Expected:** Slide-up modal with topic chips, session-type segmented (Teaching/Revision/Doubt Clearing), duration segmented (30/60/90). Confirm disabled until a topic is chosen.

> **TC-MOB-066: Booking sheet price computation**
> - **Type:** unit
> - **Priority:** P0
> - **Preconditions:** Booking sheet open.
> - **Steps:** Select session type + duration.
> - **Expected:** Price = rate × duration multiplier (0.5×/1×/1.5× for 30/60/90); CTA reads "Confirm · ₹<price>".

> **TC-MOB-067: Booking sheet confirm navigates to slot picker**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Topic selected.
> - **Steps:** Tap Confirm.
> - **Expected:** `router.push('/(student)/book/{educatorId}?session=…&duration=…&topicId=…&topicName=…')`.

> **TC-MOB-068: Demo clip playback modal**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Educator has demo clips.
> - **Steps:** Tap a clip; then close.
> - **Expected:** Fullscreen 16:9 video modal opens with play/duration badge; close returns to profile.

---

## Student — Slot picker (`app/(student)/book/[educatorId]/index.tsx`)

> **TC-MOB-070: Fetch educator + availability by date**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** From booking sheet with params.
> - **Steps:** Open slot picker.
> - **Expected:** `GET /educator/{educatorId}` for name/rates/live; `GET /educator/{educatorId}/availability?date=YYYY-MM-DD` for slots.

> **TC-MOB-071: 7-day horizontal date selector, past disabled**
> - **Type:** unit
> - **Priority:** P0
> - **Preconditions:** Slot picker.
> - **Steps:** Scroll the date rail.
> - **Expected:** 7-day window shown; past days at opacity 0.3 and non-tappable; selecting a day resets the chosen slot.

> **TC-MOB-072: Slots grid + empty state**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** Date selected.
> - **Steps:** View slots.
> - **Expected:** Available slots in a 2-column grid; if none, "No slots — unavailable" empty state.

> **TC-MOB-073: Live Now slot styling**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Educator is live now.
> - **Steps:** View slots.
> - **Expected:** A distinct "LIVE NOW" slot (green border, 📹, larger height) appears above regular slots.

> **TC-MOB-074: Slot selection + press feedback**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** Slots present.
> - **Steps:** Tap a slot.
> - **Expected:** Slot marked active (thick border + tint); other slots deselect; `PressableScale` feedback.

> **TC-MOB-075: Sticky bottom summary + Confirm & Pay**
> - **Type:** unit
> - **Priority:** P0
> - **Preconditions:** Slot picker.
> - **Steps:** Select a slot.
> - **Expected:** Sticky bottom bar shows summary; "Confirm & Pay" enabled only when a slot is selected.

> **TC-MOB-076: Confirm & Pay navigation + price**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Slot selected.
> - **Steps:** Tap "Confirm & Pay".
> - **Expected:** `router.push('/(student)/confirm-booking?…scheduledAt=<isoUTC>&durationMinutes=…&sessionType=…&price=…')`; price = ratePerHour × durationMinutes / 60 (rounded).

---

## Student — Confirm booking (`app/(student)/confirm-booking.tsx`)

> **TC-MOB-080: Summary renders from params**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** Arrived from slot picker.
> - **Steps:** Open confirm.
> - **Expected:** `GET /discover/{educatorId}` for header; card shows avatar, name, rating, date/time/duration, session type, price with optional credits + total due.

> **TC-MOB-081: Send booking request**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Idle state.
> - **Steps:** Tap "Send Booking Request".
> - **Expected:** `POST /bookings` `{ educatorId, topicId, scheduledAt, durationMinutes, sessionType, creditsToApply }`; button shows spinner; state → `awaiting_educator`.

> **TC-MOB-082: Awaiting-educator state + poll**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Request sent.
> - **Steps:** Wait.
> - **Expected:** "Request Sent!" card; polls `GET /bookings/{id}` every ~4s; on `pending_payment` navigates to payment; on `cancelled` shows error state.

> **TC-MOB-083: Awaiting-state secondary actions**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** Awaiting state.
> - **Steps:** Tap "Go to Home" / "View My Sessions".
> - **Expected:** Navigates home / to sessions; background poll continues consistently.

> **TC-MOB-084: Booking creation error**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** `POST /bookings` fails.
> - **Steps:** Send request.
> - **Expected:** Error message shown; user can retry; no navigation.

---

## Student — Payment (`payment.tsx`, `payment-pending.tsx`, `booking-confirmed.tsx`, `booking-failed.tsx`)

> **TC-MOB-090: Payment intent + UPI URI**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Payment screen with bookingId.
> - **Steps:** Open payment.
> - **Expected:** `POST /payment/{bookingId}/intent` returns `upiUri`; "Open UPI App" triggers `Linking.openURL(upiUri)`; UPI app icons (GPay/PhonePe/BHIM/Paytm) displayed.

> **TC-MOB-091: Payment status polling every 3s**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Payment in progress.
> - **Steps:** Wait for confirmation.
> - **Expected:** `GET /payment/{bookingId}/status` polled every 3s; on `confirmed` → `booking-confirmed`; on `failed` → `booking-failed`.

> **TC-MOB-092: 5-minute countdown → failure**
> - **Type:** unit
> - **Priority:** P0
> - **Preconditions:** Payment screen open.
> - **Steps:** Let timer run to 0.
> - **Expected:** Countdown starts at 300s (MM:SS); at 0 auto-navigates to `booking-failed`.

> **TC-MOB-093: Dev mock-confirm**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** `__DEV__` build.
> - **Steps:** Tap "Simulate Payment Success".
> - **Expected:** `POST /bookings/{bookingId}/mock-confirm` (or `/payment/{id}/mock-confirm`); next poll marks confirmed → confirmed screen. Button hidden in production.

> **TC-MOB-094: Booking confirmed screen**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Confirmed.
> - **Steps:** Land on confirmed.
> - **Expected:** `GET /discover/{educatorId}` for header; success hero; date/time/duration cards; booking id `#XXXXXXXX`; "View My Sessions" → `replace('/(student)/sessions')`; "Back to Home" → `replace('/(student)/')`.

> **TC-MOB-095: Add to Calendar permission flow**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** Confirmed screen.
> - **Steps:** Tap "Add to Calendar".
> - **Expected:** Requests calendar permission; on grant creates event with 15-min reminder + success alert; on denial handled gracefully (no crash).

> **TC-MOB-096: Booking failed screen**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Failure path.
> - **Steps:** Land on failed.
> - **Expected:** Error hero "Booking Unsuccessful"; `GET /educator/search?sort=rating&limit=4` alternatives grid; "Explore Educators" → search; educator card → profile.

> **TC-MOB-097: payment-pending countdown animation**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** payment-pending screen.
> - **Steps:** Open screen.
> - **Expected:** Rotating spinner ring (1.2s loop, native driver) + countdown; "Cancel Transaction" → `router.back()`.

---

## Student — Sessions (`app/(student)/sessions.tsx`)

> **TC-MOB-100: Upcoming/Past tabs fetch by status**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Sessions screen.
> - **Steps:** Switch tabs.
> - **Expected:** Upcoming → `GET /bookings/my?status=awaiting_educator,pending_payment,confirmed`; Past → `status=completed,cancelled,disputed`. Segmented control active tab elevated.

> **TC-MOB-101: Empty state per tab**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** No bookings in a tab.
> - **Steps:** Open that tab.
> - **Expected:** `EmptyState` with appropriate icon/title/subtitle.

> **TC-MOB-102: StatusPill on each card**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** Bookings of varied status.
> - **Steps:** Inspect cards.
> - **Expected:** `StatusBadge` matches canonical color/label mapping per status.

> **TC-MOB-103: pending_payment → Pay Now**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** A `pending_payment` booking.
> - **Steps:** Tap "Pay Now to Confirm".
> - **Expected:** `router.push('/(student)/payment?bookingId=…&educatorId=…')`.

> **TC-MOB-104: confirmed → Join gating**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** A `confirmed` booking.
> - **Steps:** Inspect join action.
> - **Expected:** "Join Session" enabled only within 5 min before start or during session (`isJoinable`); tap → `/(student)/session/{bookingId}`. Otherwise disabled.

> **TC-MOB-105: awaiting_educator card action**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** `awaiting_educator` booking.
> - **Steps:** Inspect card.
> - **Expected:** Non-actionable "Waiting for educator…" with warning badge.

> **TC-MOB-106: Past — Rate Session when no review**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Completed booking, not yet reviewed.
> - **Steps:** Tap "Rate Session".
> - **Expected:** `router.push('/(student)/review/{bookingId}')`.

> **TC-MOB-107: Past — recording within 24h**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** Completed within 24h.
> - **Steps:** Tap "Watch Recording".
> - **Expected:** `GET /recordings/{bookingId}` returns url; recording opens. Cards outside 24h omit the button.

> **TC-MOB-108: Cancelled → raise dispute**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Cancelled booking.
> - **Steps:** Tap "Raise a Dispute".
> - **Expected:** Opens mailto/support flow; disputed bookings show "Dispute in progress…".

---

## Student — Live session (`app/(student)/session/[bookingId].tsx`)

> **TC-MOB-110: Expo Go shows Dev Build Required**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Running in Expo Go (Zoom native module absent).
> - **Steps:** Open session.
> - **Expected:** "Dev Build Required" notice (Zoom SDK lazy-loaded); no crash; no join attempt.

> **TC-MOB-111: Token + join calls (dev build)**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Dev build; joinable confirmed booking.
> - **Steps:** Open session.
> - **Expected:** `GET /session/{bookingId}/token` → session token/name; `POST /session/{bookingId}/join`; "Joining session…" spinner until joined.

> **TC-MOB-112: OTP badge via join response / FCM foreground listener**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** In session.
> - **Steps:** Session end code delivered.
> - **Expected:** "SESSION END CODE" badge shows OTP obtained from join response or FCM foreground message (fallback `GET /sessions/{bookingId}/otp`); helper text "Educator needs this code to end session".

> **TC-MOB-113: Countdown color coding**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** In session.
> - **Steps:** Let time decrement.
> - **Expected:** Countdown badge green >10 min, amber 5–10 min, red <5 min (MM:SS).

> **TC-MOB-114: Session controls**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** In session.
> - **Steps:** Toggle mic/video; tap leave.
> - **Expected:** Mic toggles Zoom audio mute; video toggles camera; Leave shows confirm alert → `leaveSession(false)` → back to sessions.

> **TC-MOB-115: Extend flow**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** ≤10 min remaining.
> - **Steps:** Tap Extend; choose 30/60/90; confirm.
> - **Expected:** Extend button appears when time low; modal offers 30/60/90 with cost; `GET /bookings/{bookingId}/extend?minutes=N` → `upiUri` → UPI open → after delay `POST /bookings/{bookingId}/extend/confirm`; countdown extends.

> **TC-MOB-116: Auto end at 0**
> - **Type:** unit
> - **Priority:** P0
> - **Preconditions:** Countdown reaches 0.
> - **Steps:** Wait.
> - **Expected:** Alert "Session time has ended"; auto-leave session.

---

## Student — Review, Wallet, Referral, Profile, Onboarding

> **TC-MOB-120: Review — star selection + label**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** Review screen.
> - **Steps:** Tap stars.
> - **Expected:** 5 large stars (52px); labels map 1→Poor,2→Fair,3→Good,4→Very Good,5→Excellent; filled amber vs gray outline.

> **TC-MOB-121: Review submit → pending**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Rating ≥1.
> - **Steps:** Add comment (≤1000), tap Submit.
> - **Expected:** `POST /review { bookingId, rating, comment }`; thank-you alert; `replace('/(student)/sessions')`. Review created as pending (not yet public). Submit disabled while rating=0.

> **TC-MOB-122: Review skip**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** Review screen.
> - **Steps:** Tap "Skip for now".
> - **Expected:** No POST; `replace('/(student)/sessions')`.

> **TC-MOB-123: Report educator**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Review screen.
> - **Steps:** Open report modal, pick reason, submit.
> - **Expected:** Reasons list (inappropriate_behavior/harassment/fraud/no_show/other); `POST /review/report { educatorId, reason }`; "reviewed within 48 hours" alert; modal closes.

> **TC-MOB-124: Comment char counter**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Review screen.
> - **Steps:** Type in comment.
> - **Expected:** Counter shows X/1000; input capped at 1000 chars.

> **TC-MOB-125: Wallet balance + transactions**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Wallet screen.
> - **Steps:** Open wallet.
> - **Expected:** `GET /student/wallet` (balance) + `GET /student/wallet/transactions`; loading spinner first; balance card + transaction rows (credit green ↑ / debit red ↓).

> **TC-MOB-126: Wallet empty transactions**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** No transactions.
> - **Steps:** Open wallet.
> - **Expected:** Empty state ("No transactions yet").

> **TC-MOB-127: Wallet txn label mapping**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Transactions of varied reference prefixes.
> - **Steps:** Inspect labels.
> - **Expected:** `booking:*`→"Session booked", `cancellation_refund:*`→"Cancellation refund", `dispute_refund:*`→"Dispute refund", `admin_grant:*`→"Bonus credit", `referral_by:*`→"Referral bonus", `referred_by:*`→"Welcome bonus".

> **TC-MOB-128: Referral share**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** User has referral code.
> - **Steps:** Open referral, tap Share Code.
> - **Expected:** `GET /referral/me/stats` (count + bonus); code card; `Share.share` invoked with message containing code.

> **TC-MOB-129: Profile logout confirm**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Student profile.
> - **Steps:** Tap Logout, confirm.
> - **Expected:** `GET /student/me` for header; destructive-style confirm alert; on confirm clears SWR cache + `logout()` → `replace('/(auth)/welcome')`.

> **TC-MOB-130: Profile refer entry**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** Profile screen.
> - **Steps:** Tap "Refer & Earn".
> - **Expected:** `router.push('/(student)/referral')`.

> **TC-MOB-131: Onboarding tutorial carousel + mark seen**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** New student.
> - **Steps:** Page through slides; finish or skip.
> - **Expected:** 3 slides with dots; last CTA "Let's Start"; finishing/skipping calls `markOnboardingSeen()` → `POST /student/onboarding-seen` → `/(student)`.

> **TC-MOB-132: Student profile setup validation**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** New student at `(student)/onboarding`.
> - **Steps:** Submit form.
> - **Expected:** `GET /config/onboarding` for grades/boards; requires name + grade level + ≥1 exam board + terms accepted; "Complete Profile" disabled until valid; on submit `register({role:'student',…})` (x-temp-token header) + `POST /auth/terms/accept` → tutorial.

---

## Educator — Dashboard (`app/(educator)/index.tsx`)

> **TC-MOB-140: Dashboard fetch**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Approved educator.
> - **Steps:** Open dashboard.
> - **Expected:** `GET /educator/me` returns profile, today schedule, recent reviews, stats. Loading spinner while pending.

> **TC-MOB-141: Go-live toggle**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Dashboard.
> - **Steps:** Toggle the switch.
> - **Expected:** `PATCH /educator/go-live { isLive }`; optimistic UI + revalidate; label "Go Live"↔"You are Live"; success track green; pulse animation runs while live.

> **TC-MOB-142: Bento stats cards**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Dashboard loaded.
> - **Steps:** View stats.
> - **Expected:** This-month earnings (dark card), sessions this month, rating + review count with star.

> **TC-MOB-143: Today schedule empty**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** No sessions today.
> - **Steps:** View schedule.
> - **Expected:** `EmptyState` "No sessions today" / "Go live to start accepting bookings…".

> **TC-MOB-144: Today schedule Join routes to guard**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** A live session in schedule.
> - **Steps:** Tap "Join".
> - **Expected:** `router.push('/(educator)/session/{id}')` — which shows the web-app notice (see TC-MOB-170). `PressableScale` on Join.

> **TC-MOB-145: Recent reviews empty**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** No reviews.
> - **Steps:** View reviews rail.
> - **Expected:** `EmptyState` "No reviews yet".

> **TC-MOB-146: Notifications entry**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** Dashboard.
> - **Steps:** Tap the bell.
> - **Expected:** `router.push('/(educator)/notifications')`.

---

## Educator — Calendar (`app/(educator)/calendar.tsx`)

> **TC-MOB-150: Availability fetch + local display**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Calendar screen.
> - **Steps:** Open calendar.
> - **Expected:** `GET /educator/me/availability`; stored UTC times converted via `utcTimeToLocal` before display (12-hour format).

> **TC-MOB-151: Empty day**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** A day with no slots.
> - **Steps:** View that day.
> - **Expected:** `EmptyState` "No slots — unavailable".

> **TC-MOB-152: Add slot converts local→UTC**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Add-slot sheet open with start/end.
> - **Steps:** Confirm add.
> - **Expected:** `POST /educator/me/availability { dayOfWeek, startTime, endTime }` with `localTimeToUtc`-converted times.

> **TC-MOB-153: Add slot validation**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** Add-slot sheet.
> - **Steps:** Pick end ≤ start; confirm.
> - **Expected:** Alert "Invalid — End time must be after start time"; no POST.

> **TC-MOB-154: Delete slot confirm**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** A slot exists.
> - **Steps:** Tap ✕; confirm.
> - **Expected:** Confirm alert; on Delete `DELETE /educator/me/availability/{slotId}`; row shows spinner while deleting; revalidates.

> **TC-MOB-155: Add-slot button hit area**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Calendar.
> - **Steps:** Inspect "+ Add slot".
> - **Expected:** `hitSlop` 8px each side to reach ≥44px tap target.

---

## Educator — Earnings (`app/(educator)/earnings.tsx`)

> **TC-MOB-160: Earnings fetch**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Earnings screen.
> - **Steps:** Open earnings.
> - **Expected:** `GET /earning/me/earnings` + `GET /earning/me/payouts`; loading spinner; hero this-month net + gross + commission %; all-time stat cards.

> **TC-MOB-161: Payout history + status badges**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Payouts present.
> - **Steps:** View list.
> - **Expected:** Rows show amount (₹ en-IN), date, status badge — paid=green, pending=amber.

> **TC-MOB-162: No payouts empty**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** No payouts.
> - **Steps:** View list.
> - **Expected:** `EmptyState` "No payouts yet" + threshold subtitle.

---

## Educator — Profile & Edit (`profile.tsx`, `edit-profile.tsx`)

> **TC-MOB-165: Profile content + approved reviews**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Educator profile.
> - **Steps:** Open profile.
> - **Expected:** `GET /educator/me`; avatar (selfie or initials); status badge (approved=green "Approved Educator" / pending=amber); stats (rating, sessions, earnings); recent approved reviews (or "No reviews yet"); bank preview masked or "No bank account added yet."

> **TC-MOB-166: Profile logout**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Educator profile.
> - **Steps:** Tap Logout, confirm.
> - **Expected:** Destructive confirm; `POST /auth/logout`; clear SWR cache; `logout()`; `replace('/(auth)/welcome')`.

> **TC-MOB-167: Profile settings navigation**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** Educator profile.
> - **Steps:** Tap Edit Profile / Bank Account.
> - **Expected:** → `/(educator)/edit-profile` and `/(educator)/payout-setup` respectively.

> **TC-MOB-168: Edit profile save**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Edit-profile screen.
> - **Steps:** Change bio/city/rates/topics; Save.
> - **Expected:** `GET /educator/me` + `GET /subjects` prefill; `PATCH /educator/profile { bio, city, rateTeaching, rateRevision, rateDoubtClearing, topicIds }`; saving spinner; success alert → back to profile. Topics capped at 5.

> **TC-MOB-169: Teaching video upload/delete — max 3**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** Edit-profile screen, "Teaching Videos" section (hint: "up to 3 videos, 5 min max each").
> - **Steps:** Add a video; then remove one.
> - **Expected:** `ImagePicker.launchImageLibraryAsync({ mediaTypes: Videos })` → `POST /upload?purpose=demo-clip` (FormData, `demo.mp4`) → `POST /educator/demo-clips { storageKey, durationSeconds, type:'demo' }`; the new clip is appended to the local list showing "{n}s clip"; delete → `DELETE /educator/demo-clips/{id}` and removes it locally. The **"+ Add Video" button is hidden entirely once `demoClips.length >= 3`**, so the limit is reached without ever hitting the server's 422.

> **TC-MOB-211: Dedicated Intro Video upload warns about re-review**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Approved educator on edit-profile.
> - **Steps:** View the "Intro Video" section; upload a video under 5 minutes.
> - **Expected:** Its own section above Teaching Videos, hinting that uploading a new intro "puts your profile under review until an admin re-approves it". Upload posts `type:'intro'` (file named `intro.mp4`); a spinner replaces the button label while `uploadingIntro`; on success an alert reads "Intro video uploaded — Your profile is now under review. You'll be visible to students again once an admin re-approves it." The intro is **not** appended to the teaching-clip list — it's a separate slot.

> **TC-MOB-212: Client-side 5-minute guard on every video upload**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** A gallery video longer than 5 minutes.
> - **Steps:** Pick it for the intro slot; repeat for a teaching video.
> - **Expected:** `asset.duration` is in **milliseconds**, so the check is `> 5 * 60 * 1000`; an over-length pick alerts "Video too long — Each video must be 5 minutes or shorter. Please trim it and try again." and returns **before** any upload — no `/upload` call, no `/educator/demo-clips` call, and the uploading flag is never set. Parity with the web guard (TC-WEB-139).

> **TC-MOB-213: Duration is sent in seconds**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** A 90-second video selected.
> - **Steps:** Complete the upload; inspect the request body.
> - **Expected:** `durationSeconds: Math.round(asset.duration / 1000)` = 90, not 90000. Sending milliseconds here would trip the server's 300-second limit on every real video — worth pinning.

> **TC-MOB-214: Cancelled picker and failed upload are non-destructive**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Edit-profile screen.
> - **Steps:** Open the picker and cancel; then force the upload to fail.
> - **Expected:** Cancel returns immediately with no state change and no request. A failure alerts "Could not upload intro video." or "Could not upload clip." depending on the type, and the matching uploading flag is cleared in `finally` so the button becomes usable again.

---

## Educator — Session guard (`app/(educator)/session/[bookingId].tsx`) — IMPORTANT

> **TC-MOB-170: Educator session screen shows web-app notice (no join)**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Any educator arriving at `(educator)/session/{bookingId}`.
> - **Steps:** Open the screen.
> - **Expected:** Renders "Join from the web app" notice with desktop icon and instructions ("educators join live sessions from the Mole web app on a computer" / "go to your Calendar to join"). A single "Go Back" button (`router.back()`).

> **TC-MOB-171: No Zoom SDK / token / join call on educator session**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Educator session screen.
> - **Steps:** Load screen; watch network + native modules.
> - **Expected:** No Zoom SDK load; NO `GET /session/{id}/token`; NO `POST /session/{id}/join`; no countdown/OTP/controls. Nothing but the notice + Go Back.

> **TC-MOB-172: Single guard covers dashboard Join, deep link, notification**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Educator, approved.
> - **Steps:** Enter the route three ways — (a) dashboard "Join", (b) deep link `mole://…/session/{id}`, (c) tapping a session push notification.
> - **Expected:** All three land on the same notice screen; none initiates a Zoom session; behavior is identical for every entry path.

> **TC-MOB-173: Go Back returns to previous screen**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** Educator session notice shown.
> - **Steps:** Tap "Go Back".
> - **Expected:** `router.back()`; button ≥48px hit area with press feedback.

---

## Educator — Onboarding, verification, payout, notifications

> **TC-MOB-180: Educator onboarding step1 validation**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** New educator at `(educator)/onboarding`.
> - **Steps:** Submit incomplete then complete.
> - **Expected:** `GET /config/onboarding`; required name/city/degree/institution/gradYear + ≥1 subject (max 2) + bio ≥10 chars + terms (new reg); toasts on each failure; on success `register({role:'educator'})` + `POST /auth/terms/accept` → next: identity check.

> **TC-MOB-181: Identity check upload sequence**
> - **Type:** integration
> - **Priority:** P0
> - **Preconditions:** Onboarded educator.
> - **Steps:** Upload education proof, govt ID, selfie, intro video; submit.
> - **Expected:** All four required (toast otherwise); uploads via image/video upload service; `POST /educator/onboarding/step2 { govtIdUrl, selfieUrl, educationProofUrl }`; intro `POST /educator/demo-clips { storageKey, durationSeconds, type:'intro' }`; progress modal ("Please keep the app open"); → `replace('/(educator)/under-verification')`.

> **TC-MOB-182: Identity check media permission**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Identity check screen.
> - **Steps:** Tap a picker with media permission denied.
> - **Expected:** Permission prompt; on denial no crash and no upload proceeds.

> **TC-MOB-183: Under-verification screen**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** `educatorStatus=pending`.
> - **Steps:** Land here via redirect.
> - **Expected:** Hourglass hero "Account Under Verification"; checklist (Professional Info, Identity Verified); no tab bar; cannot reach dashboard.

> **TC-MOB-184: Rejected screen shows reason**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** `educatorStatus=rejected`.
> - **Steps:** Land here.
> - **Expected:** `GET /educator/me`; "Application Rejected" hero; REASON card when reason present; numbered next-steps; no tab bar.

> **TC-MOB-185: Payout setup add/update/remove bank**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Payout-setup screen.
> - **Steps:** Add bank; edit; remove.
> - **Expected:** `GET /educator/me` prefill; `POST /educator/setup-bank { bankAccountName, bankAccountNumber, bankIfscCode, bankAccountType }`; account number masked with eye toggle; IFSC uppercased; type from Savings/Current/Salary; remove → confirm alert → `DELETE /educator/bank`.

> **TC-MOB-186: Educator notifications list/empty**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** Notifications screen.
> - **Steps:** Open.
> - **Expected:** `GET /educator/notifications`; loading spinner; unread rows tinted; empty → "No notifications yet."

---

## Cross-cutting — mobile-specific

> **TC-MOB-200: PressableScale animates 0.98 / 150ms**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** Any `PressableScale` (cards, CTAs, chips, slots, Join, Go Back).
> - **Steps:** Press and release.
> - **Expected:** Scales to 0.98 over 150ms on press-in, back to 1 on release; respects `disabled` (no scale when disabled).

> **TC-MOB-201: Interactive controls ≥44–48px hit area**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** Any screen with buttons/icons/chips.
> - **Steps:** Measure tappable controls.
> - **Expected:** Every interactive control ≥44px (iOS) / 48px (Android) via size or `hitSlop`; primary CTAs are 56px tall.

> **TC-MOB-202: Safe-area insets on notched devices**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** Notched device (iPhone with notch / Android cutout).
> - **Steps:** Open each screen; inspect top/bottom.
> - **Expected:** `useSafeAreaInsets`/`SafeAreaView` applied; no content under notch or home indicator; sticky bars respect `insets.bottom`.

> **TC-MOB-203: Tab bars use correct icons + safe-area**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Student and educator tab bars.
> - **Steps:** Inspect tabs.
> - **Expected:** Student = Home/Sessions/Wallet/Profile; educator = Home/Calendar/Earnings/Profile; active tint primary, inactive textSecondary; filled icon when active; height 56 + `insets.bottom`; gated screens hide the tab bar.

> **TC-MOB-204: StatusBadge canonical colors**
> - **Type:** unit
> - **Priority:** P1
> - **Preconditions:** Cards rendering `StatusBadge`.
> - **Steps:** Render each status.
> - **Expected:** Colors/labels match the canonical mapping in Conventions (confirmed=primary, completed=success, cancelled/failed/disputed=error, awaiting/pending/pending_payment=warning, expired=textSecondary).

> **TC-MOB-205: EmptyState for all empty lists**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Any empty list (search, sessions, wallet, schedule, reviews, payouts, notifications, availability day).
> - **Steps:** Trigger empty.
> - **Expected:** `EmptyState` (icon-in-indigoTint-circle + title + optional subtitle) instead of a blank screen.

> **TC-MOB-206: Offline / network error handling**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Airplane mode / server unreachable.
> - **Steps:** Open data-backed screens; retry after reconnect.
> - **Expected:** No unhandled crash; failed fetches surface an error/empty state; SWR revalidates on focus/reconnect and recovers.

> **TC-MOB-207: Keyboard avoidance on input screens**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** phone / otp / role / onboarding screens.
> - **Steps:** Focus an input.
> - **Expected:** `KeyboardAvoidingView` (padding on iOS, height on Android) keeps the active input and CTA visible.

> **TC-MOB-208: Deep-link routing respects role gating**
> - **Type:** integration
> - **Priority:** P1
> - **Preconditions:** Logged in as one role.
> - **Steps:** Open a deep link into the other role's stack.
> - **Expected:** Root layout redirect returns the user to their permitted group; unauthenticated deep links → `(auth)/welcome`.

> **TC-MOB-209: Press feedback on chips and slots**
> - **Type:** unit
> - **Priority:** P2
> - **Preconditions:** Home chips, search filter chips, slot-picker slots.
> - **Steps:** Press each.
> - **Expected:** Visible `PressableScale` feedback (0.98/150ms); active/selected style distinct from pressed feedback.

> **TC-MOB-210: SWR revalidate-on-focus refreshes stale data**
> - **Type:** integration
> - **Priority:** P2
> - **Preconditions:** Any SWR-backed screen backgrounded then refocused.
> - **Steps:** Background app, mutate data server-side, refocus.
> - **Expected:** With `revalidateOnFocus: true`, screen re-fetches and reflects updated data on return.

## Quiz (QUIZ)

### Educator Setup Screen

> **TC-QUIZ-001: Educator navigates to quiz setup screen**
> - **Type:** UI Navigation
> - **Priority:** P0
> - **Preconditions:** User is educator; booking detail screen loaded
> - **Steps:** Tap "Set Up Quiz" button
> - **Expected:** Quiz setup screen opens with 5 question cards; each card contains 4 option fields + correctIndex radio selector

> **TC-QUIZ-002: Quiz validation on save (missing question)**
> - **Type:** Form Validation
> - **Priority:** P0
> - **Preconditions:** Quiz setup screen open; question 2 left blank
> - **Steps:** Tap "Save"
> - **Expected:** Inline error "Question required" appears below question 2; Save button disabled (greyed out)

> **TC-QUIZ-003: Quiz validation on save (missing option)**
> - **Type:** Form Validation
> - **Priority:** P0
> - **Preconditions:** Quiz setup screen; all questions filled; question 1 has only 3 options
> - **Steps:** Tap "Save"
> - **Expected:** Inline error "All 4 options required" below question 1; Save button disabled

> **TC-QUIZ-004: Educator saves valid quiz on mobile**
> - **Type:** Happy Path
> - **Priority:** P0
> - **Preconditions:** Quiz setup screen; all 5 questions × 4 options filled; each correctIndex selected
> - **Steps:** Tap "Save"
> - **Expected:** Screen transitions back to booking detail; success toast "Quiz saved"; "Set Up Quiz" button replaced with "View Quiz" (read-only)

> **TC-QUIZ-005: Quiz setup screen load failure blocks save**
> - **Type:** Error Handling
> - **Priority:** P1
> - **Preconditions:** Quiz setup screen open; network error during load
> - **Steps:** Initial page load fails
> - **Expected:** Error screen "Could not load quiz"; "Save" button permanently disabled; "Back" button functional to return to booking

> **TC-QUIZ-006: Quiz locked after student attempt**
> - **Type:** State Transition
> - **Priority:** P0
> - **Preconditions:** Booking detail screen; quiz exists; student has attempted
> - **Steps:** Navigate to booking detail; attempt to tap "Edit Quiz"
> - **Expected:** "View Quiz" (read-only) button shown instead; tapping shows "Quiz locked — cannot edit after student attempt"

### Student Quiz Screen

> **TC-QUIZ-007: Student navigates to quiz from review screen**
> - **Type:** UI Navigation
> - **Priority:** P0
> - **Preconditions:** Booking in review phase (completed); quiz exists; student has not attempted
> - **Steps:** Navigate to session review screen; tap "Take Quiz"
> - **Expected:** Quiz screen loads; displays question 1/5 with 4 MCQ options; "Submit" button at bottom

> **TC-QUIZ-008: MCQ options are >= 44px tap targets**
> - **Type:** Accessibility
> - **Priority:** P0
> - **Preconditions:** Quiz screen open
> - **Steps:** Inspect MCQ option buttons in UI layout
> - **Expected:** Each option button min-height 44px; min-width 44px (or equivalent touch area); meets WCAG 2.1 AA

> **TC-QUIZ-009: Student cannot submit quiz with unanswered questions**
> - **Type:** Form Validation
> - **Priority:** P0
> - **Preconditions:** Quiz screen open; question 3 not selected
> - **Steps:** Tap "Submit"
> - **Expected:** Inline error "All 5 questions required"; Submit button disabled; scroll to unanswered question

> **TC-QUIZ-010: Student passes quiz — confetti + coins**
> - **Type:** Happy Path / UX Celebration
> - **Priority:** P0
> - **Preconditions:** Quiz screen loaded; student answers all 5 correctly
> - **Steps:** Answer all 5 MCQs; tap "Submit"
> - **Expected:** Confetti animation on screen; result modal "Passed! 5/5 +10 coins"; coin count updates in app header; "Done" button dismisses

> **TC-QUIZ-011: Student fails quiz — score shown, no coins**
> - **Type:** Edge Case / UX Feedback
> - **Priority:** P0
> - **Preconditions:** Quiz screen loaded; student answers 4 correct, 1 incorrect
> - **Steps:** Answer 4 MCQs correctly, 1 incorrectly; tap "Submit"
> - **Expected:** Result modal "Score: 4/5" (no confetti); no coin update; "Done" button available

> **TC-QUIZ-012: Attempt-consumed state after submit**
> - **Type:** State Transition
> - **Priority:** P0
> - **Preconditions:** Student submitted quiz (passed or failed)
> - **Steps:** Navigate back to session review screen
> - **Expected:** "Take Quiz" button replaced with "Quiz Completed — <score>/5" (disabled); no retry button

> **TC-QUIZ-013: Quiz submission network error on mobile**
> - **Type:** Error Handling
> - **Priority:** P1
> - **Preconditions:** Quiz screen; student completed all answers
> - **Steps:** Network fails during POST `/bookings/:id/quiz/attempt`
> - **Expected:** Error toast "Failed to submit quiz"; "Submit" button remains enabled; student can retry

> **TC-QUIZ-014: Quiz screen back button (no save)**
> - **Type:** Navigation
> - **Priority:** P1
> - **Preconditions:** Quiz screen open; student has answered some questions but not submitted
> - **Steps:** Tap back/close button before "Submit"
> - **Expected:** Modal confirmation "Discard answers?" appears; tapping "Discard" returns to review screen with no submission; tapping "Continue" returns to quiz

