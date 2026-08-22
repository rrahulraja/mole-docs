# Mole Admin App — Test Case Reference

Test cases for the **Admin** app (`admin/` — React 19 + Vite + Tailwind + TanStack Query + React Router).
Follows the format in [README.md](./README.md): `TC-ADM-NNN`, with Type / Priority / Preconditions / Steps / Expected.

Scope verified against `admin/src/pages/*`, `admin/src/components/*`, and `admin/src/api/*`.

**Conventions used below**
- **Auth:** **passwordless** — the admin enters an email and receives either a 6-digit code or a magic link; there is no password field anywhere in the app. The resulting JWT is stored in `localStorage` under key `admin_token`. All app routes are guarded by `RequireAuth`.
- **StatusPill tones (from `Badge.tsx`):** indigo = `pending`/`under_review`/`confirmed`; green = `approved`/`completed`/`resolved`/`active`; amber = `pending_payment`/`awaiting_educator`/`open`/`at-risk`; red = `rejected`/`failed`/`disputed`/`cancelled`; slate = `suspended`/`dismissed`/`inactive`/`graduated`. `pending` renders label **"Under Review"**.
- **Tables (`DataTable.tsx`):** `TableCard` scrolls with `max-h-[calc(100vh-16rem)]`; `TableHead` is `sticky top-0`; `TableRow` shows `hover:bg-primary-tint` (or `bg-primary-tint` when `selected`). Sortable headers show `↕` (idle) / `▲` (asc) / `▼` (desc) and the active glyph is indigo.
- **Mono formatting:** IDs, phones, amounts, dates use `font-mono`; amounts prefixed `₹` and formatted `toLocaleString('en-IN')`.
- **Client-side only:** search, sort, pagination, and CSV export in Educators operate on the already-fetched list; refetch happens per status tab via query key.

---

## Auth & Login (`/login`, `RequireAuth`, Sign out)

The login screen is a small state machine: `step` is `request` → `otp-sent` | `magic-sent`,
with a `method` toggle between "Email code" and "Magic link". There is no password input.

### TC-ADM-001: Login page renders the passwordless request step
- **Type:** e2e
- **Priority:** P0
- **Preconditions:** No `admin_token` in `localStorage`.
- **Steps:** Navigate to `/login`.
- **Expected:** "Mole Admin" card with the subtitle "Sign in with your admin email", a two-option segmented toggle ("Email code" / "Magic link", `otp` selected by default), one autofocused `type=email` field, and a "Send code" button. **No username field and no password field exist** — their presence is a regression.

### TC-ADM-002: Send button disabled until an email is entered
- **Type:** unit
- **Priority:** P1
- **Preconditions:** On `/login`, request step.
- **Steps:** Leave the email empty; then type an address.
- **Expected:** Button `disabled` (opacity-50) while `loading || !email`; enables once the field is non-empty. Its label follows the toggle: "Send code" for `otp`, "Send magic link" for `magic`.

### TC-ADM-003: Requesting a code advances to the OTP step with a generic message
- **Type:** e2e
- **Priority:** P0
- **Preconditions:** On `/login`, method `otp`, an email entered.
- **Steps:** Click "Send code".
- **Expected:** `requestOtp(email.trim())` fires; `step` → `otp-sent`; info banner reads "If an admin account exists for that email, a 6-digit code is on its way." — deliberately non-committal, so the UI never confirms whether an account exists. The 6-digit input (numeric, `maxLength=6`, letter-spaced, autofocused) and a "Use a different email" link render.

### TC-ADM-004: OTP entry accepts digits only and gates submit on length 6
- **Type:** unit
- **Priority:** P1
- **Preconditions:** On the `otp-sent` step.
- **Steps:** Type letters and punctuation, then five digits, then a sixth.
- **Expected:** Non-digits are stripped on change (`replace(/\D/g,'')`) and input is capped at 6; "Sign in" stays disabled while `code.length !== 6`; Enter submits only at exactly 6 digits.

### TC-ADM-005: Successful code verification stores the token and redirects
- **Type:** e2e
- **Priority:** P0
- **Preconditions:** A valid 6-digit code entered.
- **Steps:** Click "Sign in".
- **Expected:** `verifyOtp(email, code)` returns `{ token }`; `localStorage.admin_token` set; navigates to `/`. Button reads "Verifying…" and is disabled while in flight.

### TC-ADM-006: Invalid or expired code shows an inline error and stays put
- **Type:** e2e
- **Priority:** P0
- **Preconditions:** On the `otp-sent` step.
- **Steps:** Submit a wrong code.
- **Expected:** Red banner "Invalid or expired code."; no token stored; still on `/login` at the `otp-sent` step so the admin can retype without re-requesting.

### TC-ADM-111: Magic-link request shows the check-your-inbox step
- **Type:** e2e
- **Priority:** P0
- **Preconditions:** On `/login`; switch the toggle to "Magic link".
- **Steps:** Enter an email; click "Send magic link".
- **Expected:** `requestMagicLink(email.trim())` fires; `step` → `magic-sent`; the same generic "If an admin account exists…" wording; copy states the link "expires in 15 minutes"; no code input is shown; "Use a different email" returns to the request step.

### TC-ADM-112: Landing on `/login?token=…` completes sign-in automatically
- **Type:** e2e
- **Priority:** P0
- **Preconditions:** A valid magic-link token; not signed in.
- **Steps:** Open `/login?token=<raw>`.
- **Expected:** On mount, `verifyMagicLink(token)` fires without any user interaction; on success `admin_token` is stored, **the token is stripped from the URL** via `history.replaceState({}, '', '/')` so it can't be copied out of the address bar or leak via a referrer, then navigates to `/`.

### TC-ADM-113: Expired or invalid magic link shows a recoverable error
- **Type:** e2e
- **Priority:** P0
- **Preconditions:** A used, expired or fabricated token.
- **Steps:** Open `/login?token=<bad>`.
- **Expected:** Error banner "This sign-in link is invalid or has expired. Request a new one."; no token stored; the request step remains usable so the admin can ask for a new link.

### TC-ADM-114: "Use a different email" resets without clearing the address
- **Type:** unit
- **Priority:** P2
- **Preconditions:** On the `otp-sent` or `magic-sent` step.
- **Steps:** Click "Use a different email".
- **Expected:** `reset()` sets `step` back to `request` and clears `code`, `error` and `info`. Note it deliberately leaves `email` populated, so a typo is edited rather than retyped.

### TC-ADM-007: RequireAuth redirects unauthenticated access
- **Type:** integration
- **Priority:** P0
- **Preconditions:** No `admin_token`.
- **Steps:** Navigate directly to a guarded route (e.g. `/educators`, `/payouts`).
- **Expected:** `RequireAuth` renders `<Navigate to="/login" replace>`; user lands on `/login`.

### TC-ADM-008: RequireAuth allows authenticated access
- **Type:** integration
- **Priority:** P0
- **Preconditions:** `admin_token` present.
- **Steps:** Navigate to `/` and any child route.
- **Expected:** `Layout` (sidebar + top bar + outlet) renders the page.

### TC-ADM-009: Sign out clears token and redirects
- **Type:** e2e
- **Priority:** P0
- **Preconditions:** Authenticated, sidebar visible.
- **Steps:** Click "Sign out" at the bottom of the sidebar.
- **Expected:** `localStorage.admin_token` removed; navigates to `/login`; guarded routes are no longer reachable.

---

## Dashboard / Overview (`/`)

### TC-ADM-010: Dashboard loading state
- **Type:** unit
- **Priority:** P2
- **Preconditions:** `getDashboardStats` in flight.
- **Steps:** Load `/`.
- **Expected:** "Loading…" text shows while `isLoading`; metric grid appears after resolve.

### TC-ADM-011: Four primary metric cards render with values
- **Type:** integration
- **Priority:** P1
- **Preconditions:** Stats resolved.
- **Steps:** View the top card row.
- **Expected:** Cards: **Active Educators** (`totalEducators`), **Sessions This Week** (`sessionsThisMonth ?? totalSessions`), **Pending Educator Verifications** (`pendingApproval`), **Payouts Due (₹)** (`₹` + `pendingPayouts` en-IN formatted). Missing fields default to `0`.

### TC-ADM-012: Metric delta badges render with direction & color
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Cards rendered.
- **Steps:** Inspect each card's delta badge.
- **Expected:** Delta shows `▲`/`▼` + `|delta|.toFixed(1)%`. Positive metrics green when delta>0; `invertDelta` cards (Pending Verifications, Payouts Due) treat negative delta as green (good). `delta===0` renders neutral slate with no arrow.

### TC-ADM-013: Sparklines render (presentational/deterministic)
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Cards rendered.
- **Steps:** Inspect each card's sparkline.
- **Expected:** A `Sparkline` renders when `spark.length > 1`; shape is derived from `trendFor(seed)` (deterministic). Assert the SVG renders — do **not** assert data values.

### TC-ADM-014: Secondary aggregate cards render
- **Type:** integration
- **Priority:** P2
- **Preconditions:** Stats resolved.
- **Steps:** View the second card row.
- **Expected:** Cards: Live Now (`liveNow`), Total Sessions (`totalSessions`), Revenue This Month ₹ (`revenueThisMonth` en-IN), Total Educators (`totalEducators`). No delta/sparkline on these.

### TC-ADM-015: Values use mono tabular numerals
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Cards rendered.
- **Steps:** Inspect the metric value typography.
- **Expected:** Numeric value uses `font-mono tabular-nums`.

---

## Educators — Verification (`/educators`)

### TC-ADM-016: Default status tab is Pending
- **Type:** integration
- **Priority:** P1
- **Preconditions:** Authenticated.
- **Steps:** Open `/educators`.
- **Expected:** "pending" tab is active (indigo `bg-primary`); `getEducators('pending')` fetched; header shows `{filtered.length} in queue`.

### TC-ADM-017: Switch status tabs refetches per status
- **Type:** integration
- **Priority:** P1
- **Preconditions:** On `/educators`.
- **Steps:** Click approved / rejected / suspended tabs.
- **Expected:** Query key `['educators', status]` changes, refetches; page resets to 1 and selection clears (`resetToFirstPage`).

### TC-ADM-018: Loading state
- **Type:** unit
- **Priority:** P2
- **Preconditions:** `getEducators` in flight.
- **Steps:** Load a tab.
- **Expected:** "Loading…" shows while `isLoading`.

### TC-ADM-019: Empty state for a tab with no results
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Selected status has no educators, no search term.
- **Steps:** View list.
- **Expected:** EmptyState title "No educators match these filters"; hint `There are no educators with status "{status}".`

### TC-ADM-020: Empty state when search excludes all rows
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Tab has rows; search term matches none.
- **Steps:** Type a non-matching search string.
- **Expected:** EmptyState shows with hint "Try clearing the search or switching status tabs."

### TC-ADM-021: Search filters by name, phone, or city
- **Type:** integration
- **Priority:** P1
- **Preconditions:** Tab has multiple educators.
- **Steps:** Type a substring matching a name, phone, or city (case-insensitive).
- **Expected:** Only matching rows show; page resets to 1; queue count updates to filtered length.

### TC-ADM-022: Sortable header toggles asc/desc
- **Type:** unit
- **Priority:** P1
- **Preconditions:** List has ≥2 rows.
- **Steps:** Click "Name" header once, then again.
- **Expected:** First click sets sortKey=name, dir=asc (`▲`); second click flips to desc (`▼`). Other sortable columns: City, Status, Rating, Joined. Phone/Actions are not sortable.

### TC-ADM-023: Switching sort column defaults to asc
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Currently sorted by Name desc.
- **Steps:** Click "Rating" header.
- **Expected:** sortKey=rating, dir=asc; Name header reverts to idle `↕`.

### TC-ADM-024: Rating sort handles missing ratings
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Some rows have no `overallRating`.
- **Steps:** Sort by Rating.
- **Expected:** Missing ratings treated as `0`; no crash; cells render `—`.

### TC-ADM-025: Pagination at 12 per page
- **Type:** integration
- **Priority:** P1
- **Preconditions:** Filtered list has >12 rows.
- **Steps:** View footer, click Next / Prev.
- **Expected:** `PAGE_SIZE = 12`; footer shows `{start}–{end} of {total}` and `{page} / {totalPages}`. Prev disabled on page 1; Next disabled on last page.

### TC-ADM-026: Page clamps when filter shrinks list
- **Type:** unit
- **Priority:** P2
- **Preconditions:** On page 3, then search narrows results to 1 page.
- **Steps:** Type a search term.
- **Expected:** `safePage = min(page, totalPages)`; displayed slice is valid (no blank page).

### TC-ADM-027: Row name / Profile-KYC navigates to detail
- **Type:** e2e
- **Priority:** P1
- **Preconditions:** List has rows.
- **Steps:** Click the educator's name, or the "Profile / KYC" action.
- **Expected:** Navigates to `/educators/{userId}`.

### TC-ADM-028: Approve row action opens confirm dialog
- **Type:** integration
- **Priority:** P0
- **Preconditions:** A `pending` educator row.
- **Steps:** Click "Approve".
- **Expected:** Confirm Modal "Approve educator" opens with the educator name and irreversibility warning; approve only fires on the modal's Approve button.

### TC-ADM-029: Confirm approve mutates and invalidates
- **Type:** e2e
- **Priority:** P0
- **Preconditions:** Approve dialog open.
- **Steps:** Click Approve in the dialog.
- **Expected:** `approveEducator(userId)` called; on success `['educators']` invalidated → row leaves pending tab; modal closes.

### TC-ADM-030: Reject requires a reason
- **Type:** integration
- **Priority:** P0
- **Preconditions:** A `pending` row.
- **Steps:** Click "Reject" → reject Modal opens; leave reason empty.
- **Expected:** Reject button disabled while `!reason.trim()`; enabling requires non-empty text. On submit, `rejectEducator(id, reason)` fires, `['educators']` invalidated, modal closes and reason resets.

### TC-ADM-031: Reject shows pending label on button
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Reject mutation in flight.
- **Steps:** Submit reject.
- **Expected:** Button reads "Rejecting…" and is disabled while `reject.isPending`.

### TC-ADM-032: Suspend action on approved educators
- **Type:** integration
- **Priority:** P1
- **Preconditions:** On `approved` tab.
- **Steps:** Click "Suspend" on a row.
- **Expected:** `suspendEducator(userId)` fires immediately (no confirm dialog); `['educators']` invalidated; row leaves approved tab. Note: Approve/Reject actions only render for `pending`; Suspend only for `approved`.

### TC-ADM-033: Select-all checkbox toggles current page rows
- **Type:** unit
- **Priority:** P1
- **Preconditions:** Page has rows.
- **Steps:** Click header checkbox.
- **Expected:** All rows on the current page get selected/deselected; header checkbox checked only when every page row is selected (`allOnPageSelected`).

### TC-ADM-034: Individual row checkbox toggles selection
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Page has rows.
- **Steps:** Click a row's checkbox.
- **Expected:** Row toggles in the `selected` set; row background becomes `bg-primary-tint`; checkbox click does not navigate (`stopPropagation`).

### TC-ADM-035: Bulk action bar appears when rows selected
- **Type:** integration
- **Priority:** P1
- **Preconditions:** ≥1 row selected on current page.
- **Steps:** Select rows.
- **Expected:** Bulk bar shows `{n} selected` (mono), "Approve selected", "Reject selected", "Clear".

### TC-ADM-036: Bulk approve fires per-row and clears selection
- **Type:** e2e
- **Priority:** P0
- **Preconditions:** Multiple pending rows selected.
- **Steps:** Click "Approve selected".
- **Expected:** `approveEducator` called once per selected userId (instant, no confirm); selection cleared; list invalidated.

### TC-ADM-037: Bulk reject opens shared-reason modal
- **Type:** integration
- **Priority:** P0
- **Preconditions:** Multiple rows selected.
- **Steps:** Click "Reject selected".
- **Expected:** Modal "Reject {n} educators" opens; Reject button disabled until reason entered; irreversibility warning shown.

### TC-ADM-038: Bulk reject applies shared reason to all
- **Type:** e2e
- **Priority:** P0
- **Preconditions:** Bulk reject modal open with a reason.
- **Steps:** Enter reason, confirm.
- **Expected:** `rejectEducator(id, reason)` fires per selected row (falls back to "Bulk rejection" if reason trims empty); selection clears; modal closes; reason resets.

### TC-ADM-039: Clear selection button
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Rows selected.
- **Steps:** Click "Clear".
- **Expected:** `selected` set emptied; bulk bar hides.

### TC-ADM-040: CSV export of selected rows
- **Type:** integration
- **Priority:** P1
- **Preconditions:** ≥1 row selected.
- **Steps:** Click "Export CSV".
- **Expected:** CSV of selected rows downloads as `educators-{status}.csv` with header `Name,Phone,City,Status,Rating,Joined`; values quoted and quote-escaped.

### TC-ADM-041: CSV export of full filtered set when none selected
- **Type:** integration
- **Priority:** P2
- **Preconditions:** No rows selected; filter/search applied.
- **Steps:** Click "Export CSV".
- **Expected:** Exports the whole `filtered` set (respecting search + status), not just the visible page.

### TC-ADM-042: StatusPill colors per educator status
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Rows across statuses.
- **Steps:** Inspect Status column.
- **Expected:** pending → indigo "Under Review"; approved → green; rejected → red; suspended → slate.

### TC-ADM-043: Table sticky header & hover highlight
- **Type:** unit
- **Priority:** P2
- **Preconditions:** List longer than viewport.
- **Steps:** Scroll the table; hover a row.
- **Expected:** Header stays pinned (`sticky top-0`); hovered row gets `bg-primary-tint`.

---

## Educator Detail / KYC (`/educators/:id`)

### TC-ADM-044: Detail loading state
- **Type:** unit
- **Priority:** P2
- **Preconditions:** `getEducator(id)` in flight.
- **Steps:** Open `/educators/{id}`.
- **Expected:** "Loading…" shows.

### TC-ADM-045: Educator-not-found state
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Query resolves with no educator.
- **Steps:** Open detail for a bad id.
- **Expected:** Red "Educator not found" message.

### TC-ADM-046: Header shows identity + status
- **Type:** integration
- **Priority:** P1
- **Preconditions:** Educator resolved.
- **Steps:** View header.
- **Expected:** Avatar initial, name, mono `phone · email`, and StatusPill for `e.status`. Back button navigates `-1`.

### TC-ADM-047: Profile and Rates panels
- **Type:** integration
- **Priority:** P2
- **Preconditions:** Educator resolved.
- **Steps:** View Profile and Rates cards.
- **Expected:** Profile shows City, Rating (mono), Referral Code, Referrals, Pending Payout ₹. Rates show Teaching/Revision/Doubt Clearing ₹/hr. Missing values render `—`.

### TC-ADM-048: KYC document tiles render
- **Type:** integration
- **Priority:** P1
- **Preconditions:** Educator resolved.
- **Steps:** View Document Verification section.
- **Expected:** Three tiles — Government ID, Live Selfie, Education Certificate. Present URLs render an image linking to full size (opens new tab); absent URLs render italic "Not uploaded". If `educationProofVerifiedAt`, a green "✓ Verified {date}" shows.

### TC-ADM-049: Demo/intro clips with per-clip moderation
- **Type:** integration
- **Priority:** P1
- **Preconditions:** Educator has `demoClips`.
- **Steps:** View Videos section; click Approve / Reject on a clip.
- **Expected:** Each clip shows a `<video controls>`, type, date, and StatusPill. "Approve" hidden when already `approved`; "Reject" hidden when already `rejected`. Action calls `reviewClip(clipId, action)` and invalidates `['educator', id]`.

### TC-ADM-129: "Verify all documents" stamps and approves clips
- **Type:** e2e
- **Priority:** P1
- **Preconditions:** Educator with `educationProofVerifiedAt` null and ≥1 clip in `pending`.
- **Steps:** Click "Verify all documents" in the Document Verification header.
- **Expected:** `verifyDocuments(id)` fires (`PUT /admin/educators/:id/verify-documents`); button reads "Verifying…" and is disabled while pending; on success `['educator', id]` is invalidated and the section re-renders with the verified state. Server-side this stamps `educationProofVerifiedAt` **and flips all of that educator's clips to `approved`** in one transaction — so the per-clip Approve buttons in TC-ADM-049 disappear too.

### TC-ADM-130: Verified state replaces the button
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Educator with `educationProofVerifiedAt` set.
- **Steps:** Open the detail page.
- **Expected:** The header shows a green "✓ Verified {date}" (`toLocaleDateString('en-IN')`) **instead of** the button — verification is not re-runnable from the UI. The same stamp also renders under the Education Certificate tile.

### TC-ADM-131: Verifying documents does not approve the educator
- **Type:** integration
- **Priority:** P1
- **Preconditions:** A `pending` (or `under_review`) educator.
- **Steps:** Click "Verify all documents"; observe the status pill.
- **Expected:** The StatusPill is unchanged — document verification and account approval are separate actions, and the educator only becomes discoverable via the explicit Approve action.

### TC-ADM-050: Topics and bio sections
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Educator resolved.
- **Steps:** View Topics and Bio.
- **Expected:** Topics render as `Subject › Topic` chips; empty → "No topics assigned". Bio card renders only when `e.bio` present.

---

## Bookings / Sessions (`/sessions`)

### TC-ADM-051: Bookings list loads with default (all) filter
- **Type:** integration
- **Priority:** P1
- **Preconditions:** Authenticated.
- **Steps:** Open `/sessions`.
- **Expected:** `getBookings({ status: undefined })` fetched; "All" filter active; table renders.

### TC-ADM-052: Status filter chips
- **Type:** integration
- **Priority:** P1
- **Preconditions:** On `/sessions`.
- **Steps:** Click confirmed / completed / pending_payment / failed / disputed.
- **Expected:** Query key `['bookings', status]` changes; refetch with that status; active chip indigo. Labels replace `_` with space.

### TC-ADM-053: Empty bookings state
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Filter returns no rows.
- **Steps:** Select a status with no bookings.
- **Expected:** EmptyState "No bookings match these filters", hint "Try selecting a different status.", icon `▤`.

### TC-ADM-054: Booking row formatting (mono ID + amount)
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Bookings present.
- **Steps:** Inspect a row.
- **Expected:** Booking ID = first 8 chars, mono; Duration `{n}m` mono right-aligned; Amount `₹{amountDue}` mono right-aligned bold; StatusPill in Status column; Scheduled date mono (or `—`). Topic/type default `—`.

### TC-ADM-055: Pending-payment alert banner
- **Type:** integration
- **Priority:** P0
- **Preconditions:** `getPendingPayments` returns ≥1 booking.
- **Steps:** Open `/sessions`.
- **Expected:** Amber banner "{n} booking(s) stuck in pending payment for 15+ minutes", each listing `student → educator · ₹amount` with a "Resolve" button. Banner hidden when none pending.

### TC-ADM-056: Resolve modal — Mark Confirmed
- **Type:** e2e
- **Priority:** P0
- **Preconditions:** A pending-payment booking.
- **Steps:** Click "Resolve" → optional note → "Mark Confirmed".
- **Expected:** `resolvePayment(id, 'confirmed', note)` fires; on success `['bookings']` and `['pending-payments']` invalidated; modal closes; note resets.

### TC-ADM-057: Resolve modal — Mark Failed
- **Type:** e2e
- **Priority:** P0
- **Preconditions:** A pending-payment booking.
- **Steps:** Click "Resolve" → "Mark Failed".
- **Expected:** `resolvePayment(id, 'failed', note)` fires; both queries invalidated; modal closes.

### TC-ADM-058: Resolve note is optional
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Resolve modal open.
- **Steps:** Leave the note field empty and confirm.
- **Expected:** Both Mark Confirmed / Mark Failed remain enabled; empty note sent.

### TC-ADM-059: Table sticky header + hover row on bookings
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Long booking list.
- **Steps:** Scroll; hover a row.
- **Expected:** Header pinned; row hover highlight `bg-primary-tint`.

---

## Reviews — Moderation (`/reviews`)

### TC-ADM-060: Default tab Pending with count badge
- **Type:** integration
- **Priority:** P1
- **Preconditions:** Pending reviews exist.
- **Steps:** Open `/reviews`.
- **Expected:** "Pending" tab active (indigo underline); a red mono count badge shows the pending count next to the "Reviews" title (only on pending tab when count > 0).

### TC-ADM-061: Tabs switch and refetch
- **Type:** integration
- **Priority:** P1
- **Preconditions:** On `/reviews`.
- **Steps:** Click Approved / Rejected.
- **Expected:** Query key `['reviews', tab]` changes; `getReviews(tab)` refetches; the count badge is only on pending.

### TC-ADM-062: Loading state
- **Type:** unit
- **Priority:** P2
- **Preconditions:** `getReviews` in flight.
- **Steps:** Switch tabs.
- **Expected:** Centered "Loading…" while `isLoading`.

### TC-ADM-063: Empty state per tab
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Tab has no reviews.
- **Steps:** View tab.
- **Expected:** EmptyState "No {tab} reviews", icon `★`; pending hint mentions new reviews appearing for approval, other tabs show `No {tab} reviews to show.`

### TC-ADM-064: Review card content
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Reviews present.
- **Steps:** Inspect a card.
- **Expected:** Star row (`★` filled to rating, rest muted, `aria-label="{r} out of 5"`), StatusPill, mono `#{id.slice(0,8)}`, optional topic name, `student → educator`, comment (or italic "No written comment"), mono date.

### TC-ADM-065: Approve/Reject buttons only on pending
- **Type:** integration
- **Priority:** P1
- **Preconditions:** Cards across statuses.
- **Steps:** View pending vs approved/rejected cards.
- **Expected:** Approve/Reject buttons render only when `r.status === 'pending'`; absent on approved/rejected.

### TC-ADM-066: Approve makes review public (invalidate + move)
- **Type:** e2e
- **Priority:** P0
- **Preconditions:** A pending review.
- **Steps:** Click Approve.
- **Expected:** `reviewReview(id, 'approve')` fires; `['reviews']` invalidated → card leaves Pending and appears under Approved. Cross-ref backend: approved reviews become publicly visible.

### TC-ADM-067: Reject hides review (invalidate + move)
- **Type:** e2e
- **Priority:** P0
- **Preconditions:** A pending review.
- **Steps:** Click Reject.
- **Expected:** `reviewReview(id, 'reject')` fires; card moves to Rejected; cross-ref backend: rejected reviews stay hidden from public.

### TC-ADM-068: Buttons disabled while mutating
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Approve/Reject in flight.
- **Steps:** Click a moderation button.
- **Expected:** Both buttons disabled while `moderate.isPending`.

---

## Payouts (`/payouts`)

Three tabs, fed by two queries. `getPendingPayouts` drives **Pending** (educator balances);
`getPayoutHistory` returns every `PayoutHistory` row and is split client-side into
**Queued** (`status === 'pending'`) and **History** (`status === 'paid'`).

The distinction is the point of the page: Mole cannot transfer money. Queued rows are a
worklist the `auto-payout` cron wrote; the admin makes the bank/UPI transfer by hand and
then settles the row, which is the only thing that marks it paid and reduces the balance.

### TC-ADM-069: Pending tab default with total banner
- **Type:** integration
- **Priority:** P1
- **Preconditions:** Educators with pending payout exist.
- **Steps:** Open `/payouts`.
- **Expected:** Pending tab active; header subtitle "{n} educators awaiting payout"; if `totalPending > 0`, amber banner "Total pending: ₹{sum}" (mono, en-IN); tab label reads "Pending ({n})".

### TC-ADM-070: Pending table columns & masked bank
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Pending payouts present.
- **Steps:** Inspect rows.
- **Expected:** Columns Educator, Phone (mono), Pending Amount (mono green right), Bank/UPI (masked `••••{last4}` or `—`), Action "Process payout".

### TC-ADM-071: Empty pending payouts
- **Type:** unit
- **Priority:** P2
- **Preconditions:** No pending payouts.
- **Steps:** View pending tab.
- **Expected:** EmptyState "No pending payouts", hint "All educators are settled up.", icon `₹`.

### TC-ADM-072: History tab shows settled payouts only
- **Type:** integration
- **Priority:** P2
- **Preconditions:** A mix of queued (`status:'pending'`) and settled (`status:'paid'`) rows.
- **Steps:** Click "History".
- **Expected:** Columns Educator, Amount (mono), Type (underscores → spaces), Reference (mono), Date (mono). **Only `paid` rows appear** — a queued row has no reference yet and must not sit in History looking like a completed transfer. If empty → EmptyState "No payouts yet", hint "Processed payouts will appear here.".

### TC-ADM-115: Three tabs with live counts
- **Type:** integration
- **Priority:** P1
- **Preconditions:** Pending balances, queued rows and settled rows all exist.
- **Steps:** Open `/payouts`.
- **Expected:** Three buttons in order — "Pending ({n})", "Queued ({n})", "History" (no count). `pending` is the default tab; the active one is `bg-primary text-white`. Counts come from the split described above, so a settled row leaves Queued and appears in History without a refetch of a different endpoint.

### TC-ADM-116: Queued tab lists unsettled rows with the no-transfer warning
- **Type:** integration
- **Priority:** P0
- **Preconditions:** ≥1 `PayoutHistory` row with `status:'pending'`.
- **Steps:** Click "Queued".
- **Expected:** An amber banner states these were queued by the automated run and that **Mole does not transfer money** — the admin must make the bank/UPI transfer and then settle with its reference. Table columns Educator, Amount (mono, amber — not the green used for completed money), Queued date (mono, from `createdAt`), Note (the run note, or `—`), and a "Settle" action per row.

### TC-ADM-117: Empty queued tab
- **Type:** unit
- **Priority:** P2
- **Preconditions:** No rows with `status:'pending'`.
- **Steps:** View the Queued tab.
- **Expected:** EmptyState "Nothing queued", hint "Automated payout runs will queue payouts here for settlement.", icon `₹`.

### TC-ADM-118: Settle modal states what is being confirmed
- **Type:** integration
- **Priority:** P0
- **Preconditions:** A queued row.
- **Steps:** Click "Settle".
- **Expected:** Modal titled "Settle payout — {educator name}"; body confirms the admin **has already transferred** ₹{amount} to that educator and warns it marks the payout paid, reduces their pending balance, and cannot be undone. UTR/UPI Reference input + optional Note; single "Confirm & mark paid" button (unlike the direct-payout flow, there is no second review step).

### TC-ADM-119: Settle reference validation
- **Type:** unit
- **Priority:** P0
- **Preconditions:** Settle modal open.
- **Steps:** Leave Reference empty; then type fewer than 4 characters.
- **Expected:** "Confirm & mark paid" disabled while `reference.trim().length < 4`; with a short non-empty value the border turns red and "Reference must be at least 4 characters." shows. A settled payout with no traceable reference must be impossible to create from the UI.

### TC-ADM-120: Confirming a settle refreshes both tabs
- **Type:** e2e
- **Priority:** P0
- **Preconditions:** Settle modal with a valid reference.
- **Steps:** Click "Confirm & mark paid".
- **Expected:** `settlePayout(id, reference, note)` fires; on success `['payouts-pending']` **and** `['payouts-history']` are invalidated, so the row leaves Queued, appears in History, and the educator's Pending balance drops in the same refresh. Modal closes; the shared reference/note form resets. Button shows "Settling…" and is disabled while pending.

### TC-ADM-121: Failed settle keeps the modal open with an error
- **Type:** integration
- **Priority:** P0
- **Preconditions:** Settle modal; the backend returns 409 `ALREADY_SETTLED` (another admin settled it first) or 422 `AMOUNT_EXCEEDS_PENDING`.
- **Steps:** Confirm the settle.
- **Expected:** "Could not settle this payout. Refresh and try again." shows inside the modal; nothing is invalidated and the modal stays open. The UI must not optimistically show the payout as paid — the backend is the authority on whether the balance moved.

### TC-ADM-122: Settle and direct-payout modals share one form
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Open the Settle modal, type a reference, close it, then open "Process payout" from the Pending tab.
- **Steps:** Inspect the reference/note fields.
- **Expected:** `closePay()` clears `payModal`, `settleModal`, the confirm flag, reference and note, so no reference leaks from one payout into another. Only one of the two modals can be open at a time.

### TC-ADM-073: Process flow step 1 — reference entry
- **Type:** integration
- **Priority:** P0
- **Preconditions:** A pending payout row.
- **Steps:** Click "Process payout".
- **Expected:** Modal "Pay {name}" opens showing amount; UTR/UPI Reference input + optional Note; "Review payout" button.

### TC-ADM-074: Reference min-length validation blocks step 1
- **Type:** unit
- **Priority:** P0
- **Preconditions:** Pay modal step 1.
- **Steps:** Type <4 chars in Reference.
- **Expected:** Input border turns red; inline error "Reference must be at least 4 characters."; "Review payout" disabled until `reference.trim().length >= 4`.

### TC-ADM-075: Process flow step 2 — confirm summary
- **Type:** integration
- **Priority:** P0
- **Preconditions:** Valid reference entered.
- **Steps:** Click "Review payout".
- **Expected:** Modal switches to "Confirm payout" showing amount (mono), educator name, and reference (mono) with "This is irreversible." Buttons: "Back" (returns to step 1) and "Confirm & mark paid".

### TC-ADM-076: Confirm & mark paid processes payout
- **Type:** e2e
- **Priority:** P0
- **Preconditions:** On confirm step.
- **Steps:** Click "Confirm & mark paid".
- **Expected:** `triggerPayout(userId, pendingPayout, reference, note)` fires; on success `['payouts-pending']` and `['payouts-history']` invalidated; modal closes; form resets. Button shows "Processing…" and disables while `pay.isPending`.

### TC-ADM-077: Close/cancel resets payout form
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Pay modal open (either step).
- **Steps:** Close the modal.
- **Expected:** `closePay` clears modal, confirm flag, reference, and note.

---

## Disputes (`/disputes`)

### TC-ADM-078: Disputes list loads with count badge
- **Type:** integration
- **Priority:** P1
- **Preconditions:** Disputed bookings exist.
- **Steps:** Open `/disputes`.
- **Expected:** `getDisputes()` fetched; red mono count badge next to title when count > 0; each card shows StatusPill "disputed" (red), mono `#{id.slice(0,8)}`, session type, `student → educator`, first note as reason, booking `₹amount · {mins} min · date`.

### TC-ADM-079: Loading state
- **Type:** unit
- **Priority:** P2
- **Preconditions:** `getDisputes` in flight.
- **Steps:** Open page.
- **Expected:** Centered "Loading…".

### TC-ADM-080: Empty disputes state
- **Type:** unit
- **Priority:** P2
- **Preconditions:** No disputes.
- **Steps:** View page.
- **Expected:** EmptyState "No open disputes", hint "Disputed bookings will appear here for review.", icon `⚑`.

### TC-ADM-081: Urgent highlight for disputes open >24h
- **Type:** unit
- **Priority:** P1
- **Preconditions:** A dispute with `createdAt` >24h ago.
- **Steps:** View card.
- **Expected:** Card border becomes `border-warning`; an amber "{hours}h open" mono tag shows. Disputes ≤24h have default border and no tag.

### TC-ADM-082: Note thread renders additional notes
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Dispute has >1 `disputeNotes`.
- **Steps:** View card.
- **Expected:** First note is the reason; subsequent notes render as an indented thread with `{author}: {body}`.

### TC-ADM-083: Resolve requires resolution note
- **Type:** integration
- **Priority:** P0
- **Preconditions:** A dispute.
- **Steps:** Click "Resolve"; leave note empty.
- **Expected:** Modal "Resolve Dispute" with student vs educator + `₹amount`; "Resolve Dispute" button disabled while `!note.trim()`.

### TC-ADM-084: Resolve without refund
- **Type:** e2e
- **Priority:** P0
- **Preconditions:** Resolve modal, note filled, refund unchecked.
- **Steps:** Click Resolve Dispute.
- **Expected:** `resolveDispute(id, { resolutionNote, refund:false })` fires (no `refundAmount`/`refundType`); `['disputes']` invalidated; modal closes; form resets.

### TC-ADM-085: Credit refund amount field appears when checked
- **Type:** unit
- **Priority:** P1
- **Preconditions:** Resolve modal open.
- **Steps:** Check "Issue credit refund to student".
- **Expected:** Numeric amount input (placeholder `Amount ₹ (max {amountDue})`) and a "Credits" type select appear.

### TC-ADM-086: Refund amount validation ≤ amountDue and ≥1
- **Type:** unit
- **Priority:** P0
- **Preconditions:** Refund checked, amount entered.
- **Steps:** Enter amount `0`, then `> amountDue`, then a valid amount.
- **Expected:** Invalid (`<1` or `>amountDue` or non-finite) → red border + error "Enter an amount between ₹1 and ₹{amountDue}." and Resolve disabled. Valid amount clears the error and enables Resolve.

### TC-ADM-087: Resolve with valid refund
- **Type:** e2e
- **Priority:** P0
- **Preconditions:** Note + valid refund amount.
- **Steps:** Submit.
- **Expected:** `resolveDispute(id, { resolutionNote, refund:true, refundAmount:Number(amount), refundType:'credits' })` fires; disputes invalidated; card leaves the queue.

### TC-ADM-088: Dismiss opens confirm dialog
- **Type:** integration
- **Priority:** P0
- **Preconditions:** A dispute.
- **Steps:** Click "Dismiss".
- **Expected:** Confirm Modal "Dismiss dispute" naming both parties, warning "No refund will be issued and this cannot be undone."; Cancel and "Dismiss dispute" buttons.

### TC-ADM-089: Confirm dismiss mutates
- **Type:** e2e
- **Priority:** P0
- **Preconditions:** Dismiss dialog open.
- **Steps:** Click "Dismiss dispute".
- **Expected:** `dismissDispute(id)` fires; `['disputes']` invalidated; modal closes; button shows "Dismissing…" and disables while pending.

---

## Topics & Subjects (`/topics`)

### TC-ADM-090: Topics catalog loads grouped by subject
- **Type:** integration
- **Priority:** P1
- **Preconditions:** Subjects/topics exist.
- **Steps:** Open `/topics`.
- **Expected:** `getTopics` fetched; each subject card lists its topic chips; loading shows "Loading…"; subjects with none show "No topics yet".

### TC-ADM-091: Add Subject flow
- **Type:** e2e
- **Priority:** P1
- **Preconditions:** On `/topics`.
- **Steps:** Click "+ Add Subject" → enter name → Add.
- **Expected:** Add button disabled while `!name.trim()`; `createSubject(name)` fires; `['topics']` invalidated; modal closes; name resets.

### TC-ADM-092: Add Topic to a subject
- **Type:** e2e
- **Priority:** P1
- **Preconditions:** A subject exists.
- **Steps:** Click "+ Add Topic" on a subject → enter name → Add.
- **Expected:** Modal titled "Add Topic to {subject}"; Add disabled until name non-empty; `createTopic(subjectId, name)` fires; catalog invalidated.

### TC-ADM-093: Delete (deactivate) a topic
- **Type:** e2e
- **Priority:** P1
- **Preconditions:** An active topic chip.
- **Steps:** Click the `✕` on the chip.
- **Expected:** `deleteTopic(topicId)` fires immediately; `['topics']` invalidated. Inactive topics render with `line-through` red styling and no `✕`.

---

## Admins & Audit Log (`/admins`)

### TC-ADM-094: Admins tab lists team with status
- **Type:** integration
- **Priority:** P1
- **Preconditions:** Admins exist.
- **Steps:** Open `/admins`.
- **Expected:** `listAdmins` fetched; table columns Name, **Email** (mono), Created By (or "System"), Joined (mono), Status pill (active green / inactive slate), Action.

### TC-ADM-095: Create admin takes name + email only
- **Type:** e2e
- **Priority:** P0
- **Preconditions:** On `/admins`.
- **Steps:** Click "+ Add Admin" → fill name and email → Create.
- **Expected:** The modal has exactly **two** inputs, name (`text`) and email (`type=email`) — **no password input**, and a note that the new admin signs in with this email via a one-time code or magic link. "Create Admin" is disabled until both are filled; `createAdmin({ name, email })` fires; `['admins']` invalidated; modal closes; form resets; button shows "Creating…" while pending.

### TC-ADM-123: Duplicate email surfaces a specific error
- **Type:** integration
- **Priority:** P1
- **Preconditions:** An admin already exists with the target email.
- **Steps:** Create an admin with that email.
- **Expected:** The backend's 409 `EMAIL_TAKEN` maps to "An admin with that email already exists."; any other failure falls back to "Could not create admin. Check the email and try again."; the modal stays open with the values intact.

### TC-ADM-096: Deactivate an admin
- **Type:** e2e
- **Priority:** P1
- **Preconditions:** An active admin (not self).
- **Steps:** Click "Deactivate".
- **Expected:** `deactivateAdmin(id)` fires; `['admins']` invalidated; status flips to inactive; Deactivate button hidden for inactive admins.

### TC-ADM-097: Cannot deactivate self (backend-enforced)
- **Type:** integration
- **Priority:** P0
- **Preconditions:** Logged-in admin's own row is active.
- **Steps:** Click "Deactivate" on your own row.
- **Expected:** Backend rejects self-deactivation; the current admin remains active (UI re-fetches unchanged). Cross-ref backend admin API for the self-deactivation guard.

### TC-ADM-098: Audit Log tab loads on demand
- **Type:** integration
- **Priority:** P2
- **Preconditions:** On `/admins`.
- **Steps:** Click "Audit Log".
- **Expected:** `getAuditLog` query is `enabled` only when the audit tab is active; log table shows Admin, Action (mono chip), Target (`{type}/{id.slice(0,8)}…`), Note (or `—`), Time (mono). Empty → EmptyState "No audit logs yet", icon `▤`.

---

## System Config (`/config`)

### TC-ADM-099: Config loads grouped settings
- **Type:** integration
- **Priority:** P1
- **Preconditions:** Config values exist.
- **Steps:** Open `/config`.
- **Expected:** Loading shows "Loading…"; settings render under groups "Referral & Credits", "Commission & Payouts", "App Control", each with label + description + editor.

### TC-ADM-100: Boolean config uses select
- **Type:** unit
- **Priority:** P2
- **Preconditions:** A boolean key (e.g. `auto_payout_enabled`).
- **Steps:** View the control.
- **Expected:** Renders a `true`/`false` select (not a text input).

### TC-ADM-101: Save button appears only when dirty
- **Type:** unit
- **Priority:** P1
- **Preconditions:** A config field.
- **Steps:** Edit a value, then revert to original.
- **Expected:** "Save" button appears only while `draft !== current`; disappears when reverted.

### TC-ADM-102: Numeric validation blocks save
- **Type:** unit
- **Priority:** P0
- **Preconditions:** A `number`-typed field.
- **Steps:** Type a non-numeric value (e.g. "abc").
- **Expected:** Input border turns red (`numInvalid` = non-finite Number); Save button disabled while invalid.

### TC-ADM-103: Save persists and confirms
- **Type:** e2e
- **Priority:** P1
- **Preconditions:** A dirty, valid field.
- **Steps:** Click Save.
- **Expected:** `updateConfig(key, value)` fires with type coercion (Number for numeric, boolean for booleans, string otherwise); `['config']` invalidated; "Saved ✓" shows for ~2s then clears.

---

## Navigation, Layout & Cross-cutting

### TC-ADM-104: Sidebar lists all sections
- **Type:** integration
- **Priority:** P1
- **Preconditions:** Authenticated.
- **Steps:** View the sidebar.
- **Expected:** Links in order: Overview `/`, Insights `/insights`, Educators `/educators`, Students `/students`, Bookings `/sessions`, Disputes `/disputes`, Reviews `/reviews`, Payouts `/payouts`, Topics & Subjects `/topics`, Admins `/admins`, System Config `/config` — each with an inline SVG icon, and "Sign out" pinned at the bottom.

### TC-ADM-105: Active nav uses indigo highlight
- **Type:** unit
- **Priority:** P2
- **Preconditions:** On any section.
- **Steps:** Observe the current nav item.
- **Expected:** Active `NavLink` gets `bg-primary text-white`; others muted with `hover:bg-primary-tint`. Overview uses `end` so it's active only on exact `/`.

### TC-ADM-106: Top header hosts the notification bell
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Authenticated.
- **Steps:** View the top bar.
- **Expected:** A slim `h-14` header bordered at the bottom, right-aligned, containing only the `NotificationBell` (the old status lights / env pill / global search mock are gone).

### TC-ADM-124: Notification bell badge reflects live actionable counts
- **Type:** integration
- **Priority:** P1
- **Preconditions:** Pending educators, under-review educators, reviews to moderate, open disputes and payouts due all exist.
- **Steps:** Load any admin page.
- **Expected:** `getAdminNotifications` runs on mount and re-polls every 60s (`refetchInterval: 60000`); a red badge shows `total` — the **sum of the item counts**, not the number of items — and renders "99+" above 99. With `total === 0` no badge renders at all.

### TC-ADM-125: Bell dropdown lists actionable buckets
- **Type:** integration
- **Priority:** P1
- **Preconditions:** ≥1 actionable item.
- **Steps:** Click the bell.
- **Expected:** Dropdown headed "Notifications" listing one row per item: a count chip, the server-supplied title (e.g. "3 educators awaiting approval", pluralised server-side), and a chevron. Buckets with a zero count are absent because the API omits them.

### TC-ADM-126: Empty notification state
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Nothing actionable.
- **Steps:** Open the bell.
- **Expected:** Dropdown shows "You're all caught up"; no rows.

### TC-ADM-127: Clicking a notification navigates to the right page and tab
- **Type:** e2e
- **Priority:** P1
- **Preconditions:** Both `pending` and `under_review` educators exist.
- **Steps:** Open the bell and click the "awaiting approval" item; reopen and click the "under review" item.
- **Expected:** The dropdown closes and the app navigates to `/educators?status=pending` and `/educators?status=under_review` respectively — and the Educators page **lands on that status tab**, because it initialises and syncs its tab from the `?status=` search param rather than always defaulting to `pending`. Other items route to `/reviews`, `/disputes`, `/payouts`.

### TC-ADM-128: Bell closes on outside click
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Dropdown open.
- **Steps:** Click anywhere outside the bell container.
- **Expected:** A `mousedown` listener bound only while open closes the dropdown; clicks inside it do not. The listener is removed on close/unmount.

### TC-ADM-107: All data tables share hover + sticky behavior
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Educators, Bookings, Payouts, Admins tables populated beyond viewport height.
- **Steps:** Scroll each; hover rows.
- **Expected:** Every `TableCard` scrolls within `max-h-[calc(100vh-16rem)]`, `TableHead` stays sticky, rows highlight `bg-primary-tint` on hover.

### TC-ADM-108: IDs and amounts use mono font app-wide
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Any list with IDs/amounts.
- **Steps:** Inspect booking IDs, review IDs, phones, ₹ amounts, dates.
- **Expected:** All render with `font-mono` (and `tabular-nums` for aligned numbers).

### TC-ADM-109: Modal close/backdrop dismisses without mutating
- **Type:** unit
- **Priority:** P2
- **Preconditions:** Any confirm/entry modal open (approve, reject, resolve, payout, dismiss, add admin/subject/topic).
- **Steps:** Click Cancel / close.
- **Expected:** Modal closes; no mutation fires; associated draft state (reason/note/form) resets where the handler resets it.

### TC-ADM-110: Confirm dialogs gate irreversible actions
- **Type:** integration
- **Priority:** P0
- **Preconditions:** Authenticated.
- **Steps:** Trigger educator approve, bulk reject, payout processing, dispute dismiss.
- **Expected:** Each requires an explicit confirmation step/dialog (or two-step for payouts) before the mutation fires; single-step instant actions (bulk approve, suspend, topic delete, review approve/reject, clip approve/reject) fire immediately by design.
