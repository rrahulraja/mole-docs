# Database Schema — Mole

PostgreSQL database managed via Prisma ORM.

---

## Enums

| Enum | Values |
|------|--------|
| `UserRole` | `student`, `educator`, `admin` |
| `EducatorStatus` | `pending`, `approved`, `rejected`, `suspended` |
| `EducatorTopicStatus` | `pending_review`, `approved` |
| `SessionType` | `teaching`, `revision`, `doubt_clearing` |
| `BookingStatus` | `awaiting_educator`, `pending_payment`, `confirmed`, `completed`, `failed`, `disputed`, `cancelled` |
| `PaymentStatus` | `pending`, `success`, `failed` |
| `WalletTransactionType` | `credit`, `debit` |
| `ClipType` | `demo`, `intro`, `other` |
| `ClipStatus` | `pending`, `approved`, `rejected` |
| `PayoutType` | `session_earnings`, `referral_bonus` |
| `PayoutStatus` | `pending`, `paid` |

---

## Tables

### `users`
Core user record. Both students and educators have a User.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `phone` | String UNIQUE | Primary identifier |
| `email` | String? | Optional |
| `name` | String | |
| `role` | UserRole | `student` \| `educator` \| `admin` |
| `fcm_token` | String? | Firebase push token |
| `is_active` | Boolean | Default true |
| `referral_code` | String? UNIQUE | User referral code |
| `terms_accepted_at` | DateTime? | |
| `terms_accepted_version` | String? | |
| `created_at` | DateTime | |

Relations: `Student`, `Educator`, `RefreshToken[]`, `Wallet`, `Notification[]`, `DisputeNote[]`, `EducatorReport[]`

---

### `students`
Student-specific profile. One-to-one with `users`.

| Column | Type | Notes |
|--------|------|-------|
| `user_id` | UUID PK FK→users | |
| `grade_level` | String? | e.g. "Class 10", "Class 12" |
| `exam_boards` | String[] | e.g. ["CBSE", "ICSE"] — used to filter educators |
| `referred_by` | String? | Educator referral code used at signup |
| `onboarding_seen_at` | DateTime? | |

Relations: `Booking[]`, `WalletCredit[]`

---

### `educators`
Educator profile. One-to-one with `users`.

| Column | Type | Notes |
|--------|------|-------|
| `user_id` | UUID PK FK→users | |
| `status` | EducatorStatus | Default `pending` |
| `rejection_reason` | String? | Set on admin reject |
| `rate_teaching` | Int? | ₹ per minute |
| `rate_revision` | Int? | ₹ per minute |
| `rate_doubt_clearing` | Int? | ₹ per minute |
| `is_live_now` | Boolean | Updated when session starts/ends |
| `overall_rating` | Decimal(3,2)? | Aggregated from reviews |
| `referral_code` | String? UNIQUE | Educator's referral code |
| `referral_count` | Int | Default 0 |
| `city` | String? | |
| `qualifications` | Json? | Array of qualification objects |
| `govt_id_url` | String? | S3 key or CDN URL |
| `teaching_level` | String? | |
| `bio` | String? | |
| `selfie_url` | String? | Profile photo |
| `linkedin_url` | String? | |
| `bank_account_name` | String? | |
| `bank_account_number` | String? | |
| `bank_ifsc_code` | String? | |
| `bank_account_type` | String? | |
| `years_experience` | Int? | |
| `exam_boards` | String[] | Boards educator can teach |
| `subjects_interested` | String[] | |
| `pending_payout` | Float | Accumulated earnings not yet paid |
| `education_proof_url` | String? | |
| `education_proof_verified_at` | DateTime? | |

Relations: `EducatorTopic[]`, `DemoClip[]`, `Booking[]`, `PayoutHistory[]`, `EducatorAvailability[]`

---

### `subjects`
Top-level subject catalog.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `name` | String UNIQUE | e.g. "Mathematics" |
| `is_active` | Boolean | |

Relations: `Topic[]`

---

### `topics`
Hierarchical topics under subjects. Supports parent/child nesting.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `subject_id` | UUID FK→subjects | |
| `name` | String | |
| `parent_topic_id` | UUID? FK→topics | Self-referential |
| `is_active` | Boolean | |

Unique: `(subject_id, name)`

---

### `educator_topics`
Junction: which topics an educator can teach, with approval status.

| Column | Type | Notes |
|--------|------|-------|
| `educator_id` | UUID FK→educators | PK composite |
| `topic_id` | UUID FK→topics | PK composite |
| `avg_rating` | Decimal(3,2)? | Avg rating for this topic |
| `status` | EducatorTopicStatus | `pending_review` \| `approved` |

---

### `demo_clips`
Educator video clips uploaded for profile.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `educator_id` | UUID FK→educators | |
| `storage_url` | String | S3 key or CDN URL |
| `duration_seconds` | Int | |
| `type` | ClipType | `demo` \| `intro` \| `other` |
| `status` | ClipStatus | `pending` \| `approved` \| `rejected` |
| `created_at` | DateTime | |

> Only `approved` clips shown to students in search/profile.

---

### `bookings`
Central booking record linking student ↔ educator ↔ topic.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `student_id` | UUID FK→students | |
| `educator_id` | UUID FK→educators | |
| `topic_id` | UUID FK→topics | |
| `session_type` | SessionType | |
| `duration_minutes` | Int | Can increase via extension |
| `total_price` | Int | Full price (before credits) |
| `credits_applied` | Int | Default 0 |
| `amount_due` | Int | `total_price - credits_applied` |
| `status` | BookingStatus | See status flow below |
| `zoom_session_id` | String? | Zoom session identifier |
| `scheduled_at` | DateTime? | |
| `student_joined_at` | DateTime? | |
| `educator_joined_at` | DateTime? | |
| `session_ready` | Boolean | Both parties joined |
| `review_prompt_sent` | Boolean | FCM prompt sent after session |
| `session_end_otp` | String? | OTP to verify session end |
| `session_end_otp_expires_at` | DateTime? | |
| `penalty_pct` | Int | 0 or 10 — late cancellation penalty |
| `student_call_sent_at` | DateTime? | Reminder call timestamp |
| `educator_call_sent_at` | DateTime? | |
| `created_at` | DateTime | |

Relations: `Payment?`, `Review?`, `DisputeNote[]`, `SessionRecording?`

**Booking Status Flow:**
```
awaiting_educator → pending_payment → confirmed → completed
      ↓                                   ↓
  cancelled                           disputed
```

---

### `payments`
One payment per booking.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `booking_id` | UUID UNIQUE FK→bookings | |
| `upi_transaction_ref` | String? | From UPI webhook |
| `status` | PaymentStatus | `pending` \| `success` \| `failed` |
| `amount` | Int | In ₹ |
| `webhook_received_at` | DateTime? | |

---

### `wallets`
Student wallet balance.

| Column | Type | Notes |
|--------|------|-------|
| `student_id` | UUID PK FK→users | |
| `balance` | Int | In ₹ |
| `updated_at` | DateTime | Auto-updated |

---

### `wallet_transactions`
Audit trail of wallet credits/debits.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `student_id` | UUID FK→wallets | |
| `type` | WalletTransactionType | `credit` \| `debit` |
| `amount` | Int | |
| `reference` | String? | Booking ID or promo code |
| `expires_at` | DateTime? | For expiring credits |
| `created_at` | DateTime | |

---

### `wallet_credits`
Promotional or bonus credits with expiry.

| Column | Type | Notes |
|--------|------|-------|
| `id` | CUID PK | |
| `student_id` | UUID FK→students | |
| `amount` | Int | |
| `source` | String | e.g. "referral", "promo" |
| `expires_at` | DateTime | |
| `used_at` | DateTime? | |
| `created_at` | DateTime | |

---

### `referrals`
Tracks educator referrals to students.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `educator_id` | UUID FK→educators | Who referred |
| `student_id` | UUID UNIQUE FK→users | Who was referred |
| `used_at` | DateTime | When referral code was used |

---

### `payout_history`
Educator earnings payout records.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `educator_id` | UUID FK→educators | |
| `amount` | Int | In ₹ |
| `type` | PayoutType | `session_earnings` \| `referral_bonus` |
| `status` | PayoutStatus | `pending` \| `paid` |
| `paid_at` | DateTime? | |

---

### `platform_config`
Key-value store for platform-wide settings.

| Column | Type | Notes |
|--------|------|-------|
| `key` | String PK | Config key |
| `value` | Json | Config value (any JSON) |
| `updated_at` | DateTime | |
| `updated_by` | UUID? FK→users | Admin who last changed |

---

### `refresh_tokens`
JWT refresh token store.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `user_id` | UUID FK→users | |
| `token_hash` | String | bcrypt hash of token |
| `expires_at` | DateTime | |
| `created_at` | DateTime | |

Index: `user_id`

---

### `otp_codes`
OTP records for phone verification.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `phone` | String | |
| `code` | String | 6-digit code |
| `expires_at` | DateTime | 5min from creation |
| `used` | Boolean | Prevents replay |
| `created_at` | DateTime | |

Index: `phone`

---

### `educator_availability`
Weekly availability slots per educator.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `educator_id` | UUID FK→educators | |
| `day_of_week` | Int | 0=Sunday … 6=Saturday |
| `start_time` | String | "09:00" |
| `end_time` | String | "17:00" |
| `is_active` | Boolean | |

---

### `admins`
Admin user accounts (separate from `users`).

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `username` | String UNIQUE | |
| `password_hash` | String | bcrypt |
| `name` | String | |
| `is_active` | Boolean | |
| `created_at` | DateTime | |
| `created_by_id` | UUID? FK→admins | Self-referential |

---

### `admin_audit_logs`
Immutable audit trail of admin actions.

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `admin_id` | UUID FK→admins | |
| `action` | String | e.g. `approve_educator`, `resolve_dispute` |
| `target_type` | String | `educator`, `dispute`, `booking`, `config`, `admin` |
| `target_id` | String | ID of affected record |
| `note` | String? | Admin's note |
| `created_at` | DateTime | |

---

### `reviews`
Student reviews of completed sessions.

| Column | Type | Notes |
|--------|------|-------|
| `id` | CUID PK | |
| `booking_id` | String UNIQUE FK→bookings | |
| `student_id` | UUID | |
| `educator_id` | UUID | |
| `rating` | Int | 1–5 |
| `comment` | String? | |
| `flagged_at` | DateTime? | Admin flagged |
| `flag_reason` | String? | |
| `created_at` | DateTime | |

---

### `notifications`
In-app notifications for users.

| Column | Type | Notes |
|--------|------|-------|
| `id` | CUID PK | |
| `user_id` | UUID FK→users | |
| `title` | String | |
| `body` | String | |
| `deep_link` | String? | In-app route |
| `read_at` | DateTime? | Null = unread |
| `created_at` | DateTime | |

Index: `(user_id, created_at DESC)`

---

### `dispute_notes`
Threaded notes on a disputed booking.

| Column | Type | Notes |
|--------|------|-------|
| `id` | CUID PK | |
| `booking_id` | UUID FK→bookings | |
| `author_id` | UUID FK→users | Student or admin |
| `body` | String | |
| `created_at` | DateTime | |

---

### `educator_reports`
Student reports against educators.

| Column | Type | Notes |
|--------|------|-------|
| `id` | CUID PK | |
| `educator_id` | UUID FK→users | Reported educator |
| `reporter_id` | UUID FK→users | Student who reported |
| `reason` | String | |
| `details` | String? | |
| `status` | String | `pending` \| `reviewed` \| `dismissed` |
| `created_at` | DateTime | |

---

### `session_recordings`
Zoom recording metadata after processing.

| Column | Type | Notes |
|--------|------|-------|
| `id` | CUID PK | |
| `booking_id` | UUID UNIQUE FK→bookings | |
| `s3_key` | String | S3 object key |
| `zoom_file_id` | String? | Zoom's file ID |
| `duration_secs` | Int? | |
| `file_size_bytes` | Int? | |
| `status` | String | `processing` \| `ready` \| `failed` |
| `recorded_at` | DateTime | |
| `expires_at` | DateTime | `recorded_at + 24hr` — student access window |
| `created_at` | DateTime | |
