# Mole Android — v1.0.0 release notes

| | |
|---|---|
| **Version** | `1.0.0` (`versionCode` set at build time; see [versioning](../deployment/google-play-release-plan.md#versioning)) |
| **Package** | `com.mole.app` |
| **Status** | Draft. Update before the production launch |
| **Min Android** | 9 (API 28) |
| **Tag** | `mobile-v1.0.0` (production); `mobile-v1.0.0+<code>` for each internal build |

---

## Play Store "What's new" (≤500 chars)

### Internal testing (testers)
```text
First Mole test build — thanks for testing!

Please try:
• Sign up with your phone number (OTP)
• Students: find an educator, book a slot, pay, join the live session, take the quiz, leave a review
• Educators: finish onboarding + ID check, set availability and prices, accept a booking, run a session
• Notifications, wallet, Mole Coins

Report bugs (with screenshots) in the Mole testers group.
```

### Production (users)
```text
Welcome to Mole — live 1:1 tutoring on your phone.

• Find educators by subject, language and location
• Book a slot or join an educator who's live right now
• HD video sessions with chat and screen sharing
• Pay securely with UPI, cards or your Mole wallet
• Earn Mole Coins by acing post-session quizzes
• Session recordings, reviews and reminders
• Educators: set your schedule and prices, track earnings and payouts
```

---

## What's in 1.0.0

### For students
- **Sign up in seconds**: phone number + OTP, short onboarding tutorial.
- **Find the right educator**: search by subject, teaching language and location, with regional
  pricing. Educator profiles show ratings and reviews.
- **Book your way**:
  - Pick a time slot (minimum session durations apply), or
  - **Live now**: instantly book an educator who's online.
- **Pay securely**: Razorpay (UPI, cards, netbanking) or your **Mole wallet**. Welcome and
  promotional credits apply automatically.
- **Live sessions**: HD video with in-session **chat** and **screen sharing**. Extend a session
  while it's running. The session ends with an OTP confirmation between you and the educator.
- **If your educator doesn't show up**: your booking can be reassigned to another educator.
- **After the session**:
  - 5-question quiz: score 5/5 to earn **Mole Coins** in your wallet.
  - Rate and review your educator.
  - Watch the **session recording** (available for a limited time).
  - Raise a dispute if something went wrong.
- **Refer friends** and earn a bonus.
- **Stay updated**: push and in-app notifications, calendar reminders.
- **Light and dark themes.**

### For educators
- **Onboarding + verification**: profile, subjects, languages, and upload of identity and education
  documents for review.
- **Your schedule, your prices**: calendar availability, per-session pricing, minimum durations.
- **Bookings**: accept or decline requests; go **live** to receive instant bookings.
- **Teach**: same HD video, chat and screen sharing. Attach a quiz to each session.
- **Earnings**: earnings dashboard, payout setup and payout history.

### Platform
- **Force update**: the app prompts users to update when a release is mandatory.
- Android 9+ supported. Push notifications via Firebase.

---

## Known issues / limitations
- **Notification icon** shows as a plain white shape on some devices (monochrome icon pending).
- After updating from an older test build, some launchers' **app search** may show the old icon
  until the phone restarts.
- **Over-the-air updates are not active in this build** (no update server configured). Every fix
  needs a new store build.
- Android only. iOS is not released yet.

---

## Before publishing to production (checklist)
- [ ] Re-read every bullet against the final tested build; remove anything not shipped
- [ ] Fill in the `versionCode` of the build being promoted
- [ ] Production `What's new` text pasted in Play Console (English; add Hindi later if wanted)
- [ ] Known-issues list updated
- [ ] Git tag `mobile-v1.0.0` pushed; GitHub release created from the "What's in 1.0.0" section
