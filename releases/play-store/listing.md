# MoLe — Google Play store listing

Copy + assets for **Play Console → Grow users → Store presence → Main store listing**
and **Store settings**. Language: **English (India) – en-IN** (add en-US with the same text if
wanted). Character limits are Play's; counts were checked.

## Main store listing

### App name (≤30)
```text
MoLe: Live 1:1 Tutoring
```

### Short description (≤80)
```text
Book live 1:1 video classes with verified tutors. Go deep. Learn fast.
```

### Full description (≤4000)
```text
MoLe connects students with verified tutors for live, one-to-one video classes. Pick the subject you're stuck on, choose a tutor who fits, and learn face to face, at your pace.

GO DEEP. LEARN FAST.

FOR STUDENTS
• Find the right tutor – search by subject, exam board, language and location, and compare ratings, reviews and prices
• Book your way – pick a time slot that suits you, or join a tutor who is live right now
• Learn live, one-to-one – HD video with in-session chat and screen sharing; extend a class while it's running
• Never lose a lesson – watch your session recording afterwards
• Check what you learned – take a short quiz after each class and earn Mole Coins for a perfect score
• Pay safely – UPI, cards and net banking through Razorpay, or use your MoLe wallet
• Stay on track – reminders and notifications for every booking
• Covered if plans change – cancel up to 1 hour before a class for wallet credit; if your tutor doesn't show up, we offer a replacement or a refund

FOR TUTORS
• Get verified once – identity and qualification checks build trust with students
• Teach on your terms – set your availability, subjects and prices
• Go live – switch on "Live now" to accept instant bookings
• Run great sessions – video, chat, screen sharing and a quiz for every class
• Get paid – track your earnings and receive payouts to your bank account

SAFE BY DESIGN
• Every tutor is verified before they can teach
• Phone-number sign-in with one-time password
• Payments handled by Razorpay; we never see your card or UPI details
• Report a problem or raise a dispute right from the app

Light and dark themes included.

Questions? Write to moleedtech@gmail.com
```

## Graphics

| Asset | Spec | File |
|---|---|---|
| App icon | 512×512 PNG, ≤1 MB, full-bleed square (Play rounds it) | [`icon-512.png`](icon-512.png) |
| Feature graphic | 1024×500, JPEG or 24-bit PNG (no alpha) | [`feature-graphic-1024x500.png`](feature-graphic-1024x500.png) |
| Phone screenshots | 2–8; 9:16 portrait; 1080×1920 recommended (min 320 px, max 3840 px). **≥4 at ≥1080 px** to be eligible for featuring | capture from the app, see below |
| Tablet screenshots | Optional (7" and 10") | skip for v1.0.0 |
| Video | Optional YouTube URL | skip |

Icon + feature graphic are generated from the logo package (`mole_app_icon.svg`,
`mole_logo_reversed.svg`); source SVGs live in the monorepo at `mobile/assets/brand/`.

### Screenshots to capture (in this order)
Use test accounts with realistic but **fake** names. No real student data, phone numbers or faces
of minors.

1. Welcome screen (logo + "Get Started")
2. Student home: live tutors + suggested for you
3. Search results with filters
4. Tutor profile: rating, subjects, price, Book button
5. Slot picker / booking confirmation
6. Live session: video + chat (two test devices)
7. Post-session quiz / Mole Coins reward
8. Tutor dashboard: today's sessions + earnings

Capture from a device or emulator in light mode:
```bash
adb exec-out screencap -p > 01-welcome.png
```
Optional: add a one-line caption above each screenshot on a brand-coloured background.

## Store settings
| Field | Value |
|---|---|
| App or game | App |
| Category | **Education** |
| Tags | Up to 5 from Play's list, e.g. Tutoring, Online learning, Education, Study tools, Exam prep (pick the closest available) |
| Email | moleedtech@gmail.com |
| Phone | (leave empty) |
| Website | https://app.moleedtech.com |
| Privacy policy (App content) | https://app.moleedtech.com/privacy-policy |
| Account deletion URL (Data safety) | https://app.moleedtech.com/delete-account |

## Policy notes for this copy
- No ranking or performance claims ("best", "#1", "top-rated"), no prices or promotions in the
  name or short description, no emoji. Play rejects these.
- Every feature listed is in the 1.0.0 build. Re-check before each release, along with
  [`../android-v1.0.0.md`](../android-v1.0.0.md).
- If the target audience includes under-13s (Families policy), screenshots and text must be
  suitable for children. Avoid anything that looks like ads or links out of the app.
