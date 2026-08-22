# Product Requirements — Mole

Mole is a peer-to-peer tutoring marketplace connecting students with educators for live 1:1 video sessions.

---

## User Roles

### Student
- Discovers educators by subject, topic, grade, exam board
- Books and pays for live sessions
- Joins sessions via Zoom Video SDK
- Reviews educators after sessions
- Manages wallet credits (referral bonuses, promotions)
- Raises disputes on unsatisfactory sessions
- Views session recordings for 24 hours post-session

### Educator
- Registers and goes through identity verification
- Sets topics, rates, availability, bio, demo clips
- Receives booking requests via WhatsApp (accept/decline)
- Conducts live sessions
- Earns per-minute rate; payouts triggered by admin
- Tracks earnings and referral bonuses

### Admin
- Approves/rejects/suspends educator accounts
- Reviews educator demo clips
- Manages topic and subject catalog
- Resolves student disputes
- Triggers educator payouts
- Updates platform config
- Views analytics dashboard

---

## Functional Requirements

### Authentication
- [x] Phone number + OTP login (no password)
- [x] JWT access token (15min) + refresh token (30d)
- [x] Role selection on first login (student/educator)
- [x] Terms acceptance required before access
- [x] FCM token registration for push notifications
- [x] Rate limiting on OTP send endpoint

### Educator Onboarding
- [x] Profile: name, city, bio, selfie, LinkedIn, qualifications
- [x] Topic selection + approval by admin
- [x] Demo clip upload (video); admin approves before shown to students
- [x] Intro clip upload
- [x] Rate setting (per minute for teaching, revision, doubt clearing)
- [x] Weekly availability slots
- [x] Government ID upload
- [x] Education proof upload + verification
- [x] Bank details (account number, IFSC)
- [x] Status flow: pending → approved → (suspended/rejected)

### Student Onboarding
- [x] Grade level selection
- [x] Exam board selection (CBSE, ICSE, etc.)
- [x] Onboarding tutorial (one-time)

### Educator Discovery
- [x] Live educators (currently online) list
- [x] Search by keyword (name, topic, subject)
- [x] Silent filter: search results filtered by student's exam boards
- [x] Suggested educators: board-matched + rating ≥ 4, fallback to top-rated
- [x] Only approved educators shown
- [x] Only approved demo/intro clips shown in educator profile
- [x] Educator public profile: avatar, bio, demo clips, collapsible topics, rates, reviews

### Booking
- [x] Student selects educator, topic, session type, duration
- [x] Booking request sent → `awaiting_educator` status
- [x] WhatsApp notification to educator with session details + accept/decline links
- [x] Educator can accept via WhatsApp link tap or YES/NO text reply
- [x] On acceptance: booking → `pending_payment`, student notified via FCM
- [x] On decline/no response: booking → `cancelled`, student notified
- [x] Cancel booking (student or educator) — cancellable at `awaiting_educator`, `pending_payment`, `confirmed` (before session)
- [x] Late cancellation penalty: 10% if cancelled within configured window

### Payments
- [x] UPI-based payment (no card, no Razorpay)
- [x] UPI deep link opens student's UPI app directly
- [x] HMAC-signed webhook from UPI gateway confirms payment
- [x] Booking → `confirmed` on payment success
- [x] Wallet credits applied at booking creation (deducted from amount_due)

### Live Session
- [x] Zoom Video SDK for video call
- [x] Session token (JWT) generated per user per session
- [x] Session joinable 5 minutes before scheduled time
- [x] Countdown timer in-session (colored green → orange → red at 10min/5min)
- [x] **Extend call**: button shown at 10 minutes before end
  - Student selects extension duration (30/60/90 min)
  - UPI payment for extension cost
  - `durationMinutes` updated on confirm
- [x] Session end tracked (educator/student join timestamps)
- [x] OTP-based session end verification
- [x] Auto-dispute cron: sessions where neither party joined

### Session Recording
- [x] Zoom cloud recording (where available)
- [x] Zoom webhook → download → upload to S3
- [x] Student access window: 24 hours post-session
- [x] Presigned S3 URL served via API

### Reviews
- [x] Student reviews session after completion (1–5 rating + comment)
- [x] FCM prompt sent after session ends
- [x] Review prompt shown once per booking
- [x] Admin can flag reviews

### Wallet
- [x] Wallet balance per student
- [x] Wallet credits: referral bonuses, promotions
- [x] Credits have expiry date
- [x] Cron job expires unused credits

### Referrals
- [x] Educator referral codes
- [x] Students can enter educator referral code at signup
- [x] Educator earns referral bonus per referred student (tracked via `Referral`)
- [x] Referral payout type in `PayoutHistory`

### Disputes
- [x] Student raises dispute on completed/cancelled session
- [x] Dispute note thread (student + admin messages)
- [x] Booking → `disputed` status
- [x] Admin resolves dispute via admin panel
- [x] Past sessions (cancelled + disputed) visible in student app for reference

### Notifications
- [x] FCM push for: booking accepted, booking confirmed, session reminder (1hr, 5min), session ended, review prompt, dispute updates
- [x] In-app notification feed with deep links
- [x] Mark-as-read on individual notifications

### Educator Earnings
- [x] Per-session rate: `(durationMinutes × rate_per_minute)`
- [x] Educator payout: `totalPrice × (1 - platformFeePercent)`
- [x] `pendingPayout` accumulates until admin triggers payout
- [x] Payout history tracked in `PayoutHistory`

### Admin
- [x] Educator approval / rejection / suspension
- [x] Demo clip approval / rejection
- [x] Topic and subject management (CRUD)
- [x] Dispute resolution with notes
- [x] Manual payout trigger
- [x] Platform config updates (JSON key-value store)
- [x] Admin user management (create, deactivate)
- [x] Audit log on all admin actions
- [x] Analytics: active educators, revenue, session counts, live now

---

## Non-Functional Requirements

| Requirement | Approach |
|------------|---------|
| Security | JWT auth, Helmet headers, rate limiting, HMAC webhook verification |
| Availability | Stateless API, horizontally scalable |
| Storage | AWS S3 for all media; CDN-ready via `CDN_BASE_URL` |
| Observability | Pino structured logging, Morgan HTTP logs, Sentry error tracking |
| Dev experience | Mock SMS provider (no Twilio needed locally), local file storage fallback |
| DB safety | Prisma migrations, no raw SQL |
| API validation | Zod schemas on all request bodies |

---

## Out of Scope (MVP)

- Automated bank payouts (currently manual admin trigger)
- Group sessions
- Scheduled recurring sessions
- In-app messaging between student and educator
- Educator calendar sync (external calendars)
- Multi-currency / international payments
