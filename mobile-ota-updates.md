# Mobile OTA (Over-The-Air) Updates

Ship JavaScript, styling, and asset fixes to installed Mole apps **without a Play
Store / App Store release**, using `expo-updates` + EAS Update.

- Package: `@mole/mobile` (Expo SDK 54, React Native 0.81)
- Update service: [EAS Update](https://docs.expo.dev/eas-update/introduction/)
- Wired in: `mobile/app.json`, `mobile/eas.json`, `mobile/utils/otaUpdates.ts`,
  `mobile/app/_layout.tsx`

---

## 1. What OTA can and cannot ship

A built app = **native shell** (the APK/AAB/IPA: native modules, permissions,
SDK) + **JS bundle** (all your React/TypeScript + JS-loadable assets). OTA
replaces only the JS bundle.

| Change | Ship via OTA? |
|--------|---------------|
| Bug fix in a `.tsx` screen, logic, hooks | ✅ Yes |
| Style / color / copy change | ✅ Yes |
| New image / font bundled in JS | ✅ Yes |
| New **native** module (e.g. add a native SDK) | ❌ Store build |
| New permission, `app.json` native config, icon/splash | ❌ Store build |
| Expo SDK bump / `react-native` bump / `minSdkVersion` | ❌ Store build |

Rule of thumb: **if it changes anything under `mobile/android` or `mobile/ios`
after `prebuild`, it needs a store build**. Pure JS/asset changes go OTA.

---

## 2. How it works

### Runtime flow (on the user's device)

1. App cold-starts on the **embedded** bundle (the one baked into the installed
   binary) — instant, never blocks on the network.
2. `expo-updates` asks the EAS Update server: *"is there a newer bundle on my
   channel with a matching `runtimeVersion`?"*
   - Native auto-check is on via `app.json` → `updates.checkAutomatically: "ON_LOAD"`.
   - We also call `checkForOtaUpdate()` from `utils/otaUpdates.ts` in the root
     layout so an available update is fetched and applied **this** launch (while
     the splash is still up) instead of the next one.
3. If a matching update exists, it downloads, then the app reloads onto the new
   bundle.
4. If offline / nothing new / `runtimeVersion` mismatch → app keeps the embedded
   bundle. OTA never blocks startup.

### `runtimeVersion` — the compatibility gate (critical)

A JS bundle only loads on a native shell whose `runtimeVersion` **matches**.

We use (`app.json`):

```json
"runtimeVersion": { "policy": "appVersion" }
```

So `runtimeVersion` == `expo.version` (currently `1.0.0`).

- Publish OTA while `version` stays `1.0.0` → every installed `1.0.0` shell picks
  it up. **This is the normal hotfix path.**
- Bump `version` to `1.1.0` (a store build with native changes) → its
  `runtimeVersion` becomes `1.1.0`. Old `1.0.0` shells now **ignore** any
  `1.1.0` OTA (and vice-versa).

Why it matters: it makes it **impossible** to push a JS bundle that calls a
native API the installed shell doesn't have — which would otherwise crash on
launch. Always bump `version` in the same PR that adds/changes native code.

### Channels

`eas.json` maps each build profile to an update **channel**:

| Build profile | Channel | Used for |
|---------------|---------|----------|
| `preview` | `preview` | internal APK testing |
| `production` | `production` | Play Store / App Store builds |

`eas update --channel production …` only reaches production builds; `preview`
testers are unaffected. A build listens on exactly the channel it was built with.

### How it fits the existing force-update flow

Mole already has a **hard** force-update: `utils/semver.ts` → `checkForUpdate()`
hits `GET /api/v1/app/version` and, if the installed `version` is below the
configured `minVersion`, shows a blocking "Update Required" alert linking to the
store. Two complementary layers:

| Layer | Mechanism | Use when |
|-------|-----------|----------|
| **OTA** (`otaUpdates.ts`) | silent JS swap | JS/style bug fixes, same `version` |
| **Force update** (`semver.ts` + `/app/version`) | blocking alert → store | native change or breaking API; must go to store |

---

## 3. One-time setup (first time only)

Requires an [Expo account](https://expo.dev) and the EAS CLI
(`npm i -g eas-cli`, then `eas login`). Run everything from `mobile/`.

1. **Link the repo to an EAS project** (creates `extra.eas.projectId` and sets
   the app owner):

   ```bash
   cd mobile
   eas init
   ```

2. **Configure EAS Update** (adds `updates.url = https://u.expo.dev/<projectId>`
   to `app.json` and creates the `production` / `preview` update branches):

   ```bash
   eas update:configure
   ```

   > These two commands fill in the only pieces intentionally left out of the
   > committed `app.json`: `extra.eas.projectId` and `updates.url`. They are
   > per-account values, not secrets, but they are generated — don't hand-write
   > them.

3. **Set the iOS App Store id** once the app exists in App Store Connect (used by
   the force-update deep link, `constants/config.ts`):

   ```bash
   # e.g. in eas.json build env or your shell before a build
   EXPO_PUBLIC_IOS_APP_ID=1234567890
   ```

4. **Build and distribute the native shell once** (this shell embeds
   `expo-updates` and starts listening on its channel):

   ```bash
   eas build --profile preview   --platform android   # internal APK, channel=preview
   # later, for the stores:
   eas build --profile production --platform android
   eas build --profile production --platform ios
   ```

After step 4, that installed binary can receive OTA updates for its
`runtimeVersion` + channel.

---

## 4. Publishing an OTA update (the routine)

Whenever you have a **JS-only** fix to ship:

```bash
cd mobile
# make sure app.json `version` is UNCHANGED since the target build,
# otherwise runtimeVersion won't match and nobody gets the update.

eas update --channel preview --message "fix: booking confirm crash"
# once verified on internal testers:
eas update --channel production --message "fix: booking confirm crash"
```

EAS bundles the current JS + assets, uploads them, and every matching installed
app fetches them on next launch (or this launch, per `otaUpdates.ts`).

**Rollback** (instant — no store round-trip):

```bash
eas update:rollback --channel production
# or just publish the previous known-good commit again
eas update --channel production --message "rollback to <sha>"
```

---

## 5. Testing OTA

OTA is **disabled in dev / Expo Go** by design (`Updates.isEnabled` is false, and
`checkForOtaUpdate()` early-returns on `__DEV__`). You must test against a
**real built binary**, not `expo start`.

### A. End-to-end with EAS (closest to production)

1. Build + install a `preview` APK on a device/emulator:
   ```bash
   eas build --profile preview --platform android
   # install the resulting APK, open the app once (this is the baseline bundle)
   ```
2. Make a **visible** JS-only change (e.g. change the profile version label copy
   or a button color).
3. Publish it to the same channel:
   ```bash
   eas update --channel preview --message "test: visible tweak"
   ```
4. **Fully close** the app (swipe it away) and reopen it. `checkForOtaUpdate()`
   fetches the new bundle and reloads. You should see your change without
   reinstalling the APK.
5. Confirm which bundle is live from JS if needed:
   ```ts
   import * as Updates from 'expo-updates';
   console.log(Updates.updateId, Updates.channel, Updates.runtimeVersion);
   ```

### B. Verify the `runtimeVersion` gate

1. On the installed `1.0.0` build, bump `app.json` `version` to `1.1.0` locally
   and `eas update --channel preview`.
2. Reopen the `1.0.0` app → it should **NOT** pick up the `1.1.0` update (runtime
   mismatch). This proves the safety gate works. Revert the local bump.

### C. Quick local sanity (no update server)

`checkForOtaUpdate()` is a plain async function guarded by `__DEV__` /
`Updates.isEnabled`, so in Expo Go it simply no-ops — confirm the app still boots
normally with the hook wired in.

---

## 6. Troubleshooting

| Symptom | Likely cause |
|---------|--------------|
| Update never arrives | `version` (hence `runtimeVersion`) changed since the build; or wrong `--channel`; or app not fully restarted. |
| "No update available" but you published | Published to a different channel than the build listens on. |
| Works in preview, not production | You only ran `eas update --channel preview`; publish to `production` too. |
| Crash right after an update | You shipped JS that needs a native change — bump `version` and ship a store build instead. |
| Nothing happens in Expo Go | Expected — OTA is off in dev; test a built binary. |

---

## 7. File reference

| File | Role |
|------|------|
| `mobile/app.json` | `runtimeVersion` policy + `updates` block (`enabled`, `checkAutomatically`, `fallbackToCacheTimeout`). `updates.url` + `extra.eas.projectId` added by `eas update:configure`. |
| `mobile/eas.json` | build profiles → update `channel` mapping. |
| `mobile/utils/otaUpdates.ts` | `checkForOtaUpdate()` — cold-start check/fetch/reload; no-ops in dev. |
| `mobile/app/_layout.tsx` | calls `checkForOtaUpdate()` on mount (alongside the store force-update `checkForUpdate()`). |
| `mobile/utils/semver.ts` | separate **store** force-update flow (`/app/version`). |
