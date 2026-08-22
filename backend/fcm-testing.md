# Testing Push Notifications (FCM)

How push notifications work in Mole and how to test them end-to-end.

## How it works

- Every notification goes through **`sendPushWithRecord(fcmToken, userId, title, body, data?)`**
  (`backend/src/services/notification/index.ts`). It **always writes a `Notification`
  row** (so web and token-less users still see it in-app) **and** additionally sends an
  FCM push **only if the user has an `fcmToken`**.
- FCM send uses **firebase-admin** (`backend/src/lib/firebase.ts`), initialized from the
  **`FCM_SERVICE_ACCOUNT`** env (the service-account JSON as a string). If it's unset the
  backend logs `FCM_SERVICE_ACCOUNT not set — push notifications disabled` and pushes are
  skipped (in-app notifications still work).
- The device registers its token via **`PUT /api/v1/auth/fcm-token`** `{ fcmToken }`
  (authenticated), which stores it on `User.fcmToken`.
- **Mobile** does the registration in `mobile/app/_layout.tsx` using
  `@react-native-firebase/messaging` (request permission → `getToken()` → PUT the token).
- **Web has no FCM** — it relies on the persisted `Notification` rows (in-app bell). So
  **FCM push testing = the mobile app**.

> `@react-native-firebase/messaging` is **native** — it does **not** work in **Expo Go**.
> You need a **dev build** (or a real build), a physical device or an emulator **with
> Google Play services**, and the Firebase config files.

---

## 1. Backend: configure the service account

1. Firebase console → your project → **Project settings → Service accounts → Generate new
   private key** → downloads a JSON file.
2. Put it in `backend/.env` as a **single-line** string (the whole JSON, quoted):
   ```
   FCM_SERVICE_ACCOUNT='{"type":"service_account","project_id":"…","private_key":"-----BEGIN PRIVATE KEY-----\n…\n-----END PRIVATE KEY-----\n", …}'
   ```
   Keep the `\n` escapes inside `private_key` intact.
3. Restart the backend. You should **not** see the "push notifications disabled" warning.
   (Never commit this value — `.env` is gitignored.)

## 2. Mobile: Firebase config + a dev build

1. Firebase console → add an **Android app** (package from `mobile/app.json`) → download
   **`google-services.json`**. For iOS add an iOS app → **`GoogleService-Info.plist`**.
   Place them where the Expo Firebase plugin expects (see `mobile/app.json` /
   `app.config`).
2. Build a **dev client** (not Expo Go):
   ```bash
   # EAS (cloud) — or `npx expo prebuild` + a local native build
   eas build --profile development --platform android
   ```
   Install it on a device/emulator with Google Play.
3. Point the app at your backend (`EXPO_PUBLIC_API_URL`) and run:
   ```bash
   pnpm --filter @mole/mobile start --dev-client
   ```
4. Log in. On launch the app requests notification permission and PUTs the token. **Grant
   permission.**

## 3. Verify the token was stored

```bash
# from backend/ — check the logged-in user's fcmToken landed in the DB
```
```ts
// backend/_t.ts (temp)
import { prisma } from './src/lib/prisma';
(async () => {
  const u = await prisma.user.findFirst({ where: { phone: '<your phone>' }, select: { name: true, fcmToken: true } });
  console.log(u?.name, u?.fcmToken ? 'token ✓ ' + u.fcmToken.slice(0, 12) + '…' : 'NO TOKEN');
  await prisma.$disconnect();
})();
```
```bash
cd backend && npx ts-node _t.ts && rm _t.ts
```
No token → permission denied, running Expo Go, or the config files are missing.

## 4. Trigger a notification

**Option A — through the app (real flow).** Any action that calls `sendPushWithRecord`:
- Student **books a session** → educator gets *"New session request"*.
- Educator **accepts/ends** a session → student gets a push.
- Wait for the **1-hour / 5-minute reminder** crons, or the **review-prompt** cron.

**Option B — direct send to a token (fastest).** Temp script using the same admin SDK:
```ts
// backend/_push.ts
import admin from './src/lib/firebase';
(async () => {
  const token = process.argv[2];
  const res = await admin.messaging().send({
    token,
    notification: { title: 'Mole test', body: 'FCM is working 🎉' },
    android: { priority: 'high' },
  });
  console.log('sent:', res);
})();
```
```bash
cd backend && npx ts-node _push.ts "<device-fcm-token>" && rm _push.ts
```

**Option C — Firebase console.** Cloud Messaging → **Send test message** → paste the
device FCM token. No backend involved — isolates whether the problem is Firebase or the app.

## 5. Testing without FCM (local dev)

You don't need FCM to test the notification **content/flow**: `sendPushWithRecord` always
writes a `Notification` row, so the **in-app bell** (`GET /notification`) shows every
notification on web and mobile regardless of FCM. Configure FCM only when you specifically
need the **device push banner**.

Check in-app delivery:
```bash
# with a user's bearer token
curl -s "$API/api/v1/notification?limit=5" -H "Authorization: Bearer <token>"
```

---

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Backend logs "push notifications disabled" | `FCM_SERVICE_ACCOUNT` unset/empty |
| `Failed to parse FCM_SERVICE_ACCOUNT` | JSON not single-line / broken `\n` in `private_key` |
| No `fcmToken` in DB | Running Expo Go (not a dev build), permission denied, or missing `google-services.json` |
| `[FCM] Push failed` `messaging/registration-token-not-registered` | Stale/uninstalled token — re-register from the app |
| Push works via Firebase console but not the app | Backend service account is for a **different** Firebase project than the app's config files |
| In-app bell works, no device banner | FCM not configured (expected) — see §5 |

## Related
- Notification generation & the crons that send them: `docs/backend/crons.md`.
