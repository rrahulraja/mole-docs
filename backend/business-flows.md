# Business Flows — Mole

---

## 1. Educator Registration & Onboarding

```
Educator downloads app
        │
        ▼
Phone + OTP login → role = educator
        │
        ▼
Onboarding flow (mobile):
  1. Basic info (name, city, bio, selfie)
  2. Topic selection (subject → topics)
  3. Demo clip upload (video)
  4. Rate setting (teaching / revision / doubt_clearing per minute)
  5. Availability slots (weekly schedule)
  6. Identity check (govt ID, education proof)
  7. Bank details (account number, IFSC)
        │
        ▼
Educator.status = pending
        │
        ▼ (admin reviews in mole-admin)
Admin approves:
  - Educator.status = approved
  - Educator shown in search / live now
Admin rejects:
  - Educator.status = rejected
  - rejection_reason stored
  - educator notified via FCM
Admin suspends (post-approval):
  - Educator.status = suspended
  - Removed from search immediately
```

---

## 2. Student Onboarding

```
Student downloads app
        │
        ▼
Phone + OTP login → role = student
        │
        ▼
Onboarding flow (mobile):
  1. Grade level selection
  2. Exam board selection (CBSE / ICSE / etc.)
  3. Onboarding tutorial (one-time)
        │
        ▼
Student profile created
  - gradeLevel saved
  - examBoards saved
  - used to silently filter educator search results
```

---

## 3. Educator Discovery

```
Student opens app
        │
        ├─── Home screen
        │      ├── Live Now: educators with isLiveNow=true, board-filtered
        │      └── Suggested for You: rating ≥ 4 + board match, top 8
        │              fallback → top-rated if no board match
        │
        └─── Search
               ├── keyword → name/topic/subject match
               ├── silently filtered by student.examBoards (hasSome)
               └── only approved educators returned

Educator Profile:
  - Hero: avatar, name, city, rating
  - Bio
  - Demo Clips (only status=approved clips)
  - Topics (collapsible, 3 shown by default)
  - Rates (per session type)
  - Reviews
  - Book button → booking sheet
```

---

## 4. Booking Flow

```
Student selects:
  - Topic
  - Session type (teaching / revision / doubt_clearing)
  - Duration (30 / 60 / 90 min)
  - Scheduled time
        │
        ▼
Price calculated:
  totalPrice = durationMinutes × rate_per_minute
  creditsApplied = min(walletBalance, totalPrice)
  amountDue = totalPrice - creditsApplied
        │
        ▼
Student taps "Send Booking Request"
  POST /bookings
  Booking created: status = awaiting_educator
        │
        ▼
Backend sends WhatsApp to educator:
  "New booking request!
   Topic: [name]
   Type: [type]
   Duration: [N] min
   Date: [date]
   Amount: ₹[amount]
   ✅ Accept: https://api.mole.app/bookings/:id/respond?t=JWT
   ❌ Decline: https://api.mole.app/bookings/:id/respond?t=JWT"
        │
Mobile: show "Awaiting educator confirmation" screen
        │
        ├── Educator taps Accept link (GET /respond?t=JWT)
        │     OR educator replies "YES" → WhatsApp webhook
        │           │
        │           ▼
        │     JWT verified
        │     Booking → pending_payment
        │     Payment record created
        │     FCM push to student: "Educator accepted! Pay now."
        │           │
        │           ▼
        │     Student taps notification → Payment screen
        │     POST /payment/:id/intent → UPI URI
        │     Student pays in UPI app
        │           │
        │           ▼ (UPI webhook, HMAC verified)
        │     Payment → success
        │     Booking → confirmed
        │     FCM to both: "Booking confirmed!"
        │
        └── Educator taps Decline link (or replies "NO")
              Booking → cancelled
              FCM to student: "Educator declined your request."
```

---

## 5. Payment Flow

```
UPI Payment Intent:
  POST /payment/:id/intent
    → generates: upi://pay?pa=MERCHANT_VPA&pn=MERCHANT_NAME&am=AMOUNT&tn=BOOKING_ID
    → client calls Linking.openURL(upiUri)
    → student's UPI app opens, pre-filled

UPI Webhook (payment confirmation):
  POST /webhooks/upi
    → HMAC-SHA256 header verified against UPI_WEBHOOK_SECRET
    → match booking by reference
    → Payment.status = success
    → Booking.status = confirmed
    → FCM push to student + educator

Polling (client-side):
  GET /payment/:id/status every 3 seconds
    → returns booking status
    → on confirmed → navigate to booking-confirmed screen
    → 5min timeout → navigate to booking-failed screen

Extension Payment:
  Student selects 30/60/90 min extension during session
    → GET /bookings/:id/extend?minutes=N
    → backend returns UPI URI (no new Payment record)
    → student pays
    → POST /bookings/:id/extend/confirm?minutes=N
    → Booking.durationMinutes += N
```

---

## 6. Live Session Flow

```
Both parties see "Join Session" button:
  - Active 5 minutes before scheduledAt
  - Button disabled outside window

Session start:
  GET /session/:id/token
    → Zoom SDK JWT generated (user + session ID encoded)
    → client joins Zoom Video SDK session

During session:
  - Countdown timer displayed (green → orange at 10min → red at 5min)
  - 10 minutes before end: "Extend Call" button appears

  Extend Call:
    Student taps extend
      → select duration (30/60/90 min)
      → GET /bookings/:id/extend → UPI URI + cost shown
      → student pays via UPI
      → POST /bookings/:id/extend/confirm
      → durationMinutes updated
      → countdown resets

Session end:
  POST /session/:id/end
    → Booking.status = completed
    → Educator.isLiveNow = false
    → educatorEarnings = durationMinutes × rate × (1 - platformFeePercent / 100)
    → Educator.pendingPayout += educatorEarnings
    → FCM push to student: "Rate your session"
    → reviewPromptSent = true

Missed session (cron job):
  Every 5 minutes: find sessions that should have started > threshold ago
  with sessionReady = false
    → auto-transition to disputed
    → FCM to both parties
```

---

## 7. Recording Flow

```
Session ends
  → Zoom cloud recording processes (async)
  → Zoom sends webhook: POST /recordings/zoom-webhook
      → HMAC verified
      → recording downloaded from Zoom
      → uploaded to S3 (s3_key stored)
      → SessionRecording created:
          status = ready
          expiresAt = recordedAt + 24hr

Student accesses recording:
  GET /recordings/:bookingId
    → check expiresAt > now
    → generate presigned S3 URL (1hr validity)
    → return URL
    → student watches in browser/app

After 24hr:
  Recording access expired (expiresAt check fails)
  S3 object may be deleted via lifecycle policy
```

---

## 8. Review Flow

```
After session ends:
  FCM push to student: "How was your session with [educator]?"
  Deep link → rate-session screen

Student submits review:
  POST /review/:bookingId
    rating: 1–5
    comment: optional
    → Review record created
    → Educator.overallRating recalculated (avg of all reviews)
    → EducatorTopic.avgRating updated

Admin moderation:
  Admin can flag reviews in mole-admin
  Flagged reviews hidden from educator profile
```

---

## 9. Dispute Flow

```
Student raises dispute:
  On cancelled or completed session → "Raise a Dispute" button
  Opens mailto: link with booking ID pre-filled
  OR POST /disputes/:bookingId
    → Booking.status = disputed
    → FCM to admin

Admin reviews:
  mole-admin /disputes
    → See booking details, parties, timeline
    → Add notes (DisputeNote records)
    → Resolve in favour of student or educator

Resolution options (manual):
  - Refund student (manual wallet credit via admin)
  - No action (educator keeps earnings)
  - Partial refund
```

---

## 10. Payout Flow

```
Session completes:
  Educator.pendingPayout += session earnings

Admin triggers payout:
  mole-admin /payouts
    → Select educator
    → Enter amount
    → POST /admin/payouts
        → PayoutHistory record created (status=pending)
        → Educator.pendingPayout -= amount

Manual bank transfer:
  Admin does IMPS/NEFT transfer outside platform
  Admin marks payout as paid:
    → PayoutHistory.status = paid
    → PayoutHistory.paidAt = now

Referral bonus payout:
  Same flow but type = referral_bonus
```

---

## 11. WhatsApp Webhook (Reply Flow)

```
Educator receives WhatsApp: "Accept? Reply YES or NO"
Educator replies "YES"
        │
        ▼
Twilio → POST /webhooks/whatsapp (form-encoded)
  → Parse message body (ButtonPayload / Body)
  → Extract sender phone from `From` (e.g. "whatsapp:+919876543210")
  → Match to User via phone variants (+91..., raw, etc.)
  → Find latest awaiting_educator booking for this educator
  → If YES:
      Booking → pending_payment
      Payment record created
      FCM to student
      WhatsApp confirmation back to educator: "Booking accepted! ✅"
  → If NO:
      Booking → cancelled
      FCM to student
      WhatsApp to educator: "Booking declined."
```

---

## 12. Wallet & Credits Flow

```
Credits earned:
  Referral: student uses educator referral code
    → WalletCredit created (source=referral, expiresAt=90 days)
    → Wallet.balance += amount
  Promotion: admin grants credits manually

Credits applied at booking:
  creditsApplied = min(wallet.balance, totalPrice)
  amountDue = totalPrice - creditsApplied
  WalletTransaction (debit) created
  Wallet.balance -= creditsApplied

Credits expiry (cron — daily midnight):
  Find WalletCredit where expiresAt < now AND usedAt IS NULL
    → Wallet.balance -= credit.amount
    → WalletTransaction (debit, reference="expiry")
    → credit.usedAt = now (marks as expired)
```
