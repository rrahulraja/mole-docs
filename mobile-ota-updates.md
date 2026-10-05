# Mobile OTA (Over-The-Air) Updates

How MoLe ships fixes to installed Android apps **without a Play Store release**,
how it's set up, and the release playbook for both OTA fixes and store builds.

- App: `@mole/mobile`, Expo SDK 54, React Native 0.81, package `com.mole.app`
- Builds: **local Gradle** (`expo prebuild` → `./gradlew bundleRelease`), not EAS Build
- Update service: **EAS Update** (Expo's hosted update server; separate from EAS Build)
- Status: **not active yet** (issue #309). Must be set up **before the production build**.

---

## 1. What OTA is

An installed app has two parts:

| Part | What it contains | How it changes |
|---|---|---|
| **Native shell** (the AAB/APK) | Android code, native libraries (LiveKit, Firebase, Razorpay…), permissions, icon, splash, `app.json` native config | Only through a **new Play Store release** (review, rollout) |
| **JS bundle** | All React/TypeScript screens and logic, styles, text, JS-loaded images | Can be **replaced over the air** |

OTA replaces only the **JS bundle**. When we publish an update, installed apps
download the new bundle from the update server and switch to it. There's no store
review, users don't have to do anything, and it reaches them within minutes.

```
Developer                         Expo update server                 User's phone
─────────                         ──────────────────                 ────────────
eas update --channel production ─▶ stores bundle for
                                   (runtime 1.0.0, channel production)
                                                                     app launches
                                                                     ├─ shows current bundle (no waiting)
                                                                     ├─ asks server: newer bundle for
                                                                     │  my runtime + channel?
                                   ◀──────────────────────────────── │
                                   yes → sends new bundle ─────────▶ ├─ downloads it
                                                                     └─ reloads on the new bundle
```

In MoLe the check runs at every cold start (`utils/otaUpdates.ts`). Because it
runs while the splash screen is still showing, the user lands on the newest bundle
**in the same launch**. If the phone is offline or the check fails, the app simply
keeps its current bundle; OTA never blocks startup.

### What can and can't go over the air

| Change | OTA? |
|---|---|
| Bug fix in a screen, hook or service (`.ts/.tsx`) | ✅ Yes |
| Text, colours, layout, styles | ✅ Yes |
| New JS-only screen or flow using libraries already in the app | ✅ Yes |
| Images/fonts imported from JS | ✅ Yes |
| Calling a new or changed **backend API** | ✅ Yes, if the backend is deployed first |
| Adding or upgrading a **native** library (anything needing `prebuild`) | ❌ Store build |
| Permissions, `app.json` native config, app icon, splash, notification icon | ❌ Store build |
| Expo SDK / React Native upgrade, `minSdkVersion`, target SDK | ❌ Store build |
| Anything under `mobile/android` after `prebuild` | ❌ Store build |

**Rule of thumb:** if the change needs a new `prebuild`, it needs a store build.
Everything else can go OTA.

---

## 2. Key concepts

### Runtime version: which builds can take an update
`app.json` has `"runtimeVersion": { "policy": "appVersion" }`, so a build's
**runtime version = its `version`** (e.g. `1.0.0`).

- An update published from code with `version: 1.0.0` reaches **only** installed
  builds whose version is `1.0.0`.
- This is the safety gate: a JS bundle that expects new native code can never
  land on an old shell that lacks it.

Consequences:
- **OTA fixes keep `version` unchanged.** Don't bump `version` for an OTA.
- **Bumping `version`** (1.0.0 → 1.0.1) starts a **new runtime line**. It's done
  together with a store build, and updates for 1.0.1 won't reach phones still on 1.0.0.
- `versionCode` (the Play upload counter) has **no effect** on OTA.

### Channel: which audience gets an update
Each build is baked with a **channel** name, and an update is published to a channel.

| Channel | Builds | Used for |
|---|---|---|
| `production` | Every AAB uploaded to Play (closed testing and production) | Real users and Play testers |
| `preview` | Locally built test APKs, side-loaded on our own phones | Trying an update before it goes to `production` |

A build only receives updates published to **its** channel **and** its runtime
version.

### Update URL and project ID
`eas update:configure` links the app to an Expo project. It writes two values
into `app.json`:
- `expo.extra.eas.projectId`, the Expo project's ID;
- `expo.updates.url` = `https://u.expo.dev/<projectId>`, where the app checks for updates.

They aren't secrets, but they're generated, so don't hand-write them. Because we
build locally (EAS Build would normally inject the channel), the channel also has
to be set in the app config:

```json
"updates": {
  "url": "https://u.expo.dev/<projectId>",
  "requestHeaders": { "expo-channel-name": "production" }
}
```

For `preview` APKs, the build sets the channel to `preview` through an environment
variable (see §4.4). That needs `app.json` converted to a small `app.config.js`,
which is part of the setup work.

### How OTA relates to force update
MoLe also has a **store force update**: `utils/semver.ts` calls
`GET /api/v1/app/version`. If the installed `version` is below the admin-configured
`min_app_version_android`, it shows a blocking "Update Required" dialog that links
to the Play Store.

| Layer | Mechanism | Use when |
|---|---|---|
| **OTA** | Silent JS swap | JS fixes on the same `version` |
| **Force update** | Blocking dialog → Play Store | A store release users *must* install (native fix, breaking API change) |

---

## 3. One-time setup (issue #309)

Prerequisite: an **Expo account** that will own the project. Use a shared or
company email if possible, since it controls who can publish updates.

1. **Log in** (once per machine that publishes):
   ```bash
   cd mobile
   npx eas-cli login          # or set EXPO_TOKEN (expo.dev → Account → Access tokens)
   ```
2. **Create and link the Expo project:**
   ```bash
   npx eas-cli init               # writes extra.eas.projectId (+ owner)
   npx eas-cli update:configure   # writes updates.url, sets up channels
   ```
3. **Set the channel for local builds:** add
   `updates.requestHeaders["expo-channel-name"]`. It defaults to `production` and
   can be overridden to `preview` through an env var at build time
   (`app.config.js`).
4. **Commit** the `app.json` / `app.config.js` changes (PR linked to #309).
5. **Verify end to end** before relying on it (§5): build a `preview` APK,
   publish a visible change to `preview`, and confirm it arrives.
6. **Ship the first OTA-capable store build.** Only builds made **after** this
   setup can receive updates. The closed-test build `1.0.0 (3)` can't, so the
   next upload must include it.

**Cost:** the Expo free plan includes EAS Update up to a monthly-active-users
allowance. Paid plans raise it. Check current limits at expo.dev/pricing before
launch, and watch usage on the expo.dev dashboard.

---

## 4. Release playbook

### 4.1 Decide: OTA or store build?

```
Is the fix JS/TS only (see the table in §1)?
├─ yes → Does it need a backend change?
│        ├─ yes → deploy backend first (backward compatible!), then OTA (§4.2)
│        └─ no  → OTA (§4.2)
└─ no (native change / new library / permission / icon / SDK)
         → Store build (§4.3); add force update if old builds must not stay in use
```

When in doubt, it's a store build. OTA-ing JS that depends on a native change
crashes the app on launch for everyone on that runtime.

### 4.2 Shipping a fix over the air (most bug fixes)

1. **Fix on a branch → PR → review → merge to `main`** (normal issue/PR workflow).
   Don't touch `version` in `app.json`.
2. **Check the runtime:** `app.json` `version` must equal the version of the
   builds you're targeting (what's live on Play).
3. **Publish to `preview` first** and check it on a preview APK:
   ```bash
   cd mobile && git switch main && git pull
   export EXPO_PUBLIC_API_URL=https://api.moleedtech.com
   npx eas-cli update --channel preview --message "fix: <what> (#<issue>)"
   ```
   Fully close and reopen the preview app, then confirm the fix.
4. **Publish to `production`:**
   ```bash
   npx eas-cli update --channel production --message "fix: <what> (#<issue>)"
   ```
   Users get it on their **next app launch**. Most are updated within hours or a
   day, depending on how often they open the app.
5. **Record it:** tag the commit `mobile-ota-1.0.0-<yyyymmdd>-<n>` and add a line
   to the release's notes file (`releases/android-v<version>.md` → "OTA updates").
6. **Watch** crash reports and support messages for an hour.
   **Roll back** (§4.5) if anything breaks.

`EXPO_PUBLIC_*` values are baked into the bundle **at publish time**, so always
export the production values before `eas update`. A wrong API URL in an update
breaks every user, just like in a build.

### 4.3 Shipping a store build (native changes, or bundling many fixes)

1. Merge everything to `main`.
2. **Bump versions** in `mobile/app.json`:
   - `android.versionCode` **+1**. Always, for every upload.
   - `version`:
     - **native change** → bump it (e.g. 1.0.0 → 1.0.1 or 1.1.0). This starts a new OTA runtime line;
     - **no native change** (just repackaging JS) → you may keep it, and the build stays on the same OTA line.
3. **Build:**
   ```bash
   cd mobile
   export EXPO_PUBLIC_API_URL=https://api.moleedtech.com
   npx expo prebuild --clean --platform android
   cd android && ./gradlew bundleRelease
   keytool -printcert -jarfile app/build/outputs/bundle/release/app-release.aab | head -3   # upload key, not Android Debug
   ```
4. **Upload** to Play: Closed testing (or Production once unlocked) → release notes
   from `releases/android-v<version>.md` → staged rollout.
5. **Tag** `mobile-v<version>+<versionCode>`.
6. **Force update (optional):** if older versions must stop being used, set
   `min_app_version_android` in **admin → Config** to the new `version` **after** the
   rollout reaches 100%.

### 4.4 Building a `preview` APK (to test OTA updates)

```bash
cd mobile
export EXPO_PUBLIC_API_URL=https://api.moleedtech.com
EXPO_UPDATES_CHANNEL=preview npx expo prebuild --clean --platform android
cd android && ./gradlew assembleRelease
adb install -r app/build/outputs/apk/release/app-*-release.apk
```
Use the same `version` as production, so the preview app is on the same runtime
and receives the same updates you're about to ship. The variable name is
finalised during setup.

### 4.5 Rollback

```bash
# Point the channel back to the previous update (instant, no store round-trip)
npx eas-cli update:rollback --channel production
# or republish the last known-good commit
git checkout <good-sha> && npx eas-cli update --channel production --message "rollback to <sha>"
```
Users get the rolled-back bundle on their next launch. If the bad update crashes
before it can check for updates, expo-updates falls back to the bundle embedded
in the build.

### 4.6 Worked examples

| Situation | Path |
|---|---|
| Wrong text on the booking screen | OTA (§4.2) |
| Crash in the quiz screen's JS | OTA (§4.2) |
| New backend field shown in the app | Deploy backend → OTA |
| Quiz title overlapping the status bar (layout/JS) | OTA |
| New notification icon | Store build (native resource) |
| Add a new native SDK (e.g. analytics) | Store build + bump `version` |
| Expo SDK upgrade | Store build + bump `version` + consider force update |
| Critical security fix in native code | Store build + force update |

---

## 5. Testing OTA

OTA is **off in dev and Expo Go** (`Updates.isEnabled` is false, and
`checkForOtaUpdate()` returns early on `__DEV__`). Always test on a **release
build**.

1. Build and install a `preview` APK (§4.4). Open it once.
2. Make a **visible** JS-only change (e.g. a label on the Profile screen).
3. `npx eas-cli update --channel preview --message "test: visible tweak"`.
4. **Fully close** the app (swipe it away) and reopen it. The change appears
   without reinstalling.
5. **Runtime gate check:** bump `version` locally (e.g. 1.0.0 → 1.0.1), publish to
   `preview`, and reopen. The 1.0.0 app must **not** pick it up. Revert the bump.
6. **Channel check:** a `production` build must not pick up a `preview` update.

Useful for debugging (temporary log or dev screen):
```ts
import * as Updates from 'expo-updates';
console.log(Updates.updateId, Updates.channel, Updates.runtimeVersion, Updates.isEmbeddedLaunch);
```

---

## 6. Monitoring and troubleshooting

**expo.dev → Project → Updates** shows each published update, its channel,
runtime and adoption.

| Symptom | Likely cause |
|---|---|
| Update never arrives | `version` differs from the installed build; wrong `--channel`; app not fully restarted; build made before OTA setup |
| "No update available" | Published to a different channel than the build listens on |
| Works on preview, not on Play builds | Only published to `preview`; publish to `production` too |
| App crashes right after an update | JS needs native code the shell doesn't have → roll back (§4.5) and ship a store build |
| Update points to the wrong API | `EXPO_PUBLIC_API_URL` wasn't exported when publishing → republish with the correct env |
| Nothing happens in Expo Go / `expo start` | Expected: OTA is off in dev |

---

## 7. Who does what

| Task | Who |
|---|---|
| Owns the Expo account and project | Founders (shared/company login) |
| Publishes OTA updates | Release manager (1–2 people with Expo access) |
| Builds and uploads store releases | Release manager (holds the upload key) |
| Decides OTA vs store build | Developer who made the fix, with reviewer approval in the PR |
| Sets force-update minimum version | Admin, in admin → Config |

---

## 8. File reference

| File | Role |
|---|---|
| `mobile/app.json` (→ `app.config.js` after setup) | `runtimeVersion` policy, `updates` (`enabled`, `checkAutomatically`, `url`, `requestHeaders` channel), `extra.eas.projectId`, `version`, `android.versionCode` |
| `mobile/utils/otaUpdates.ts` | `checkForOtaUpdate()`: cold-start check → fetch → reload; skipped in dev |
| `mobile/app/_layout.tsx` | Calls `checkForOtaUpdate()` and the force-update `checkForUpdate()` on startup |
| `mobile/utils/semver.ts` | Store force-update flow (`GET /api/v1/app/version` vs `min_app_version_android`) |
| `deployment/google-play-release-plan.md` | Store release, versioning rules, upload key |
| `releases/android-v<version>.md` | Release notes, plus a log of OTA updates for that version |
