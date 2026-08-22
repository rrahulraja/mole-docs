# Epic D — Languages + Student Location (Design Spec)

**Date:** 2026-08-04
**Status:** Approved (design)
**Part of:** Booking/marketplace revamp (Epics D → C → B → A). D is first: lowest
risk, unblocks Epic C's international pricing (needs student country).

## Goal

Capture the **languages** an educator and a student speak (onboarding + editable in
profile), and silently capture a **student's location** at onboarding. Use languages
to softly boost educator suggestions. Store country so Epic C can charge non-India
students international rates.

## Scope

In scope:
- `User.languages` (both roles) + curated language list in `@mole/shared`.
- Student location fields (`latitude`, `longitude`, `country`, `city`).
- Language multi-select on educator onboarding, student onboarding, and both
  edit-profile screens — **web + mobile parity**.
- Silent GPS + reverse-geocode at student onboarding (no location UI).
- Soft language ranking boost in educator discovery.

Out of scope (later epics):
- International pricing itself (Epic C consumes `country`).
- Proximity/distance matching from coords (fields stored now, unused until later).
- Educator location.
- Admin-editable language list (MVP uses the shared constant; extend later).

## Data model

`backend/prisma/schema.prisma`:

```prisma
model User {
  // ...existing...
  languages String[] @default([]) @map("languages")
}

model Student {
  // ...existing...
  latitude          Float?    @map("latitude")
  longitude         Float?    @map("longitude")
  country           String?   @map("country")            // ISO-3166 alpha-2, e.g. "IN"
  city              String?   @map("city")
  locationUpdatedAt DateTime? @map("location_updated_at")
}
```

Rationale:
- `languages` on **User** (not per role) — one field, both roles have a `User`, and
  matching joins `educator.user.languages`. No duplicate columns.
- Store **both** raw coords **and** derived `country`/`city`. `country` is read
  directly by Epic C (India-vs-not); coords avoid re-geocoding and unlock future
  proximity features.
- Nullable throughout — location is best-effort (permission may be denied).

Migration + `pnpm generate:prisma` before typecheck (generated types gate `tsc`).

## Shared language constant

`shared/src/constants/languages.ts`, re-exported from `shared/src/index.ts`:

```ts
export interface LanguageOption { code: string; label: string }

// code = lowercase ISO-639-1 where available; label = English name.
export const LANGUAGES: LanguageOption[] = [
  { code: 'en', label: 'English' },
  { code: 'hi', label: 'Hindi' },
  { code: 'bn', label: 'Bengali' },
  { code: 'ta', label: 'Tamil' },
  { code: 'te', label: 'Telugu' },
  { code: 'mr', label: 'Marathi' },
  { code: 'gu', label: 'Gujarati' },
  { code: 'kn', label: 'Kannada' },
  { code: 'ml', label: 'Malayalam' },
  { code: 'pa', label: 'Punjabi' },
  { code: 'ur', label: 'Urdu' },
  { code: 'or', label: 'Odia' },
  { code: 'as', label: 'Assamese' },
];

export const LANGUAGE_CODES = LANGUAGES.map((l) => l.code);
```

Stored value = array of `code` strings (e.g. `["en","hi"]`). Frontends render `label`,
persist `code`. Backend validates each code ∈ `LANGUAGE_CODES`.

## Backend

**Language validation helper** — reuse `LANGUAGE_CODES` from `@mole/shared` in Zod:
`z.array(z.string()).refine(a => a.every(c => LANGUAGE_CODES.includes(c)))`.

**Student onboarding** (`backend/src/controllers/sudent.controller.ts` `profileSetup`):
accept optional `languages: string[]`; write to `user.languages`.

**Educator onboarding** (`backend/src/controllers/educator.controller.ts`): accept
`languages` in the onboarding-submit handler; write to `user.languages`.

**Profile update** — student (`updateStudentProfile` in `sudent.controller.ts`) and the
educator profile-update handler: accept optional `languages`; update `user.languages`.

**`getStudentMe` / educator `me`**: include `languages` in the response so profile
screens can prefill.

**New endpoint — silent location upload**
`POST /api/v1/student/location` (route in `backend/src/routes/v1/student.ts`, handler in
`sudent.controller.ts`):

```ts
// body: { latitude:number, longitude:number, country?:string, city?:string }
export async function updateStudentLocation(req, res) {
  // auth: student. Zod-validate. Update student row + locationUpdatedAt = now().
  // Returns 204. Called at onboarding AND opportunistically on later app opens.
}
```

## Matching — soft language boost

`backend/src/controllers/discover.controller.ts`. Add a helper alongside
`getStudentBoardFilter`:

```ts
async function getStudentLanguages(userId: string): Promise<string[]> {
  const u = await prisma.user.findUnique({ where: { id: userId }, select: { languages: true } });
  return u?.languages ?? [];
}
```

Add `user: { select: { name: true, languages: true } }` to `EDUCATOR_SELECT` (name is
already selected — extend it to also pull `languages`).

After fetching the candidate list (in `getSuggestedEducators`, `searchEducators`,
`getLiveEducators`), reorder in JS **before** `formatEducators`:

```ts
function rankByLanguage<T extends { user: { languages: string[] } }>(
  list: T[], studentLangs: string[],
): T[] {
  if (studentLangs.length === 0) return list;               // no student langs → unchanged
  const set = new Set(studentLangs);
  const matched = list.filter((e) => e.user.languages?.some((l) => set.has(l)));
  const rest    = list.filter((e) => !e.user.languages?.some((l) => set.has(l)));
  return [...matched, ...rest];                              // each side already rating-sorted by the query
}
```

Soft boost only — nobody is filtered out. `formatEducator` also gains `languages` in its
output so cards/profile can show them.

## Frontend

### Curated multi-select (shared UX, per-app impl)
Chips/checkboxes over `LANGUAGES`, multi-pick, min 1 required at onboarding. Web uses MUI,
mobile uses `StyleSheet.create` + `C`/`S` tokens, matching each app's conventions.

- **Educator onboarding** — mobile `mobile/app/(educator)/onboarding.tsx`, web
  `web/app/educator/onboarding`.
- **Student onboarding** — mobile `mobile/app/(student)/onboarding.tsx`, web
  `web/app/student/onboarding`.
- **Edit profile** — mobile `mobile/app/(educator)/edit-profile.tsx` +
  `mobile/app/(student)/edit-profile.tsx`; web `web/app/educator/edit-profile` +
  student profile edit. Prefill from `me`, save via profile-update.

### Silent location (student onboarding only)
At the **end** of student onboarding, after profile submit:

- **Mobile** (`expo-location`, already an Expo module):
  ```ts
  const { status } = await Location.requestForegroundPermissionsAsync();
  if (status !== 'granted') return;                          // silently skip
  const pos = await Location.getCurrentPositionAsync({});
  const geo = (await Location.reverseGeocodeAsync(pos.coords))[0];
  await api.post('/student/location', {
    latitude: pos.coords.latitude, longitude: pos.coords.longitude,
    country: geo?.isoCountryCode ?? undefined, city: geo?.city ?? undefined,
  });
  ```
- **Web** (`navigator.geolocation`): get coords; browser has no native reverse-geocode —
  send coords with `country`/`city` omitted (backend stores coords; country stays null
  until derived later, or a lightweight lookup is added in Epic C). Permission denied →
  skip silently.

No map, no country picker, no toast — fully silent. The only visible artifact is the OS
permission dialog (unavoidable).

## Error handling
- Location permission denied / GPS error / geocode empty → **skip**, store nothing. Never
  block onboarding. Student with no `country` is treated as domestic (India) downstream.
- Invalid language code → 400 from backend Zod; frontends only ever send list codes.
- `POST /student/location` failure → swallow on client (best-effort); onboarding proceeds.

## Testing
- **Backend (jest):** `LANGUAGE_CODES` validation rejects unknown codes; `profileSetup` +
  educator onboarding persist `languages`; `updateStudentLocation` writes coords + country
  + `locationUpdatedAt`; `rankByLanguage` orders matched-first and is a no-op when student
  has no languages or list is empty.
- **Manual:** onboard educator (mobile) with langs → onboard student sharing a lang →
  student's suggestions show that educator first. Deny location permission → onboarding
  completes, no crash.

## Rollout order (issues/PRs, each web+mobile parity where UI)
1. Shared `LANGUAGES` constant + schema migration + Prisma generate.
2. Backend: onboarding/profile accept `languages`; `me` returns them; `rankByLanguage`.
3. Backend: `POST /student/location`.
4. Frontend: language multi-select (educator + student onboarding + edit-profile), web + mobile.
5. Frontend: silent location capture at student onboarding, web + mobile.

Each unit = GitHub issue → branch → PR → merge → Project board #2 (Done).
