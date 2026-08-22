# End-to-End Architecture — Mole

## Overview

Mole is a peer-to-peer tutoring marketplace. Students discover and book live 1:1 video sessions with educators. Sessions run on Zoom Video SDK. Payments via UPI. Educator notifications via WhatsApp.

---

## System Components

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                              │
│                                                                  │
│   mole-mobile (Expo/RN)          mole-admin (React/Vite)        │
│   ┌──────────────┐               ┌──────────────────┐           │
│   │  Student App  │               │  Admin Dashboard  │          │
│   │  Educator App │               │  (internal only)  │          │
│   └──────┬───────┘               └────────┬─────────┘           │
└──────────┼────────────────────────────────┼─────────────────────┘
           │ HTTPS + JWT                    │ HTTPS + Admin JWT
           ▼                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      mole-backend (Express)                      │
│                                                                  │
│  /api/v1/auth        /api/v1/bookings    /api/v1/session         │
│  /api/v1/educator    /api/v1/payment     /api/v1/discover        │
│  /api/v1/student     /api/v1/review      /api/v1/admin           │
│  /api/v1/webhooks/whatsapp               /api/v1/recordings      │
│                                                                  │
│  Middleware: JWT auth │ role guard │ rate limit │ Helmet          │
└──────────────────────┬──────────────────────────────────────────┘
                       │
         ┌─────────────┼─────────────────┐
         ▼             ▼                 ▼
   PostgreSQL      AWS S3           External APIs
   (Prisma ORM)   (media/docs)
                                ┌────────────────┐
                                │  Twilio SMS     │  OTP delivery
                                │  Firebase FCM   │  Push notifications
                                │  Zoom Video SDK │  Session tokens
                                │ Twilio WhatsApp │  Educator notifications
                                │  Sentry         │  Error tracking
                                └────────────────┘
```

---

## Authentication Flow

```
Mobile App                     Backend                    Twilio
    │                              │                         │
    │── POST /auth/send-otp ──────▶│── sendSMS(phone, code)─▶│
    │                              │                         │
    │◀─ { message: "OTP sent" } ───│                         │
    │                              │
    │── POST /auth/verify-otp ────▶│ verify OTP in db
    │                              │ create/find User
    │                              │ if new → TEMP token
    │                              │ if existing → access + refresh tokens
    │◀─ { accessToken, refreshToken, user } ──────────────────│
    │                              │
    │  (token stored in SecureStore)
    │
    │── All requests: Authorization: Bearer <accessToken>
    │
    │── POST /auth/refresh ────────▶│ verify refreshToken
    │◀─ { accessToken } ────────────│ issue new accessToken
```

---

## Booking Flow (with WhatsApp)

```
Student                    Backend                  Educator (WhatsApp)
   │                          │                            │
   │── POST /bookings ────────▶│
   │                          │ create Booking(awaiting_educator)
   │                          │ sign JWT {bookingId, action}
   │                          │── sendWhatsApp(educator) ──▶│
   │◀─ { status: awaiting } ──│    "Accept: /respond?t=..."  │
   │                          │    "Decline: /respond?t=..."  │
   │  [waiting state shown]   │                              │
   │                          │◀─ educator taps Accept link ─│
   │                          │   OR replies "YES"
   │                          │
   │                          │ verify JWT
   │                          │ Booking → pending_payment
   │                          │ create Payment record
   │                          │── FCM push to student ──▶│
   │◀─ push: "Educator accepted" │
   │                          │
   │── POST /payment/:id/intent▶│ generate UPI URI
   │◀─ { upiUri } ────────────│
   │── Linking.openURL(upiUri)│  [student pays in UPI app]
   │                          │
   │                          │◀─ UPI webhook (HMAC verified)
   │                          │ Payment → success
   │                          │ Booking → confirmed
   │                          │── FCM push to both ─────▶│
   │◀─ push: "Booking confirmed"
```

---

## Live Session Flow

```
Student App              Backend                  Educator App
    │                       │                          │
    │── GET /session/:id/token ──▶│                    │
    │                       │ generate Zoom JWT         │
    │◀─ { sessionToken }    │                           │
    │                       │                          │── GET /session/:id/token
    │                       │── Zoom JWT ─────────────▶│
    │                       │                          │
    │── join Zoom session ──────────────────────────── │
    │                       │                          │
    │  [live video call]     │                         │
    │                       │                          │
    │  [10min before end]    │                         │
    │  show Extend Call btn  │                         │
    │── GET /bookings/:id/extend ▶│                    │
    │◀─ { upiUri, cost }    │                          │
    │  [student pays]        │                         │
    │── POST /extend/confirm▶│ durationMinutes += N    │
    │                       │                          │
    │  [session ends]        │                         │
    │── POST /session/:id/end▶│                        │
    │                       │ Booking → completed       │
    │                       │ educator pendingPayout += │
    │                       │── FCM: "Rate your session"│
```

---

## Recording Flow

```
Zoom Cloud                 Backend                   Student App
     │                        │                          │
     │── POST /recordings/webhook ──▶│                   │
     │   (HMAC verified)      │                          │
     │                        │ download recording       │
     │                        │── upload to S3 ─────────▶│
     │                        │ create SessionRecording   │
     │                        │ expiresAt = now + 24hr    │
     │                        │                          │
     │                        │◀─ GET /recordings/:id ───│
     │                        │ check expiresAt           │
     │                        │ generate S3 presigned URL │
     │                        │── { url } ───────────────▶│
```

---

## Infrastructure Stack

| Service | Purpose | Provider |
|---------|---------|---------|
| Database | PostgreSQL | Self-hosted / Managed PG (Railway, Supabase, RDS) |
| File Storage | Media, docs, clips | AWS S3 |
| Push Notifications | FCM | Firebase |
| SMS / OTP | Phone verification | Twilio |
| Video Sessions | Live 1:1 calls | Zoom Video SDK |
| WhatsApp | Educator booking notifications | Twilio |
| Error Monitoring | Runtime errors | Sentry |
| Mobile Build | App distribution | Expo EAS |
| CDN | Media delivery | AWS CloudFront (optional) |

---

## Security

| Concern | Approach |
|---------|---------|
| API Auth | JWT access token (15min) + refresh token (30d) |
| Admin Auth | Separate admin JWT, `authenticateAdmin` middleware |
| OTP | 6-digit code, 5min expiry, `used` flag prevents replay |
| Webhook integrity | HMAC-SHA256 on UPI + Zoom webhooks |
| WhatsApp accept links | JWT-signed token with 24hr expiry |
| File uploads | S3 presigned URLs, no direct public access |
| Rate limiting | `express-rate-limit` on auth endpoints |
| SQL injection | Prisma parameterised queries |
| XSS | Helmet headers |
| Secrets | Environment variables via Zod-validated config |

---

## Data Flow Diagram

```
User Phone
    │ OTP
    ▼
[Auth] ──── JWT ────▶ [All protected routes]
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
              [Student routes]    [Educator routes]
                    │                   │
              [Booking service]         │
                    │                   │
                    ├── WhatsApp ───────▶ Educator phone
                    ├── FCM ────────────▶ Student device
                    │
              [Payment service]
                    │
                    ├── UPI URI ─────────▶ UPI App
                    └── Webhook ◀──────── UPI Gateway
                              │
                        [Session service]
                              │
                        Zoom Video SDK
                              │
                        [Recording service]
                              │
                            AWS S3
```
