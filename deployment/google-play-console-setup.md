# Google Play Console — new account setup (step by step)

Step-by-step guide to create a **new Google Play developer account** for Mole and get the
Android app (`com.mole.app`) from zero to a production listing.

> Related docs:
> - [`integrations-setup.md` §9](./integrations-setup.md) — where Play Console fits among all integrations.
> - [`google-play-release-plan.md`](./google-play-release-plan.md) — personal → internal testing → org transfer, versioning, release notes.
> - [`../mobile-ota-updates.md`](../mobile-ota-updates.md) — OTA (EAS Update) channels.
>
> Google changes Console screens and policy thresholds often. Numbers below (fees, tester counts,
> target API level) were correct when written — re-check them on the Console/Help Center when you
> do each step.

---

## Phase 0 — Decide before you start (can't easily undo)

### Step 0.1 — Personal vs Organization account

| | Personal | Organization |
|---|---|---|
| Who | An individual | A registered business (company, LLP, …) |
| Needs | Govt ID, address, phone, email | **D-U-N-S number**, business website, business phone + email, govt ID of the account owner |
| Before production | **Closed test: ≥12 testers opted in for 14 continuous days** (accounts created after Nov 2023) | No mandatory tester period |
| Developer name shown on store | Your legal name | Company name |
| Changing later | Hard — moving the app to another account is a formal **app transfer** | — |

**Recommendation for Mole:** create an **Organization** account as the company once it is
registered (payments via Razorpay are already company-bound). Use a personal account only if you
must ship before the company exists, and accept the 14-day closed-test gate + a later app transfer.

### Step 0.2 — Pick the Google account that will own it

- Use a **dedicated** Google account (e.g. `play@<company-domain>` or a new Gmail), **not** a
  personal inbox — the owner account can't be changed without support involvement.
- Turn on **2-Step Verification** on it (Google requires it for Console access).
- Plan to add other people as **users** (Step 2.3) rather than sharing this login.

### Step 0.3 — Gather documents

**Organization account:**
- [ ] D-U-N-S number — request free at `dnb.com` (search "D-U-N-S lookup / request"). **Takes up to
      ~30 days** — start this first. Company name + address on D&B must exactly match what you type
      into Play Console.
- [ ] Company website (on a domain you control; Google may ask you to verify it in Search Console).
- [ ] Business email + business phone number (both get verified by code).
- [ ] Govt ID of the person creating the account (PAN/Aadhaar/passport).

**Personal account:**
- [ ] Govt ID (passport / Aadhaar / PAN / driving licence) — name must match the payment card.
- [ ] Address proof if asked.
- [ ] Phone + contact email (both verified).
- [ ] A **physical Android device** — personal accounts must verify access to a real device via the
      Play Console mobile app.

**Both:**
- [ ] Credit/debit card for the **US$25 one-time** registration fee (international transactions
      enabled).

---

## Phase 1 — Create the developer account

### Step 1.1 — Sign up

1. Sign in to the owner Google account (Step 0.2).
2. Open `https://play.google.com/console/signup`.
3. Choose **"An organization or business"** or **"Yourself"** (Step 0.1).

### Step 1.2 — Fill developer profile

1. **Developer name** — shown publicly on the store listing (e.g. `Mole` / company name).
2. **Contact email + phone** — the *private* ones Google uses to reach you; verify both by code.
3. **Public contact email** (and optional website/phone) — shown on the store listing.
4. Organization only: enter **D-U-N-S**, organization type, size, address — must match D&B record.
5. Answer the "about you / your apps" questions (experience, expected number of apps, monetisation —
   Mole: free app, payments via third-party gateway).

### Step 1.3 — Pay the fee

Pay **US$25** with the card. Non-refundable.

### Step 1.4 — Identity verification

1. Upload the ID documents requested.
2. Wait for Google — usually 1–3 days, can be longer. You'll get an email.
3. If rejected: the usual cause is a name/address mismatch between ID, card, and D-U-N-S. Fix and
   resubmit; don't create a second account (duplicate accounts get terminated).

### Step 1.5 — Device verification (personal accounts)

1. Install the **Google Play Console** app on a physical Android phone.
2. Sign in with the owner account and complete the verification prompt.

**Done when:** Console dashboard shows no pending verification banners and **Create app** is enabled.

---

## Phase 2 — Account-level settings

### Step 2.1 — Developer page (optional)
**Settings → Developer page** — logo, header image, description. Nice-to-have.

### Step 2.2 — Payments profile (only if you sell via Google)
Mole is free and takes payments through **Razorpay**, so no Google merchant account is needed now.
Set one up only if you ever add Google Play Billing.

> ⚠️ **Check before launch:** Google's *Payments policy* requires Play Billing for **digital** goods
> consumed in the app. Live 1:1 tutoring is a real-time service, but **Mole Coins / wallet top-ups
> / promotional credits** could be read as digital currency. Review the current Payments policy
> (Play Console Help → "Understanding Google Play's Payments policy") and be ready to justify the
> Razorpay flow in the review notes.

### Step 2.3 — Add team members
**Users and permissions → Invite new users** — add developers with only the permissions they need
(e.g. *Release to testing tracks*, *Manage store presence*). Keep **Admin** to 1–2 people.

---

## Phase 3 — Create the app entry

### Step 3.1 — Create app
**Home → Create app**:
- App name: `MoLe` (matches `mobile/app.json` → `expo.name`; ≤30 chars)
- Default language: English (United States) or English (India)
- App or game: **App**
- Free or paid: **Free** (can't change free → paid later)
- Accept the declarations (Developer Program Policies, US export laws).

> The package name is **not** chosen here — it's locked by the **first uploaded AAB**. Ours is
> `com.mole.app` (`mobile/app.json` → `android.package`). Once uploaded it can **never change**.

### Step 3.2 — Play App Signing
On first release Google offers **Play App Signing** — accept it (default). Google holds the app
signing key; EAS holds the **upload key** (Step 4.3).

---

## Phase 4 — Prepare the build (repo side)

Mole builds Android **locally with Gradle** (no EAS). The full steps are in
[`google-play-release-plan.md`](./google-play-release-plan.md) Phase A3–A7:

1. **Upload keystore**: `keytool -genkeypair …` → `~/keys/mole-upload.jks`, with passwords in
   `~/.gradle/gradle.properties`. Back it up and never commit it. Play rejects debug-signed builds.
2. **Version**: bump `android.versionCode` (+1 for every upload) and `version` in `mobile/app.json`.
3. **API URL**: `export EXPO_PUBLIC_API_URL=https://<prod-api-host>`. It's baked into the bundle.
4. **Build**: `npx expo prebuild --clean --platform android` → `cd android && ./gradlew bundleRelease`
   → `app/build/outputs/bundle/release/app-release.aab`.
5. **Firebase**: production `google-services.json` for `com.mole.app`. After the first upload, add
   Play's **app-signing SHA-1/SHA-256** (*Test and release → App integrity*) to the Firebase Android app.
6. **Target API level**: Google raises the minimum every August. Upgrade the Expo SDK if the
   Console blocks the upload.

---

## Phase 5 — Store listing + App content (required before any review)

Left menu → **Grow users → Store presence → Main store listing**, and **Policy and programs →
App content**. Every item below must be green.

### Step 5.1 — Main store listing
- [ ] App name (≤30), short description (≤80), full description (≤4000)
- [ ] App icon **512×512 PNG** (32-bit, ≤1 MB)
- [ ] Feature graphic **1024×500** JPG/PNG
- [ ] **≥2 phone screenshots** (16:9 or 9:16, 320–3840 px) — student + educator flows
- [ ] (Optional) 7" / 10" tablet screenshots, promo video (YouTube URL)
- [ ] App category: **Education**; contact email; website

### Step 5.2 — App content declarations
- [ ] **Privacy policy URL**: `https://app.moleedtech.com/privacy-policy` (terms: `/terms-conditions`), served by the web app (monorepo #314). It must be a live page, not a PDF, and must name the
      developer and describe data collected.
- [ ] **App access** — Mole is phone-OTP gated. Select *"All or some functionality is restricted"*
      and give reviewers a **test phone number + fixed OTP** (configure a test-number bypass in the
      backend OTP provider), one student and one educator login, plus steps.
- [ ] **Ads** — "No, my app does not contain ads" (unless that changes).
- [ ] **Content rating** — fill the IARC questionnaire (Category: reference/education; mention
      user-to-user communication: live video + chat).
- [ ] **Target audience and content** — pick age groups. If **under-13** is included, the app falls
      under the **Families policy** (stricter: no unreviewed SDKs, teacher-approved etc.). Decide
      deliberately; 18+ or 13+ is simpler if educators/students are adults/teens.
- [ ] **Data safety** — declare everything collected/shared:
  | Data | Why |
  |---|---|
  | Phone number | Login (OTP) |
  | Name, email, photo | Profile |
  | Approximate/precise location | Nearby educators, regional pricing |
  | Photos / files | KYC + education documents |
  | Audio, video (camera/mic) | Live sessions; recordings stored |
  | Payment info | Processed by Razorpay (declare "shared with payment processor") |
  | Device IDs / push token | FCM notifications |
  | Crash logs / diagnostics | Sentry, if enabled |
  Mark: data encrypted in transit = yes; users can request deletion = yes (Step 5.2 next item).
- [ ] **Account deletion** — Google requires an in-app path **and** a web URL where users can
      request account + data deletion. Provide the URL in the Data safety form.
- [ ] **Government apps / Financial features / Health** — answer "No" / pick none unless that
      changes (wallet is a stored-value credit, not a financial product — re-read the Financial
      features question carefully).
- [ ] **Permissions declarations** — `FOREGROUND_SERVICE_MEDIA_PROJECTION` (LiveKit screen share)
      requires a **foreground service type declaration** with a short video showing the feature.
      Camera/mic/location are justified by video sessions and nearby matching.
- [ ] **News app** — No.

---

## Phase 6 — Testing tracks → production

### Step 6.1 — Internal testing (minutes, no review)
1. **Test and release → Testing → Internal testing → Create new release**.
2. Upload `app-release.aab` (from Phase 4).
3. Release name + notes → **Save → Review release → Start rollout**.
4. **Testers** tab → create an email list (≤100) → copy the **opt-in link** → share.
5. Testers open the link, accept, install from Play. Smoke-test login, booking, payment, video.

### Step 6.2 — Closed testing (mandatory for personal accounts)
1. **Closed testing → Create track** (or use *Alpha*) → add the same AAB.
2. Add ≥12 testers (Google Group or email list); they must **opt in and keep the app installed**
   for **14 continuous days**.
3. Submit for review (first closed release is reviewed — can take days).
4. After 14 days: **Dashboard → Apply for production** → answer the questions about your test.

Organization accounts can skip straight to production but should still run a short closed test.

### Step 6.3 — Production
1. **Production → Create new release** → promote the tested AAB.
2. **Countries/regions** — start with **India** (where Razorpay/Twilio are set up).
3. Use a **staged rollout** (e.g. 20% → 50% → 100%).
4. **Send for review**. First review: typically a few days, sometimes up to ~7.

### Step 6.4 — Every later release
Bump `versionCode` (and `version` if needed) → rebuild (Phase 4) → upload the AAB to the track →
paste release notes → tag. Versioning rules, tag format and release notes are in
[`google-play-release-plan.md`](./google-play-release-plan.md#versioning). JS-only fixes can go
over OTA instead (see [`mobile-ota-updates.md`](../mobile-ota-updates.md)).

---

## Checklist summary

- [ ] Account type decided (Org recommended) · D-U-N-S requested
- [ ] Dedicated owner Google account with 2SV
- [ ] $25 paid · identity verified · (personal) device verified
- [ ] Team users invited with least-privilege
- [ ] App created (Free, Education)
- [ ] Upload keystore created + backed up · release signing wired · prod `google-services.json`
- [ ] Store listing assets uploaded
- [ ] Privacy policy · account-deletion URL · Data safety · content rating · target audience ·
      app access test login · foreground-service declaration
- [ ] Internal test passed
- [ ] Closed test 12×14 days (personal) → production access granted
- [ ] Production staged rollout · Play signing SHA added to Firebase

## Common rejection reasons (pre-empt them)

- Reviewer **couldn't log in** (OTP) — always provide a working test number + OTP.
- **Data safety mismatch** — SDK collects something not declared (Firebase, LiveKit, Sentry).
- **Missing account-deletion** URL / in-app option.
- **Foreground service** type used without a declaration video.
- **Broken API URL** baked into the build — app shows network errors to the reviewer.
- Metadata issues — keyword stuffing, screenshots not showing the real app.
