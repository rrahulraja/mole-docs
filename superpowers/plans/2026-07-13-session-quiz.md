# Session Quiz + Mole Coin Reward — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Educators attach a 5-question MCQ quiz to a booking; after the session completes the student takes it once from the review page, and a perfect 5/5 credits Mole Coins with an on-screen celebration.

**Architecture:** Two new Prisma tables (`SessionQuiz`, `SessionQuizAttempt`) on the existing Booking. Pure, unit-tested backend helpers for validation + grading; a thin booking-scoped controller for persistence, guards, and the reward transaction. Web (MUI) and mobile (RN) get an educator setup form (from the dashboard session cards) and a student quiz view (on the existing review screen) with confetti.

**Tech Stack:** Express + Prisma (Postgres) backend; Next.js 14 + MUI web; Expo/React Native mobile; Jest + ts-jest + supertest (already deps, not yet configured).

## Global Constraints

- MCQ single-correct; **exactly 5 questions, 4 options each**, `correctIndex` 0–3.
- Grading **all-or-nothing**; coins + celebration only on **5/5**.
- **One attempt per session** (enforced by `SessionQuizAttempt.bookingId @unique`); no retake.
- Reward amount from platform config key **`quiz_reward_coins`** (default **10**), read via `platformConfig` (same pattern as commission in `earning.controller.ts`).
- Mole Coins == student wallet `balance`. Award = increment `Wallet.balance` **and** create `WalletTransaction(type: 'credit', reference: "quiz_reward:<bookingId>")` in one `prisma.$transaction`.
- `correctIndex` is **never** returned to the student.
- Educator setup on **web + mobile** from the dashboard session cards; editable until the student attempts (then `QUIZ_LOCKED`).
- Student takes it on **web + mobile** review screen; independent of submitting the rating/review.
- Additive only — no change to existing booking/session/review/payment behavior. All work on branch `feature/session-quiz`.
- Use `@mole/shared` tokens (`C`, `S`) on frontends; no hardcoded hex that duplicates a token.

## File Structure

**Backend**
- `backend/prisma/schema.prisma` — add `SessionQuiz`, `SessionQuizAttempt`, Booking back-relations *(Task 1)*
- `backend/prisma/migrations/<ts>_add_session_quiz/migration.sql` — additive migration *(Task 1)*
- `backend/jest.config.ts` — Jest/ts-jest config *(Task 2)*
- `backend/src/services/quiz/grade.ts` — pure `validateQuizInput`, `gradeQuiz` *(Task 2)*
- `backend/src/services/quiz/grade.test.ts` — unit tests *(Task 2)*
- `backend/src/controllers/quiz.controller.ts` — `upsertQuiz`, `getQuiz`, `submitAttempt` *(Task 3)*
- `backend/src/routes/v1/booking.ts` — mount quiz routes *(Task 3)*

**Web**
- `web/services/quiz.ts` — typed API calls *(Task 4)*
- `web/components/quiz/StudentQuiz.tsx` — student quiz view + confetti *(Task 4)*
- `web/app/student/review/[bookingId]/page.tsx` — mount StudentQuiz *(Task 4)*
- `web/components/quiz/QuizSetupDialog.tsx` — educator setup form *(Task 5)*
- `web/app/educator/page.tsx` — "Set up quick test" button on schedule cards *(Task 5)*
- `web/package.json` — add `canvas-confetti` *(Task 4)*

**Mobile**
- `mobile/services/quiz.ts` — typed API calls *(Task 6)*
- `mobile/app/(student)/quiz/[bookingId].tsx` — student quiz screen + confetti *(Task 6)*
- `mobile/app/(student)/review/[bookingId].tsx` — link to quiz screen *(Task 6)*
- `mobile/app/(educator)/quiz-setup/[bookingId].tsx` — educator setup screen *(Task 7)*
- `mobile/app/(educator)/index.tsx` — "Set up quick test" on schedule cards *(Task 7)*
- `mobile/package.json` — add `react-native-confetti-cannon` *(Task 6)*

**Docs**
- `docs/test-cases/backend-api.md`, `web.md`, `mobile.md` — add `QUIZ` cases *(Task 8)*

---

## Task 1: Data model + migration + reward config default

**Files:**
- Modify: `backend/prisma/schema.prisma`
- Create: `backend/prisma/migrations/20260713120000_add_session_quiz/migration.sql`

**Interfaces:**
- Produces: `SessionQuiz { id, bookingId, questions: Json, createdAt, updatedAt }`, `SessionQuizAttempt { id, bookingId, studentId, answers: Json, correctCount, passed, coinsAwarded, createdAt }`. `questions` JSON shape: `Array<{ question: string; options: string[]; correctIndex: number }>` (length 5). `answers` JSON: `number[]` (length 5).

- [ ] **Step 1: Add models to `schema.prisma`** (after the `Review` model)

```prisma
model SessionQuiz {
  id        String   @id @default(cuid())
  bookingId String   @unique @map("booking_id")
  questions Json
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")
  booking   Booking  @relation(fields: [bookingId], references: [id])

  @@map("session_quizzes")
}

model SessionQuizAttempt {
  id           String   @id @default(cuid())
  bookingId    String   @unique @map("booking_id")
  studentId    String   @map("student_id")
  answers      Json
  correctCount Int      @map("correct_count")
  passed       Boolean
  coinsAwarded Int      @default(0) @map("coins_awarded")
  createdAt    DateTime @default(now()) @map("created_at")
  booking      Booking  @relation(fields: [bookingId], references: [id])

  @@map("session_quiz_attempts")
}
```

- [ ] **Step 2: Add back-relations to the `Booking` model** (inside `model Booking { ... }`, near the other relations)

```prisma
  quiz        SessionQuiz?
  quizAttempt SessionQuizAttempt?
```

- [ ] **Step 3: Validate the schema**

Run: `cd backend && npx prisma validate`
Expected: `The schema at prisma/schema.prisma is valid 🚀`

- [ ] **Step 4: Write the migration SQL** (`backend/prisma/migrations/20260713120000_add_session_quiz/migration.sql`)

```sql
-- CreateTable
CREATE TABLE "session_quizzes" (
    "id" TEXT NOT NULL,
    "booking_id" TEXT NOT NULL,
    "questions" JSONB NOT NULL,
    "created_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updated_at" TIMESTAMP(3) NOT NULL,
    CONSTRAINT "session_quizzes_pkey" PRIMARY KEY ("id")
);

CREATE TABLE "session_quiz_attempts" (
    "id" TEXT NOT NULL,
    "booking_id" TEXT NOT NULL,
    "student_id" TEXT NOT NULL,
    "answers" JSONB NOT NULL,
    "correct_count" INTEGER NOT NULL,
    "passed" BOOLEAN NOT NULL,
    "coins_awarded" INTEGER NOT NULL DEFAULT 0,
    "created_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT "session_quiz_attempts_pkey" PRIMARY KEY ("id")
);

-- Unique + FKs
CREATE UNIQUE INDEX "session_quizzes_booking_id_key" ON "session_quizzes"("booking_id");
CREATE UNIQUE INDEX "session_quiz_attempts_booking_id_key" ON "session_quiz_attempts"("booking_id");
ALTER TABLE "session_quizzes" ADD CONSTRAINT "session_quizzes_booking_id_fkey" FOREIGN KEY ("booking_id") REFERENCES "bookings"("id") ON DELETE RESTRICT ON UPDATE CASCADE;
ALTER TABLE "session_quiz_attempts" ADD CONSTRAINT "session_quiz_attempts_booking_id_fkey" FOREIGN KEY ("booking_id") REFERENCES "bookings"("id") ON DELETE RESTRICT ON UPDATE CASCADE;

-- Seed the reward config if absent (id/key column names follow platform_config table)
INSERT INTO "platform_config" ("key", "value")
VALUES ('quiz_reward_coins', '10')
ON CONFLICT ("key") DO NOTHING;
```

> Before writing, confirm the real column names of `platform_config` in the schema (`key`, `value`, and whether an `id`/`updated_at` is required). Adjust the INSERT to match; if the table needs an `id`, use `gen_random_uuid()` or the table's default. If unsure, omit the INSERT here and rely on the config helper's default (Task 3, Step 3) — the helper already defaults to 10.

- [ ] **Step 5: Apply migration + regenerate client**

Run: `cd backend && npx prisma migrate deploy && npx prisma generate`
Expected: migration `20260713120000_add_session_quiz` applied; client regenerated.

- [ ] **Step 6: Commit**

```bash
git add backend/prisma/schema.prisma backend/prisma/migrations/20260713120000_add_session_quiz/
git commit -m "feat(quiz): add SessionQuiz + SessionQuizAttempt models and migration"
```

---

## Task 2: Pure grading + validation helpers (TDD)

**Files:**
- Create: `backend/jest.config.ts`, `backend/src/services/quiz/grade.ts`, `backend/src/services/quiz/grade.test.ts`

**Interfaces:**
- Produces:
  - `type QuizQuestion = { question: string; options: string[]; correctIndex: number }`
  - `validateQuizInput(questions: unknown): { ok: true; questions: QuizQuestion[] } | { ok: false; error: string }`
  - `gradeQuiz(questions: QuizQuestion[], answers: number[]): { correctCount: number; passed: boolean; corrections: boolean[] }`

- [ ] **Step 1: Add `backend/jest.config.ts`**

```ts
import type { Config } from 'jest';
const config: Config = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  testMatch: ['**/*.test.ts'],
  roots: ['<rootDir>/src'],
};
export default config;
```

- [ ] **Step 2: Write the failing tests** (`backend/src/services/quiz/grade.test.ts`)

```ts
import { validateQuizInput, gradeQuiz, QuizQuestion } from './grade';

const q = (correctIndex: number): QuizQuestion => ({
  question: 'Q?', options: ['a', 'b', 'c', 'd'], correctIndex,
});
const five = [q(0), q(1), q(2), q(3), q(0)];

describe('validateQuizInput', () => {
  it('accepts exactly 5 well-formed questions', () => {
    expect(validateQuizInput(five)).toEqual({ ok: true, questions: five });
  });
  it('rejects wrong question count', () => {
    expect(validateQuizInput([q(0)]).ok).toBe(false);
  });
  it('rejects != 4 options', () => {
    expect(validateQuizInput([{ question: 'x', options: ['a', 'b', 'c'], correctIndex: 0 }, q(0), q(0), q(0), q(0)]).ok).toBe(false);
  });
  it('rejects empty option text', () => {
    expect(validateQuizInput([{ question: 'x', options: ['a', '', 'c', 'd'], correctIndex: 0 }, q(0), q(0), q(0), q(0)]).ok).toBe(false);
  });
  it('rejects correctIndex out of range', () => {
    expect(validateQuizInput([q(4), q(0), q(0), q(0), q(0)]).ok).toBe(false);
  });
  it('rejects empty question text', () => {
    expect(validateQuizInput([{ question: '', options: ['a', 'b', 'c', 'd'], correctIndex: 0 }, q(0), q(0), q(0), q(0)]).ok).toBe(false);
  });
});

describe('gradeQuiz', () => {
  it('passes on all correct', () => {
    const r = gradeQuiz(five, [0, 1, 2, 3, 0]);
    expect(r).toEqual({ correctCount: 5, passed: true, corrections: [true, true, true, true, true] });
  });
  it('fails on one wrong', () => {
    const r = gradeQuiz(five, [0, 1, 2, 3, 1]);
    expect(r.correctCount).toBe(4);
    expect(r.passed).toBe(false);
    expect(r.corrections[4]).toBe(false);
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `cd backend && npx jest src/services/quiz/grade.test.ts`
Expected: FAIL — `Cannot find module './grade'`.

- [ ] **Step 4: Implement `backend/src/services/quiz/grade.ts`**

```ts
export type QuizQuestion = { question: string; options: string[]; correctIndex: number };

export function validateQuizInput(
  input: unknown,
): { ok: true; questions: QuizQuestion[] } | { ok: false; error: string } {
  if (!Array.isArray(input) || input.length !== 5) return { ok: false, error: 'Exactly 5 questions required' };
  for (const q of input as any[]) {
    if (!q || typeof q.question !== 'string' || q.question.trim() === '') return { ok: false, error: 'Each question needs text' };
    if (!Array.isArray(q.options) || q.options.length !== 4) return { ok: false, error: 'Each question needs 4 options' };
    if (q.options.some((o: unknown) => typeof o !== 'string' || o.trim() === '')) return { ok: false, error: 'Options cannot be empty' };
    if (!Number.isInteger(q.correctIndex) || q.correctIndex < 0 || q.correctIndex > 3) return { ok: false, error: 'correctIndex must be 0-3' };
  }
  return { ok: true, questions: input as QuizQuestion[] };
}

export function gradeQuiz(questions: QuizQuestion[], answers: number[]) {
  const corrections = questions.map((q, i) => answers[i] === q.correctIndex);
  const correctCount = corrections.filter(Boolean).length;
  return { correctCount, passed: correctCount === questions.length, corrections };
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd backend && npx jest src/services/quiz/grade.test.ts`
Expected: PASS — 8 passing.

- [ ] **Step 6: Commit**

```bash
git add backend/jest.config.ts backend/src/services/quiz/
git commit -m "feat(quiz): pure validate + grade helpers with unit tests"
```

---

## Task 3: Quiz controller + routes + reward transaction

**Files:**
- Create: `backend/src/controllers/quiz.controller.ts`
- Modify: `backend/src/routes/v1/booking.ts`

**Interfaces:**
- Consumes: `validateQuizInput`, `gradeQuiz` (Task 2); `authenticate`, `requireRole` middleware; `prisma`.
- Produces routes under `/bookings`:
  - `POST /:id/quiz` (educator) — body `{ questions }`
  - `GET /:id/quiz` (educator or student)
  - `POST /:id/quiz/attempt` (student) — body `{ answers: number[] }`

- [ ] **Step 1: Implement `backend/src/controllers/quiz.controller.ts`**

```ts
import { Request, Response } from 'express';
import { prisma } from '../lib/prisma';
import { AuthenticatedRequest } from '../middleware/authenticate';
import { validateQuizInput, gradeQuiz, QuizQuestion } from '../services/quiz/grade';

async function loadBooking(id: string) {
  return prisma.booking.findUnique({
    where: { id },
    include: { quiz: true, quizAttempt: true, educator: true },
  });
}

// POST /bookings/:id/quiz  (educator-owner)
export async function upsertQuiz(req: Request, res: Response) {
  const authReq = req as AuthenticatedRequest;
  const booking = await loadBooking(req.params.id);
  if (!booking) return res.status(404).json({ error: 'NOT_FOUND' });
  if (booking.educatorId !== authReq.user.userId) return res.status(403).json({ error: 'FORBIDDEN' });
  if (booking.status === 'cancelled' || booking.status === 'failed') return res.status(422).json({ error: 'BOOKING_NOT_ELIGIBLE' });
  if (booking.quizAttempt) return res.status(409).json({ error: 'QUIZ_LOCKED' });

  const parsed = validateQuizInput(req.body?.questions);
  if (!parsed.ok) return res.status(400).json({ error: 'VALIDATION_ERROR', message: parsed.error });

  const quiz = await prisma.sessionQuiz.upsert({
    where: { bookingId: booking.id },
    create: { bookingId: booking.id, questions: parsed.questions as any },
    update: { questions: parsed.questions as any },
  });
  return res.json({ quiz: { id: quiz.id, bookingId: quiz.bookingId, questions: quiz.questions } });
}

// GET /bookings/:id/quiz  (educator-owner => full; student-owner => stripped)
export async function getQuiz(req: Request, res: Response) {
  const authReq = req as AuthenticatedRequest;
  const booking = await loadBooking(req.params.id);
  if (!booking) return res.status(404).json({ error: 'NOT_FOUND' });

  const isEducator = booking.educatorId === authReq.user.userId;
  const isStudent = booking.studentId === authReq.user.userId;
  if (!isEducator && !isStudent) return res.status(403).json({ error: 'FORBIDDEN' });

  if (!booking.quiz) return res.json({ exists: false });
  const questions = booking.quiz.questions as unknown as QuizQuestion[];

  if (isEducator) {
    return res.json({ exists: true, locked: !!booking.quizAttempt, questions });
  }
  // student view — strip correctIndex
  const cfg = await prisma.platformConfig.findUnique({ where: { key: 'quiz_reward_coins' } });
  const rewardCoins = cfg ? parseInt(cfg.value as string) : 10;
  return res.json({
    exists: true,
    alreadyAttempted: !!booking.quizAttempt,
    rewardCoins,
    attempt: booking.quizAttempt
      ? { correctCount: booking.quizAttempt.correctCount, passed: booking.quizAttempt.passed, coinsAwarded: booking.quizAttempt.coinsAwarded }
      : null,
    questions: questions.map((q) => ({ question: q.question, options: q.options })),
  });
}

// POST /bookings/:id/quiz/attempt  (student-owner)
export async function submitAttempt(req: Request, res: Response) {
  const authReq = req as AuthenticatedRequest;
  const booking = await loadBooking(req.params.id);
  if (!booking) return res.status(404).json({ error: 'NOT_FOUND' });
  if (booking.studentId !== authReq.user.userId) return res.status(403).json({ error: 'FORBIDDEN' });
  if (booking.status !== 'completed') return res.status(422).json({ error: 'BOOKING_NOT_COMPLETED' });
  if (!booking.quiz) return res.status(404).json({ error: 'QUIZ_NOT_FOUND' });
  if (booking.quizAttempt) return res.status(409).json({ error: 'ALREADY_ATTEMPTED' });

  const answers = req.body?.answers;
  if (!Array.isArray(answers) || answers.length !== 5 || answers.some((a) => !Number.isInteger(a) || a < 0 || a > 3)) {
    return res.status(400).json({ error: 'VALIDATION_ERROR', message: 'answers must be 5 integers 0-3' });
  }

  const questions = booking.quiz.questions as unknown as QuizQuestion[];
  const { correctCount, passed, corrections } = gradeQuiz(questions, answers);

  const cfg = await prisma.platformConfig.findUnique({ where: { key: 'quiz_reward_coins' } });
  const rewardCoins = cfg ? parseInt(cfg.value as string) : 10;
  const coinsAwarded = passed ? rewardCoins : 0;

  try {
    await prisma.$transaction(async (tx) => {
      await tx.sessionQuizAttempt.create({
        data: { bookingId: booking.id, studentId: booking.studentId, answers: answers as any, correctCount, passed, coinsAwarded },
      });
      if (coinsAwarded > 0) {
        await tx.wallet.upsert({
          where: { studentId: booking.studentId },
          update: { balance: { increment: coinsAwarded } },
          create: { studentId: booking.studentId, balance: coinsAwarded },
        });
        await tx.walletTransaction.create({
          data: { studentId: booking.studentId, type: 'credit', amount: coinsAwarded, reference: `quiz_reward:${booking.id}` },
        });
      }
    });
  } catch (e: any) {
    // Unique bookingId violation => a concurrent attempt already landed
    if (e?.code === 'P2002') return res.status(409).json({ error: 'ALREADY_ATTEMPTED' });
    throw e;
  }

  return res.json({ correctCount, passed, coinsAwarded, corrections });
}
```

- [ ] **Step 2: Mount routes in `backend/src/routes/v1/booking.ts`**

Add the import and routes (place the specific `/quiz` routes BEFORE the catch-all `GET /:id`):

```ts
import { upsertQuiz, getQuiz, submitAttempt } from '../../controllers/quiz.controller';
// ... after the extend routes, before `router.get('/:id', ...)`:
router.post('/:id/quiz', authenticate, requireRole('educator'), upsertQuiz);
router.get('/:id/quiz', authenticate, getQuiz);
router.post('/:id/quiz/attempt', authenticate, requireRole('student'), submitAttempt);
```

- [ ] **Step 3: Verify it compiles + typechecks**

Run: `cd /Users/rrahul/Developer/Personal/Mole/code && pnpm --filter @mole/backend exec tsc --noEmit`
Expected: no NEW errors in `quiz.controller.ts` / `booking.ts` (pre-existing `prisma/seed.ts` rootDir error is unrelated — ignore it).

- [ ] **Step 4: Manual endpoint check** (backend running via `pnpm backend`, with a completed booking + educator/student tokens)

```bash
# educator sets a quiz (expect 200 { quiz })
curl -s -X POST localhost:3000/api/v1/bookings/<BID>/quiz -H "Authorization: Bearer <EDU>" -H 'Content-Type: application/json' \
  -d '{"questions":[{"question":"Q1","options":["a","b","c","d"],"correctIndex":0},{"question":"Q2","options":["a","b","c","d"],"correctIndex":1},{"question":"Q3","options":["a","b","c","d"],"correctIndex":2},{"question":"Q4","options":["a","b","c","d"],"correctIndex":3},{"question":"Q5","options":["a","b","c","d"],"correctIndex":0}]}'
# student GET (expect questions WITHOUT correctIndex, rewardCoins:10)
curl -s localhost:3000/api/v1/bookings/<BID>/quiz -H "Authorization: Bearer <STU>"
# student perfect attempt (expect passed:true, coinsAwarded:10)
curl -s -X POST localhost:3000/api/v1/bookings/<BID>/quiz/attempt -H "Authorization: Bearer <STU>" -H 'Content-Type: application/json' -d '{"answers":[0,1,2,3,0]}'
# second attempt (expect 409 ALREADY_ATTEMPTED)
curl -s -X POST localhost:3000/api/v1/bookings/<BID>/quiz/attempt -H "Authorization: Bearer <STU>" -H 'Content-Type: application/json' -d '{"answers":[0,1,2,3,0]}'
```
Expected: as annotated; verify the student's wallet balance increased by 10 and a `quiz_reward:<BID>` transaction exists.

- [ ] **Step 5: Commit**

```bash
git add backend/src/controllers/quiz.controller.ts backend/src/routes/v1/booking.ts
git commit -m "feat(quiz): booking-scoped quiz endpoints with grading + coin reward"
```

---

## Task 4: Web — student quiz on the review page (+ confetti)

**Files:**
- Modify: `web/package.json` (add `canvas-confetti` + `@types/canvas-confetti`)
- Create: `web/services/quiz.ts`, `web/components/quiz/StudentQuiz.tsx`
- Modify: `web/app/student/review/[bookingId]/page.tsx`

**Interfaces:**
- Consumes: `GET /bookings/:id/quiz`, `POST /bookings/:id/quiz/attempt` (Task 3).

- [ ] **Step 1: Add the dependency**

Run: `cd /Users/rrahul/Developer/Personal/Mole/code && pnpm --filter @mole/web add canvas-confetti && pnpm --filter @mole/web add -D @types/canvas-confetti`

- [ ] **Step 2: Create `web/services/quiz.ts`**

```ts
import { api } from './api';

export type StudentQuiz = {
  exists: boolean;
  alreadyAttempted?: boolean;
  rewardCoins?: number;
  attempt?: { correctCount: number; passed: boolean; coinsAwarded: number } | null;
  questions?: { question: string; options: string[] }[];
};
export type AttemptResult = { correctCount: number; passed: boolean; coinsAwarded: number; corrections: boolean[] };

export const getStudentQuiz = (bookingId: string) =>
  api.get<StudentQuiz>(`/bookings/${bookingId}/quiz`).then((r) => r.data);
export const submitQuizAttempt = (bookingId: string, answers: number[]) =>
  api.post<AttemptResult>(`/bookings/${bookingId}/quiz/attempt`, { answers }).then((r) => r.data);
```

- [ ] **Step 3: Create `web/components/quiz/StudentQuiz.tsx`**

```tsx
'use client';

import { useEffect, useState } from 'react';
import confetti from 'canvas-confetti';
import Box from '@mui/material/Box';
import Card from '@mui/material/Card';
import CardContent from '@mui/material/CardContent';
import Button from '@mui/material/Button';
import Typography from '@mui/material/Typography';
import Radio from '@mui/material/Radio';
import RadioGroup from '@mui/material/RadioGroup';
import FormControlLabel from '@mui/material/FormControlLabel';
import { C } from '@mole/shared';
import { getStudentQuiz, submitQuizAttempt, StudentQuiz as Q, AttemptResult } from '@/services/quiz';

export function StudentQuiz({ bookingId }: { bookingId: string }) {
  const [quiz, setQuiz] = useState<Q | null>(null);
  const [open, setOpen] = useState(false);
  const [answers, setAnswers] = useState<number[]>(Array(5).fill(-1));
  const [result, setResult] = useState<AttemptResult | null>(null);
  const [submitting, setSubmitting] = useState(false);

  useEffect(() => { getStudentQuiz(bookingId).then(setQuiz).catch(() => setQuiz({ exists: false })); }, [bookingId]);

  if (!quiz?.exists) return null;

  if (quiz.alreadyAttempted && !result) {
    const a = quiz.attempt!;
    return (
      <Card sx={{ mt: 3 }}>
        <CardContent>
          <Typography variant="subtitle1" fontWeight={700}>Quick test</Typography>
          <Typography variant="body2" color="text.secondary">
            You scored {a.correctCount}/5{a.passed ? ` · earned ${a.coinsAwarded} Mole Coins 🪙` : ''}.
          </Typography>
        </CardContent>
      </Card>
    );
  }

  async function submit() {
    setSubmitting(true);
    try {
      const r = await submitQuizAttempt(bookingId, answers);
      setResult(r);
      if (r.passed) confetti({ particleCount: 160, spread: 80, origin: { y: 0.6 } });
    } catch (e: any) {
      alert(e?.response?.data?.error === 'ALREADY_ATTEMPTED' ? 'Already attempted.' : 'Could not submit.');
    } finally { setSubmitting(false); }
  }

  if (result) {
    return (
      <Card sx={{ mt: 3, textAlign: 'center', bgcolor: result.passed ? C.indigoTint : undefined }}>
        <CardContent>
          {result.passed ? (
            <>
              <Typography variant="h5" fontWeight={800}>🎉 Perfect!</Typography>
              <Typography color="text.secondary">{result.coinsAwarded} Mole Coins added to your wallet.</Typography>
            </>
          ) : (
            <>
              <Typography variant="h6" fontWeight={700}>You scored {result.correctCount}/5</Typography>
              <Typography color="text.secondary">No coins this time — better luck next session!</Typography>
            </>
          )}
        </CardContent>
      </Card>
    );
  }

  if (!open) {
    return (
      <Button fullWidth variant="outlined" sx={{ mt: 3, borderRadius: 2 }} onClick={() => setOpen(true)}>
        Take quick test — earn {quiz.rewardCoins} Mole Coins
      </Button>
    );
  }

  return (
    <Card sx={{ mt: 3 }}>
      <CardContent>
        <Typography variant="subtitle1" fontWeight={700} mb={2}>Quick test</Typography>
        {quiz.questions!.map((q, qi) => (
          <Box key={qi} sx={{ mb: 2 }}>
            <Typography variant="body2" fontWeight={600} mb={0.5}>{qi + 1}. {q.question}</Typography>
            <RadioGroup value={answers[qi]} onChange={(e) => setAnswers((a) => a.map((v, i) => (i === qi ? Number(e.target.value) : v)))}>
              {q.options.map((opt, oi) => (
                <FormControlLabel key={oi} value={oi} control={<Radio size="small" />} label={opt} />
              ))}
            </RadioGroup>
          </Box>
        ))}
        <Button fullWidth variant="contained" sx={{ borderRadius: 2 }} disabled={submitting || answers.some((a) => a < 0)} onClick={submit}>
          {submitting ? 'Submitting…' : 'Submit answers'}
        </Button>
      </CardContent>
    </Card>
  );
}
```

- [ ] **Step 4: Mount it on the review page** — in `web/app/student/review/[bookingId]/page.tsx`, import and render `<StudentQuiz bookingId={bookingId} />` after the review `<Card>`.

```tsx
import { StudentQuiz } from '@/components/quiz/StudentQuiz';
// ...inside the returned <Box>, after the review Card:
<StudentQuiz bookingId={bookingId} />
```

- [ ] **Step 5: Verify**

Run: `cd /Users/rrahul/Developer/Personal/Mole/code && pnpm --filter @mole/web exec tsc --noEmit`
Expected: exit 0.
Manual: with a completed booking that has a quiz, open `/student/review/<id>` → button shows → answer all 5 correct → confetti + coins message; reload → shows prior score, no retake.

- [ ] **Step 6: Commit**

```bash
git add web/package.json web/pnpm-lock.yaml web/services/quiz.ts web/components/quiz/ web/app/student/review/
git commit -m "feat(web): student quiz on review page with confetti + coin reward"
```

---

## Task 5: Web — educator quiz setup from dashboard

**Files:**
- Create: `web/components/quiz/QuizSetupDialog.tsx`
- Modify: `web/app/educator/page.tsx` (schedule cards)

**Interfaces:**
- Consumes: `GET /bookings/:id/quiz` (educator view), `POST /bookings/:id/quiz` (Task 3); `getStudentQuiz` types reused where helpful.

- [ ] **Step 1: Create `web/components/quiz/QuizSetupDialog.tsx`**

```tsx
'use client';

import { useEffect, useState } from 'react';
import Dialog from '@mui/material/Dialog';
import DialogTitle from '@mui/material/DialogTitle';
import DialogContent from '@mui/material/DialogContent';
import DialogActions from '@mui/material/DialogActions';
import Button from '@mui/material/Button';
import TextField from '@mui/material/TextField';
import Box from '@mui/material/Box';
import Radio from '@mui/material/Radio';
import Typography from '@mui/material/Typography';
import { api } from '@/services/api';

type Q = { question: string; options: string[]; correctIndex: number };
const blank = (): Q => ({ question: '', options: ['', '', '', ''], correctIndex: 0 });

export function QuizSetupDialog({ bookingId, open, onClose }: { bookingId: string; open: boolean; onClose: () => void }) {
  const [qs, setQs] = useState<Q[]>(Array.from({ length: 5 }, blank));
  const [locked, setLocked] = useState(false);
  const [saving, setSaving] = useState(false);

  useEffect(() => {
    if (!open) return;
    api.get(`/bookings/${bookingId}/quiz`).then((r) => {
      if (r.data.exists) { setQs(r.data.questions); setLocked(!!r.data.locked); }
    }).catch(() => {});
  }, [open, bookingId]);

  const valid = qs.every((q) => q.question.trim() && q.options.every((o) => o.trim()));

  async function save() {
    setSaving(true);
    try {
      await api.post(`/bookings/${bookingId}/quiz`, { questions: qs });
      onClose();
    } catch (e: any) {
      alert(e?.response?.data?.error === 'QUIZ_LOCKED' ? 'Student already took the test.' : 'Could not save.');
    } finally { setSaving(false); }
  }

  return (
    <Dialog open={open} onClose={onClose} fullWidth maxWidth="sm">
      <DialogTitle>Quick test (5 questions)</DialogTitle>
      <DialogContent>
        {locked && <Typography color="warning.main" variant="body2" mb={1}>Locked — the student already took this test.</Typography>}
        {qs.map((q, qi) => (
          <Box key={qi} sx={{ mb: 2 }}>
            <TextField fullWidth size="small" label={`Question ${qi + 1}`} value={q.question} disabled={locked}
              onChange={(e) => setQs((p) => p.map((x, i) => (i === qi ? { ...x, question: e.target.value } : x)))} sx={{ mb: 1 }} />
            {q.options.map((opt, oi) => (
              <Box key={oi} sx={{ display: 'flex', alignItems: 'center', gap: 1 }}>
                <Radio size="small" checked={q.correctIndex === oi} disabled={locked}
                  onChange={() => setQs((p) => p.map((x, i) => (i === qi ? { ...x, correctIndex: oi } : x)))} />
                <TextField fullWidth size="small" placeholder={`Option ${oi + 1}`} value={opt} disabled={locked}
                  onChange={(e) => setQs((p) => p.map((x, i) => (i === qi ? { ...x, options: x.options.map((o, k) => (k === oi ? e.target.value : o)) } : x)))} />
              </Box>
            ))}
          </Box>
        ))}
      </DialogContent>
      <DialogActions>
        <Button onClick={onClose}>Cancel</Button>
        <Button variant="contained" disabled={locked || saving || !valid} onClick={save}>{saving ? 'Saving…' : 'Save test'}</Button>
      </DialogActions>
    </Dialog>
  );
}
```

- [ ] **Step 2: Wire a "Set up quick test" button into the dashboard schedule cards** — in `web/app/educator/page.tsx`, add state `const [quizFor, setQuizFor] = useState<string | null>(null);`, render a small `Button` on each schedule item (`onClick={() => setQuizFor(booking.id)}`), and mount once: `{quizFor && <QuizSetupDialog bookingId={quizFor} open onClose={() => setQuizFor(null)} />}`. Import `QuizSetupDialog`.

- [ ] **Step 3: Verify**

Run: `cd /Users/rrahul/Developer/Personal/Mole/code && pnpm --filter @mole/web exec tsc --noEmit`
Expected: exit 0.
Manual: educator dashboard → schedule card → "Set up quick test" → fill 5 Qs → Save → reopen shows saved values.

- [ ] **Step 4: Commit**

```bash
git add web/components/quiz/QuizSetupDialog.tsx web/app/educator/page.tsx
git commit -m "feat(web): educator quiz setup dialog from dashboard schedule"
```

---

## Task 6: Mobile — student quiz screen (+ confetti)

**Files:**
- Modify: `mobile/package.json` (add `react-native-confetti-cannon`)
- Create: `mobile/services/quiz.ts`, `mobile/app/(student)/quiz/[bookingId].tsx`
- Modify: `mobile/app/(student)/review/[bookingId].tsx`

**Interfaces:** same endpoints as Task 3.

- [ ] **Step 1: Add the dependency**

Run: `cd mobile && npx expo install react-native-confetti-cannon`

- [ ] **Step 2: Create `mobile/services/quiz.ts`**

```ts
import { api } from './api';

export type StudentQuiz = {
  exists: boolean;
  alreadyAttempted?: boolean;
  rewardCoins?: number;
  attempt?: { correctCount: number; passed: boolean; coinsAwarded: number } | null;
  questions?: { question: string; options: string[] }[];
};
export type AttemptResult = { correctCount: number; passed: boolean; coinsAwarded: number; corrections: boolean[] };

export const getStudentQuiz = (bookingId: string) => api.get<StudentQuiz>(`/bookings/${bookingId}/quiz`).then((r) => r.data);
export const submitQuizAttempt = (bookingId: string, answers: number[]) =>
  api.post<AttemptResult>(`/bookings/${bookingId}/quiz/attempt`, { answers }).then((r) => r.data);
```

- [ ] **Step 3: Create `mobile/app/(student)/quiz/[bookingId].tsx`**

```tsx
import { useEffect, useRef, useState } from 'react';
import { View, Text, ScrollView, TouchableOpacity, StyleSheet, ActivityIndicator } from 'react-native';
import { useLocalSearchParams, useRouter } from 'expo-router';
import ConfettiCannon from 'react-native-confetti-cannon';
import { C, S } from '@mole/shared';
import { getStudentQuiz, submitQuizAttempt, StudentQuiz, AttemptResult } from '../../../services/quiz';

export default function QuizScreen() {
  const { bookingId } = useLocalSearchParams<{ bookingId: string }>();
  const router = useRouter();
  const [quiz, setQuiz] = useState<StudentQuiz | null>(null);
  const [answers, setAnswers] = useState<number[]>(Array(5).fill(-1));
  const [result, setResult] = useState<AttemptResult | null>(null);
  const [submitting, setSubmitting] = useState(false);

  useEffect(() => { getStudentQuiz(bookingId).then(setQuiz).catch(() => setQuiz({ exists: false })); }, [bookingId]);

  if (!quiz) return <View style={styles.center}><ActivityIndicator color={C.primary} /></View>;
  if (!quiz.exists || quiz.alreadyAttempted) {
    return <View style={styles.center}><Text style={styles.msg}>{quiz.alreadyAttempted ? 'You already took this test.' : 'No test for this session.'}</Text></View>;
  }

  async function submit() {
    setSubmitting(true);
    try { setResult(await submitQuizAttempt(bookingId, answers)); }
    catch { /* show inline */ } finally { setSubmitting(false); }
  }

  if (result) {
    return (
      <View style={styles.center}>
        {result.passed && <ConfettiCannon count={160} origin={{ x: 180, y: 0 }} fadeOut />}
        <Text style={styles.big}>{result.passed ? '🎉 Perfect!' : `You scored ${result.correctCount}/5`}</Text>
        <Text style={styles.msg}>{result.passed ? `${result.coinsAwarded} Mole Coins added to your wallet.` : 'No coins this time.'}</Text>
        <TouchableOpacity style={styles.btn} onPress={() => router.back()}><Text style={styles.btnTxt}>Done</Text></TouchableOpacity>
      </View>
    );
  }

  return (
    <ScrollView contentContainerStyle={{ padding: S.containerPadding }}>
      <Text style={styles.title}>Quick test — earn {quiz.rewardCoins} Mole Coins</Text>
      {quiz.questions!.map((q, qi) => (
        <View key={qi} style={{ marginBottom: 20 }}>
          <Text style={styles.q}>{qi + 1}. {q.question}</Text>
          {q.options.map((opt, oi) => {
            const sel = answers[qi] === oi;
            return (
              <TouchableOpacity key={oi} style={[styles.opt, sel && styles.optSel]} onPress={() => setAnswers((a) => a.map((v, i) => (i === qi ? oi : v)))}>
                <Text style={[styles.optTxt, sel && styles.optTxtSel]}>{opt}</Text>
              </TouchableOpacity>
            );
          })}
        </View>
      ))}
      <TouchableOpacity style={[styles.btn, (submitting || answers.some((a) => a < 0)) && { opacity: 0.5 }]} disabled={submitting || answers.some((a) => a < 0)} onPress={submit}>
        <Text style={styles.btnTxt}>{submitting ? 'Submitting…' : 'Submit answers'}</Text>
      </TouchableOpacity>
    </ScrollView>
  );
}

const styles = StyleSheet.create({
  center: { flex: 1, justifyContent: 'center', alignItems: 'center', padding: S.containerPadding, gap: 12, backgroundColor: C.background },
  title: { fontSize: 18, fontWeight: '700', color: C.textPrimary, marginBottom: 16 },
  q: { fontSize: 15, fontWeight: '600', color: C.textPrimary, marginBottom: 8 },
  opt: { borderWidth: 1, borderColor: C.borderColor, borderRadius: S.radiusMd, padding: 12, marginBottom: 8, minHeight: 48, justifyContent: 'center' },
  optSel: { borderColor: C.primary, backgroundColor: C.indigoTint },
  optTxt: { fontSize: 14, color: C.textPrimary },
  optTxtSel: { color: C.primary, fontWeight: '600' },
  btn: { backgroundColor: C.primary, borderRadius: S.radiusXl, paddingVertical: 14, alignItems: 'center', minHeight: 48, justifyContent: 'center' },
  btnTxt: { color: C.onPrimary, fontWeight: '700', fontSize: 15 },
  big: { fontSize: 24, fontWeight: '800', color: C.textPrimary },
  msg: { fontSize: 14, color: C.textSecondary, textAlign: 'center' },
});
```

- [ ] **Step 4: Add a button on the review screen** — in `mobile/app/(student)/review/[bookingId].tsx`, on mount also `getStudentQuiz(bookingId)`; if `exists && !alreadyAttempted`, render a `TouchableOpacity` "Take quick test — earn N Mole Coins" that does `router.push(\`/(student)/quiz/${bookingId}\`)`.

- [ ] **Step 5: Verify**

Run: `cd /Users/rrahul/Developer/Personal/Mole/code && pnpm --filter @mole/mobile exec tsc --noEmit`
Expected: exit 0.
Manual (Expo): completed booking with a quiz → review screen shows the button → quiz screen → all correct → confetti + coins.

- [ ] **Step 6: Commit**

```bash
git add mobile/package.json mobile/services/quiz.ts mobile/app/\(student\)/quiz/ mobile/app/\(student\)/review/
git commit -m "feat(mobile): student quiz screen with confetti + coin reward"
```

---

## Task 7: Mobile — educator quiz setup from dashboard

**Files:**
- Create: `mobile/app/(educator)/quiz-setup/[bookingId].tsx`
- Modify: `mobile/app/(educator)/index.tsx` (schedule cards)

**Interfaces:** `GET /bookings/:id/quiz` (educator), `POST /bookings/:id/quiz`.

- [ ] **Step 1: Create `mobile/app/(educator)/quiz-setup/[bookingId].tsx`**

```tsx
import { useEffect, useState } from 'react';
import { View, Text, ScrollView, TextInput, TouchableOpacity, StyleSheet, Alert } from 'react-native';
import { useLocalSearchParams, useRouter } from 'expo-router';
import { api } from '../../../services/api';
import { C, S } from '@mole/shared';

type Q = { question: string; options: string[]; correctIndex: number };
const blank = (): Q => ({ question: '', options: ['', '', '', ''], correctIndex: 0 });

export default function QuizSetup() {
  const { bookingId } = useLocalSearchParams<{ bookingId: string }>();
  const router = useRouter();
  const [qs, setQs] = useState<Q[]>(Array.from({ length: 5 }, blank));
  const [locked, setLocked] = useState(false);
  const [saving, setSaving] = useState(false);

  useEffect(() => {
    api.get(`/bookings/${bookingId}/quiz`).then((r) => {
      if (r.data.exists) { setQs(r.data.questions); setLocked(!!r.data.locked); }
    }).catch(() => {});
  }, [bookingId]);

  const valid = qs.every((q) => q.question.trim() && q.options.every((o) => o.trim()));

  async function save() {
    setSaving(true);
    try { await api.post(`/bookings/${bookingId}/quiz`, { questions: qs }); router.back(); }
    catch (e: any) { Alert.alert('Error', e?.response?.data?.error === 'QUIZ_LOCKED' ? 'Student already took the test.' : 'Could not save.'); }
    finally { setSaving(false); }
  }

  return (
    <ScrollView contentContainerStyle={{ padding: S.containerPadding }}>
      <Text style={styles.title}>Quick test (5 questions)</Text>
      {locked && <Text style={styles.locked}>Locked — the student already took this test.</Text>}
      {qs.map((q, qi) => (
        <View key={qi} style={{ marginBottom: 16 }}>
          <TextInput style={styles.input} editable={!locked} placeholder={`Question ${qi + 1}`} value={q.question}
            onChangeText={(t) => setQs((p) => p.map((x, i) => (i === qi ? { ...x, question: t } : x)))} />
          {q.options.map((opt, oi) => (
            <View key={oi} style={styles.optRow}>
              <TouchableOpacity disabled={locked} onPress={() => setQs((p) => p.map((x, i) => (i === qi ? { ...x, correctIndex: oi } : x)))}
                style={[styles.radio, q.correctIndex === oi && styles.radioOn]} />
              <TextInput style={[styles.input, { flex: 1 }]} editable={!locked} placeholder={`Option ${oi + 1}`} value={opt}
                onChangeText={(t) => setQs((p) => p.map((x, i) => (i === qi ? { ...x, options: x.options.map((o, k) => (k === oi ? t : o)) } : x)))} />
            </View>
          ))}
        </View>
      ))}
      <TouchableOpacity style={[styles.btn, (locked || saving || !valid) && { opacity: 0.5 }]} disabled={locked || saving || !valid} onPress={save}>
        <Text style={styles.btnTxt}>{saving ? 'Saving…' : 'Save test'}</Text>
      </TouchableOpacity>
    </ScrollView>
  );
}

const styles = StyleSheet.create({
  title: { fontSize: 18, fontWeight: '700', color: C.textPrimary, marginBottom: 12 },
  locked: { color: C.warning, marginBottom: 8 },
  input: { borderWidth: 1, borderColor: C.borderColor, borderRadius: S.radiusMd, padding: 10, marginBottom: 8, color: C.textPrimary },
  optRow: { flexDirection: 'row', alignItems: 'center', gap: 8 },
  radio: { width: 22, height: 22, borderRadius: 11, borderWidth: 2, borderColor: C.outlineVariant },
  radioOn: { borderColor: C.primary, backgroundColor: C.primary },
  btn: { backgroundColor: C.primary, borderRadius: S.radiusXl, paddingVertical: 14, alignItems: 'center', minHeight: 48, justifyContent: 'center' },
  btnTxt: { color: C.onPrimary, fontWeight: '700', fontSize: 15 },
});
```

- [ ] **Step 2: Add a "Set up quick test" action on the dashboard schedule cards** — in `mobile/app/(educator)/index.tsx`, on each schedule row add a `TouchableOpacity` "Set up quick test" → `router.push(\`/(educator)/quiz-setup/${s.id}\`)`.

- [ ] **Step 3: Verify**

Run: `cd /Users/rrahul/Developer/Personal/Mole/code && pnpm --filter @mole/mobile exec tsc --noEmit`
Expected: exit 0.
Manual (Expo): educator dashboard → schedule card → "Set up quick test" → fill + Save → reopen shows saved values.

- [ ] **Step 4: Commit**

```bash
git add mobile/app/\(educator\)/quiz-setup/ mobile/app/\(educator\)/index.tsx
git commit -m "feat(mobile): educator quiz setup screen from dashboard"
```

---

## Task 8: Test-case docs (QUIZ area)

**Files:**
- Modify: `docs/test-cases/backend-api.md`, `docs/test-cases/web.md`, `docs/test-cases/mobile.md`

- [ ] **Step 1: Append a `## Quiz (QUIZ)` section** to `backend-api.md` with cases (following the file's existing format) covering: validation (bad count/options/index), educator-owner guard, `QUIZ_LOCKED` after attempt, student GET strips `correctIndex`, attempt requires `completed`, 5/5 credits `quiz_reward_coins` + creates `quiz_reward:<id>` transaction, 4/5 credits nothing, `ALREADY_ATTEMPTED` on second attempt, double-credit prevented by unique attempt.

- [ ] **Step 2: Append `## Quiz (QUIZ)`** to `web.md` and `mobile.md` covering: educator setup validation + locked state; student button visibility (exists / attempted / none); pass path (confetti + coins) and fail path (score, no coins); attempt-consumed state.

- [ ] **Step 3: Commit**

```bash
git add docs/test-cases/
git commit -m "docs(test-cases): add QUIZ area cases for session quiz feature"
```

---

## Self-Review

**Spec coverage:** data model (T1), config default (T1/T3), API + grading + reward transaction + `correctIndex` stripping (T2/T3), educator setup web+mobile (T5/T7), student quiz web+mobile + confetti + coins (T4/T6), once-per-session unique constraint + double-credit guard (T1/T3), edit-lock after attempt (T3/T5/T7), tests + docs (T2/T8). All spec sections mapped.

**Placeholder scan:** No TBD/TODO; every code step has complete code. The only conditional is Task 1 Step 4's note to confirm `platform_config` column names before writing the seed INSERT — the config helper's default (10) makes the INSERT non-critical.

**Type consistency:** `QuizQuestion { question, options, correctIndex }`, attempt result `{ correctCount, passed, coinsAwarded, corrections }`, and the student GET shape `{ exists, alreadyAttempted, rewardCoins, attempt, questions:[{question,options}] }` are identical across backend (T2/T3), web (T4/T5), and mobile (T6/T7). Endpoints `/bookings/:id/quiz` (POST/GET) and `/bookings/:id/quiz/attempt` (POST) consistent throughout.
