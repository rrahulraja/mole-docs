# Session Quiz + Mole Coin Reward — Design Spec

**Date:** 2026-07-13
**Branch:** `feature/session-quiz` (off `main`)
**Status:** Approved design → ready for implementation plan

## Goal

Let an educator attach a short 5-question quiz to a booked session. After the session
completes, the student can take the quiz once from the review page. If they get all 5
correct, they see a celebration (confetti + message) and earn **Mole Coins** (wallet credit).

## Global constraints

- MCQ, **single-correct, exactly 5 questions, 4 options each**.
- Grading is **all-or-nothing**: coins + celebration only on **5/5**.
- **One attempt per session** (per booking), pass or fail — no retake.
- Reward amount from platform config key **`quiz_reward_coins`** (default **10**), admin-editable
  via the existing `platformConfig` table (same mechanism as commission / referral bonus).
- "Mole Coins" == student wallet `balance`. Award = increment `Wallet.balance` **and** create a
  `WalletTransaction(type: credit, reference: "quiz_reward:<bookingId>")` in one transaction.
- Educator quiz setup available on **both web and mobile**, from the **Upcoming Session** area. Optional.
- Student takes the quiz on **both web and mobile**, from the existing review page.
- Additive, behind no flag; no change to existing booking/session/review/payment logic.

## Data model (Prisma, Postgres — `backend/prisma/schema.prisma`)

```prisma
model SessionQuiz {
  id         String   @id @default(cuid())
  bookingId  String   @unique @map("booking_id")
  questions  Json     // [{ question: string, options: string[4], correctIndex: 0..3 } x5]
  createdAt  DateTime @default(now()) @map("created_at")
  updatedAt  DateTime @updatedAt @map("updated_at")
  booking    Booking  @relation(fields: [bookingId], references: [id])
  @@map("session_quizzes")
}

model SessionQuizAttempt {
  id           String   @id @default(cuid())
  bookingId    String   @unique @map("booking_id")   // one attempt per session
  studentId    String   @map("student_id")
  answers      Json     // int[5], each 0..3
  correctCount Int      @map("correct_count")
  passed       Boolean
  coinsAwarded Int      @default(0) @map("coins_awarded")
  createdAt    DateTime @default(now()) @map("created_at")
  booking      Booking  @relation(fields: [bookingId], references: [id])
  @@map("session_quiz_attempts")
}
```
- Add back-relations on `Booking` (`quiz SessionQuiz?`, `quizAttempt SessionQuizAttempt?`).
- Migration must be additive; no backfill needed.
- Seed/ensure `platformConfig` has `quiz_reward_coins` = `10` (create-if-absent; do not overwrite).

**Why JSON for questions** (not a child table): always exactly 5 fixed-shape MCQs; a `Json` column
keeps reads/writes atomic and the code simple. `correctIndex` is **never** sent to the student.

## API (booking-scoped, `backend/src/controllers/`, mounted under `/bookings`)

All under existing `authenticate`. Ownership checked against the booking.

| Method | Path | Role | Behavior |
|---|---|---|---|
| `POST` | `/bookings/:id/quiz` | educator-owner | Create/replace the quiz. Body: `{ questions: [{ question, options[4], correctIndex } x5] }`. Rejected if an attempt already exists (`QUIZ_LOCKED`). |
| `GET` | `/bookings/:id/quiz` | educator-owner **or** student-owner | Educator: full quiz incl. `correctIndex`. Student: `{ exists, questions:[{question,options}], alreadyAttempted, rewardCoins, attempt? }` — **`correctIndex` stripped**. |
| `POST` | `/bookings/:id/quiz/attempt` | student-owner | Body `{ answers: int[5] }`. Preconditions: booking `completed`, quiz exists, no prior attempt. Grades server-side, writes the attempt, and on 5/5 credits coins (guarded by the unique attempt row). Returns `{ correctCount, passed, coinsAwarded, corrections: boolean[5] }`. |

**Error codes:** `NOT_FOUND` (404), `FORBIDDEN` (403, not a participant / wrong role),
`QUIZ_NOT_FOUND` (404, attempt with no quiz), `QUIZ_LOCKED` (409, edit after attempt),
`ALREADY_ATTEMPTED` (409), `BOOKING_NOT_COMPLETED` (422), `VALIDATION_ERROR` (400).

**Validation:** exactly 5 questions; each `question` non-empty; exactly 4 non-empty `options`;
`correctIndex` integer 0–3; `answers` length 5, each integer 0–3.

**Grading + reward (single DB transaction on attempt):**
1. Load quiz + verify no existing attempt (unique constraint is the backstop).
2. `correctCount = count(answers[i] === questions[i].correctIndex)`; `passed = correctCount === 5`.
3. If `passed`: read `quiz_reward_coins`; `coinsAwarded = reward`; upsert `Wallet` (`increment balance`)
   + create `WalletTransaction(credit, reference "quiz_reward:<bookingId>")`.
4. Create `SessionQuizAttempt`. Return result.
- Idempotency: the unique `bookingId` on the attempt prevents double-credit under retries/races.

## Educator UX (web + mobile) — Upcoming Session

- In the educator's session list, the **"Set up quick test"** action appears on any non-cancelled
  booking that has **no student attempt yet** (i.e. from confirmation through completion, until the
  student takes it). Shows a **"Test ready"** badge once saved / **"Edit test"** to modify.
- Form: 5 question blocks, each = question text + 4 option inputs + a radio to mark the correct option.
  Client validates all fields before enabling Save → `POST /bookings/:id/quiz`.
- Editable until the student submits an attempt; after that the API returns `QUIZ_LOCKED` and the UI shows read-only.
- Web: educator session/dashboard area (MUI). Mobile: educator dashboard/sessions (RN), reusing existing tokens/components.

## Student UX (web + mobile) — Review page

- On `/student/review/[bookingId]` (web) and the RN review screen: after loading, call `GET /bookings/:id/quiz`.
  - If `exists && !alreadyAttempted` → show **"Take quick test — earn N Mole Coins"** (separate from the rating/review submit; taking the test is independent of submitting a review).
  - If `alreadyAttempted` → show the prior result (score, passed/failed), no retake.
  - If no quiz → show nothing extra.
- Quiz view: 5 MCQs (radio per question), one **Submit** → `POST /bookings/:id/quiz/attempt`.
  - **5/5:** confetti + celebration message ("🎉 Perfect! N Mole Coins added to your wallet."); balance reflects in wallet.
  - **<5/5:** show `correctCount`/5 and per-question correct/incorrect (from `corrections`), no coins.
  - Attempt consumed either way.
- Confetti: web `canvas-confetti` (tiny); mobile `react-native-confetti-cannon` (small) or a lightweight animated emoji burst — keep deps minimal.

## Testing

- **Backend (unit/integration):** validation (bad counts/indices), ownership/role guards, edit-after-attempt lock,
  attempt-before-completed, grading correctness (5/5 vs 4/5), coin credit on pass + no credit on fail,
  double-attempt blocked (409), double-credit prevented, `correctIndex` never in student GET.
- **Web/Mobile:** educator create/edit + validation + locked state; student button visibility (exists/attempted/none),
  submit → pass path (confetti + coins) and fail path (score, no coins), attempt-consumed state.
- Add representative cases to `docs/test-cases/` (backend-api, web, mobile) under a new `QUIZ` area prefix.

## Out of scope

- Multiple quizzes per session; partial-credit rewards; retakes; question bank / reuse across sessions;
  timers; educator analytics on quiz results. (Can follow later.)
