# Google Play — release plan: personal account → internal testing → org account

Plan for Mole's **first Android release** (`com.mole.app`):

1. Register a **personal** Play developer account now.
2. Ship builds to **internal testing** from it.
3. When the company + D-U-N-S exist, create an **organization** account and **transfer** the app.
4. Launch to **production** from the organization account.

Also covers **versioning** and the **v1.0.0 release notes**.

> Account creation, store listing and policy forms are covered in
> [`google-play-console-setup.md`](./google-play-console-setup.md). This doc covers the plan above.
> Google changes Console menus and policy rules often. Re-check anything marked ⚠️ in the
> Play Console Help Center when you get to that step.

---

## Why this order works

- **Internal testing has no 12-testers/14-days rule.** That rule only applies when a *personal*
  account asks for **production** access. If you stay on internal testing until the transfer, you
  never hit it. Organization accounts don't have the rule. ⚠️ After the transfer, confirm in the
  org account's dashboard that production isn't gated.
- **The package name is permanent.** After the first upload, `com.mole.app` belongs to that Play
  account, and deleting the app doesn't free the name. The only way to get it into the org account
  is an **app transfer**. You can't recreate it there.
- **Users keep their install.** A transferred app keeps its package, signing key, reviews and
  install base. Testers get updates as normal and don't need to reinstall.

---

## Phase A — Personal account + first internal build

### A1. Create the personal account
Follow [`google-play-console-setup.md`](./google-play-console-setup.md) Phases 0–1 with
**"Yourself"**:
- Use a **dedicated Google account**, not your main inbox. The transfer is easier when the source
  account only holds Mole.
- Pay the $25 fee and complete ID + device verification.
- **Save the payment receipt** (Google Payments → Activity). Transfers ask for registration
  transaction IDs.

### A2. Create the app
**Home → Create app** → name `MoLe`, language English, **App**, **Free**, accept declarations.
See setup doc Phase 3.

### A3. Create the upload keystore (one time, keep it forever)
Play **rejects debug-signed bundles**. Today `android/app/build.gradle` signs release builds with
the debug keystore, so make a real upload key first.

```bash
mkdir -p ~/keys && cd ~/keys
keytool -genkeypair -v -storetype PKCS12 \
  -keystore mole-upload.jks -alias mole-upload \
  -keyalg RSA -keysize 2048 -validity 10000
```

- Store `mole-upload.jks` and both passwords in the team password manager, with an offline backup.
  The **same upload key keeps working after the transfer**.
- If it's lost, you can request an upload-key reset from Play, but that takes days. Avoid it.
- **Never commit it.** Keep it outside the repo.

Add the passwords to your user-level Gradle properties (`~/.gradle/gradle.properties`, not the repo):
```properties
MOLE_UPLOAD_STORE_FILE=/home/<you>/keys/mole-upload.jks
MOLE_UPLOAD_STORE_PASSWORD=...
MOLE_UPLOAD_KEY_ALIAS=mole-upload
MOLE_UPLOAD_KEY_PASSWORD=...
```

Connect them to the build. `npx expo prebuild --clean` regenerates `android/`, so a hand edit to
`build.gradle` gets wiped. Use a small **config plugin** (`mobile/plugins/withReleaseSigning.js`)
that adds a `signingConfigs.release` block reading the `MOLE_UPLOAD_*` properties, and point
`buildTypes.release.signingConfig` at it. Done in monorepo PR #311 (issue #308): with the
properties set, release builds use the upload key; without them they fall back to the debug key.

### A3a. Giving another developer upload access

Two separate things:
- **Uploading to Play** needs a Play Console permission.
- **Signing a build** needs the upload key.

Give each person only what their job needs.

**Option 1 (recommended): the developer uploads, one release manager signs.**
The developer never gets the key.
1. Play Console → **Users and permissions → Invite new users** → the developer's Google email.
2. **App permissions** → `MoLe` → tick only **Release to testing tracks** (add *Release to
   production* later, only for trusted release managers). Don't make them Admin.
3. You (key holder) build and sign the AAB, then hand them the `.aab` file. They upload it and
   write the release notes.

**Option 2: the developer also builds and signs.**
1. Do steps 1–2 above.
2. Share the key **only through the team password manager** (1Password / Bitwarden shared vault):
   attach `mole-upload.jks` and store the two passwords + alias in the same item.
   **Never** send it over email, Slack/WhatsApp, Drive links, or git.
3. Developer saves it outside any repo, e.g. `~/keys/mole-upload.jks` (`chmod 600`), and adds the
   four `MOLE_UPLOAD_*` lines to **their own** `~/.gradle/gradle.properties`.
4. Developer checks their setup: `./gradlew :app:signingReport` → release shows `Config: release`,
   alias `mole-upload`. The SHA-256 must match Play Console → **App integrity → Upload key
   certificate**.

**Why sharing is survivable:** with **Play App Signing**, the upload key isn't the key users'
devices trust. Google holds that one. A leaked or lost upload key can be replaced without
affecting installs.

**When someone leaves, or the key may have leaked:**
1. Remove their Play Console access (Users and permissions).
2. Generate a new keystore (A3 `keytool` command, new file name).
3. Export its certificate:
   `keytool -export -rfc -keystore mole-upload-v2.jks -alias mole-upload -file upload_cert.pem`
4. Play Console → **App integrity → App signing → Request upload key reset** → upload
   `upload_cert.pem`, give a reason. ⚠️ The account owner must do this; Google takes a few days.
5. After approval, update the password-manager item and everyone's `gradle.properties`.

Only one upload key is active per app at a time. Keep the number of key holders small (ideally
1–2).

### A4. Set the version (see [Versioning](#versioning))
For the first upload, `mobile/app.json`:
```json
"version": "1.0.0",
"android": { "versionCode": 1 }
```

### A5. Build the AAB locally
```bash
cd ~/workspace/work/Mole/Code/mole && git switch main && git pull && pnpm install
cd mobile
export EXPO_PUBLIC_API_URL="https://<prod-api-host>"   # baked into the JS bundle. Never a tunnel.
npx expo prebuild --clean --platform android
cd android && ./gradlew bundleRelease
# → android/app/build/outputs/bundle/release/app-release.aab
```
Check the signature (it must **not** be `CN=Android Debug`):
```bash
keytool -printcert -jarfile app/build/outputs/bundle/release/app-release.aab | head -5
```
For a quick side-load APK with the same code: `./gradlew assembleRelease`.

### A6. Upload to internal testing
1. **Test and release → Testing → Internal testing → Create new release**.
2. Accept **Play App Signing** (first time only). Google keeps the app-signing key and you keep the
   upload key.
3. Upload `app-release.aab`. Release name: `1.0.0 (1)`. Paste the **internal notes** from
   [Release notes](#release-notes).
4. **Save → Review release → Start rollout to Internal testing**.
5. **Testers** tab → create the list `mole-internal` (≤100 emails) → copy the **opt-in link** →
   send it to testers.

Internal releases go live in minutes without review. You don't need the store listing or
data-safety forms yet, but fill them in now anyway (setup doc Phase 5), since production needs them.

### A7. After the first upload
- **App integrity → App signing**: copy the **app-signing SHA-1/SHA-256** into the Firebase
  Android app (`com.mole.app`). FCM and Firebase Auth checks run against the Play-signed build.
- Repeat A4–A6 for each new tester build, bumping `versionCode` every time.

---

## Phase B — Organization account

### B1. Prerequisites
- [ ] Company registered (certificate of incorporation, PAN)
- [ ] **D-U-N-S number** for the exact legal name + address (free from `dnb.co.in`, up to ~30 days)
- [ ] Company website on your domain + business email + phone

### B2. Create the org account
Setup doc Phases 0–2 with **"An organization or business"**. Use a **different** Google account
from the personal one (e.g. `play@<company-domain>`). Pay the $25, finish verification, and wait
for the dashboard to show no pending verification.

Note down from the new account:
- **Developer account ID**: the number in the Console URL (`.../developers/<ID>/...`)
- **Registration transaction ID**: Google Payments receipt for the $25

---

## Phase C — Transfer the app (personal → org)

⚠️ Google's process changes. Follow the current Help Center article **"Transfer apps to a different
developer account"**. Outline:

### C1. Prepare (in the personal account)
- [ ] No release **in review** or rolling out. Let it finish or halt it first.
- [ ] Note any existing **API access / service-account** links and **Firebase / Google Cloud /
      Ads** links. These don't move.
- [ ] Export tester lists. Email lists belong to the account and **don't** transfer.
- [ ] Have ready: org **developer account ID** + **transaction IDs** for both registrations.

### C2. Request the transfer
- Start it from the personal (source) account in Play Console. Look for **App transfer**; the Help
  article has the current path. Older flows used a support form.
- Select `com.mole.app`, enter the target account ID + transaction IDs, submit.
- The **org (target) account accepts** the transfer.
- Google reviews it, usually within a few days.

### C3. What moves / what doesn't

| Moves with the app | Does **not** move. Redo in org account |
|---|---|
| Package name `com.mole.app` | Users & permissions (re-invite team) |
| Play App Signing key | Tester email lists (recreate `mole-internal`) |
| Store listing, screenshots, policy declarations | API access / service accounts |
| Release history + all testing tracks | Linked Firebase / Google Cloud / Ads projects |
| Ratings, reviews, stats, install base | Developer page, payments profile |

The **upload keystore** stays with you, so keep signing with `mole-upload.jks`. The developer
name shown on the listing changes to the company.

### C4. After the transfer (in the org account)
- [ ] Re-invite team members with least privilege.
- [ ] Recreate tester lists and re-share the opt-in link. Existing installs keep updating.
- [ ] Re-link Firebase (Project settings → Integrations → Google Play) if used.
- [ ] Recheck the policy declarations and privacy policy URL. Update the developer name, address
      and contact email to the company's.
- [ ] Upload the next build (bumped `versionCode`) to internal testing to confirm everything works.
- [ ] Delete or close the personal account only after the transfer is confirmed. It keeps no apps.

---

## Phase D — Production launch (org account)

1. Optional but recommended: a short **closed test** (a week) with real students and educators.
2. **Production → Create new release** → promote the tested AAB. Paste the **production notes**.
3. Countries: **India** first.
4. **Staged rollout** 20% → 50% → 100%, watching crash reports (Play vitals + Sentry) at each step.
5. Backend: set `min_app_version_android` in the admin **Config** page only when you need to
   force older builds to update (see [Versioning](#versioning)).
6. Tag the commit (see below).

---

## Versioning

Two numbers in `mobile/app.json`:

| Field | Play name | Rule |
|---|---|---|
| `expo.version` | **versionName** (shown to users) | Semantic `MAJOR.MINOR.PATCH` |
| `expo.android.versionCode` | **versionCode** (internal) | Integer. **+1 for every upload, on every track, forever.** Never reused, never decreased |

### Rules
- **versionCode** is a plain counter: 1, 2, 3, … across internal, closed and production builds.
  Play rejects a code that's already been used, even if that release was discarded.
- **version (semver)**:
  - **PATCH** (`1.0.1`): bug fixes, no new features.
  - **MINOR** (`1.1.0`): new features, backwards compatible.
  - **MAJOR** (`2.0.0`): big redesign or breaking change (old builds stop working with the API).
- Internal test builds before launch all stay at **`1.0.0`**, with codes 1, 2, 3, … The first
  production release is `1.0.0` with whatever code the last tested build had.
- `expo.version` matters at runtime too:
  - `runtimeVersion.policy = appVersion`: OTA updates only reach builds with the **same**
    `version`. Bumping it means a new store build.
  - The force-update check (`utils/semver.ts` → `GET /api/v1/app/version`) compares the installed
    `version` to `min_app_version_android`. Bump `version` whenever a release must be enforceable.
- Keep `ios.buildNumber` in step when iOS ships.

### Release checklist (every build)
```bash
# 1. bump mobile/app.json  -> versionCode (+1), version (if releasing a new semver)
# 2. commit:  chore(mobile): release 1.0.0 (3)
# 3. build:   prebuild --clean -> ./gradlew bundleRelease  (Phase A5)
# 4. tag after it's uploaded:
git tag -a mobile-v1.0.0+3 -m "Android 1.0.0 (3) — internal testing"
git push origin mobile-v1.0.0+3
```
Tag format: `mobile-v<version>+<versionCode>`. The production launch also gets `mobile-v1.0.0`.

---

## Release notes

Each release has its own file under [`../releases/`](../releases/), with the Play "What's new"
text (≤500 chars, internal + production), the full changelog, and known issues:

- **v1.0.0**: [`releases/android-v1.0.0.md`](../releases/android-v1.0.0.md) (draft)

For a new release, copy the latest file to `android-v<version>.md` and edit it.

---

## Timeline at a glance

| Step | Owner | Typical time |
|---|---|---|
| Personal account + verification | you | 1–3 days |
| Upload key + signing plugin | dev | ½ day |
| First internal build live | dev | same day |
| Company + D-U-N-S | founders | up to ~30 days (start now, in parallel) |
| Org account + verification | founders | 1–3 days |
| App transfer review | Google | a few days |
| Production review | Google | a few days to ~1 week |
