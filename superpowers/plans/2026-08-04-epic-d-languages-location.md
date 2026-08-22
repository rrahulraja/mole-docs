# Epic D — Languages + Student Location Implementation Plan

> **For agentic workers:** implement task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Capture educator/student languages (onboarding + profile) and silent student location, and softly boost educator suggestions by shared language.

**Architecture:** `languages String[]` on `User` (both roles); location fields on `Student`. Curated `LANGUAGES` constant in `@mole/shared`. Backend accepts languages in onboarding/profile + a silent `POST /student/location`. Discovery reorders matched-language educators first (soft boost, never filters). Language multi-select UI + silent GPS capture on web + mobile.

**Tech Stack:** Express + Prisma + Postgres · Next.js 14 + MUI v5 · Expo/React Native · `@mole/shared` · Zod · SWR · expo-location.

## Global Constraints

- **Web+mobile parity:** every student/educator UI change ships on BOTH web and mobile.
- **Shared = source of truth:** language list lives in `@mole/shared`, imported by all frontends + backend validation.
- **Prisma flow:** after `schema.prisma` edits → migrate + `pnpm generate:prisma` before typecheck.
- **VCS:** commit as `Rrahul Raja <rahulraja.ngp@gmail.com>`; strip any Claude trailers. Each unit = GitHub issue → branch → PR → merge → Project board #2 (Done).
- **Never commit** `.env*`, `admin/vite.config.ts` (local deploy-mod), `docs/deployment/vps-deploy.md`.
- **Language storage:** array of ISO-639-1 `code` strings (`["en","hi"]`); validate each ∈ `LANGUAGE_CODES`.
- **Country:** ISO-3166 alpha-2 (`"IN"`). Location is best-effort; permission denied = store nothing, never block onboarding.

---

## Unit 1 — Shared constant + schema migration

**Files:**
- Create: `shared/src/constants/languages.ts`
- Modify: `shared/src/index.ts`
- Modify: `backend/prisma/schema.prisma` (User + Student)
- Migration: `backend/prisma/migrations/*`

**Produces:** `LANGUAGES`, `LANGUAGE_CODES`, `LanguageOption` from `@mole/shared`; `User.languages`, `Student.{latitude,longitude,country,city,locationUpdatedAt}` columns.

- [ ] **Step 1:** Create `shared/src/constants/languages.ts` with `LanguageOption`, `LANGUAGES` (en,hi,bn,ta,te,mr,gu,kn,ml,pa,ur,or,as), `LANGUAGE_CODES`.
- [ ] **Step 2:** Add `export * from './constants/languages';` to `shared/src/index.ts`.
- [ ] **Step 3:** Add `languages String[] @default([]) @map("languages")` to `User`; add location fields to `Student`.
- [ ] **Step 4:** `cd backend && pnpm db:migrate:dev --name epic_d_languages_location && pnpm generate:prisma`.
- [ ] **Step 5:** `pnpm --filter @mole/backend typecheck`. Expected: PASS.
- [ ] **Step 6:** Commit `feat(shared,db): languages constant + user.languages + student location fields`.

## Unit 2 — Backend: languages in onboarding/profile + soft matching

**Files:**
- Modify: `backend/src/controllers/sudent.controller.ts` (profileSetup, updateStudentProfile, getStudentMe schemas + writes)
- Modify: `backend/src/controllers/educator.controller.ts` (onboarding submit + profile update + me)
- Modify: `backend/src/controllers/discover.controller.ts` (EDUCATOR_SELECT, getStudentLanguages, rankByLanguage, formatEducator output)
- Test: `backend/src/**/__tests__` (matching existing jest pattern)

**Consumes:** `LANGUAGE_CODES` (Unit 1). **Produces:** `languages` persisted + returned in `me`; suggestions language-boosted.

- [ ] **Step 1:** Write failing jest tests — `rankByLanguage` puts matched first + no-op on empty; language Zod rejects unknown code.
- [ ] **Step 2:** Run tests, verify FAIL.
- [ ] **Step 3:** Add `languagesSchema = z.array(z.string()).refine(a => a.every(c => LANGUAGE_CODES.includes(c)))`; wire into student+educator onboarding + profile-update; write to `user.languages`; include `languages` in both `me` responses.
- [ ] **Step 4:** In `discover.controller`: extend `EDUCATOR_SELECT.user.select` with `languages`; add `getStudentLanguages` + `rankByLanguage`; apply before `formatEducators` in suggested/search/live; add `languages` to `formatEducator` output.
- [ ] **Step 5:** Run tests, verify PASS + `pnpm --filter @mole/backend typecheck`.
- [ ] **Step 6:** Commit `feat(backend): accept languages in onboarding/profile + soft language boost in discovery`.

## Unit 3 — Backend: silent location endpoint

**Files:**
- Modify: `backend/src/controllers/sudent.controller.ts` (updateStudentLocation)
- Modify: `backend/src/routes/v1/student.ts`
- Test: backend jest

**Produces:** `POST /api/v1/student/location`.

- [ ] **Step 1:** Failing test — posting coords+country updates student row + `locationUpdatedAt`.
- [ ] **Step 2:** Run, verify FAIL.
- [ ] **Step 3:** Add `updateStudentLocation` (Zod: latitude/longitude required numbers, country/city optional strings) → update student, set `locationUpdatedAt = new Date()`, return 204. Mount authenticated route `POST /location`.
- [ ] **Step 4:** Run test PASS + typecheck.
- [ ] **Step 5:** Commit `feat(backend): POST /student/location silent coords+country upload`.

## Unit 4 — Frontend: language multi-select (web + mobile)

**Files:**
- Mobile: `mobile/app/(educator)/onboarding.tsx`, `mobile/app/(student)/onboarding.tsx`, `mobile/app/(educator)/edit-profile.tsx`, `mobile/app/(student)/edit-profile.tsx`
- Web: `web/app/educator/onboarding/*`, `web/app/student/onboarding/*`, `web/app/educator/edit-profile/*`, student profile edit
- Reusable: a `LanguageSelect` component per app (mobile `components/`, web `components/`)

**Consumes:** `LANGUAGES` (Unit 1), backend languages fields (Unit 2).

- [ ] **Step 1:** Mobile `LanguageSelect` (chips over `LANGUAGES`, multi-toggle, `C`/`S` tokens); web `LanguageSelect` (MUI chips/autocomplete).
- [ ] **Step 2:** Add to educator + student onboarding (min 1 required, submit `languages` codes) — mobile + web.
- [ ] **Step 3:** Add to educator + student edit-profile, prefill from `me`, save via profile-update — mobile + web.
- [ ] **Step 4:** Typecheck web (`next dev` unaffected by known 18/19 overload noise) + mobile tsc.
- [ ] **Step 5:** Commit `feat(web,mobile): language multi-select in onboarding + profile`.

## Unit 5 — Frontend: silent location capture at student onboarding

**Files:**
- Mobile: `mobile/app/(student)/onboarding.tsx` (+ `expo-location` in `mobile/app.json` plugins if needed)
- Web: `web/app/student/onboarding/*`

**Consumes:** `POST /student/location` (Unit 3).

- [ ] **Step 1:** Mobile — after profile submit: request `expo-location` foreground permission; if granted, get coords + `reverseGeocodeAsync` → POST `/student/location` with lat/lng/country/city. Denied/error → skip silently.
- [ ] **Step 2:** Add expo-location plugin + `NSLocationWhenInUseUsageDescription` / permission strings to `mobile/app.json`.
- [ ] **Step 3:** Web — after profile submit: `navigator.geolocation.getCurrentPosition` → POST coords (country/city omitted). Denied → skip.
- [ ] **Step 4:** Mobile tsc + web typecheck.
- [ ] **Step 5:** Commit `feat(web,mobile): silent GPS location capture at student onboarding`.

---

## Self-Review

- **Coverage:** spec's schema (U1), shared constant (U1), backend onboarding/profile/me (U2), matching boost (U2), location endpoint (U3), language UI (U4), silent capture (U5) — all mapped.
- **Types:** `LANGUAGE_CODES`/`LANGUAGES`/`LanguageOption` consistent U1→U2→U4; `rankByLanguage` signature matches spec.
- **Parity:** U4 + U5 each list web + mobile files.
