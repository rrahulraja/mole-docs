# Running Mole locally with PM2, sharing for testing, and the mobile app on WSL

This guide covers three things:
1. Running backend + web + admin under **PM2** on your machine.
2. Letting **other people on your network (or the internet)** reach it — including the **WSL2** networking quirks.
3. Running the **mobile** app against an **Android emulator** when your code lives in WSL.

Ports used throughout: **backend 3000**, **web 3001**, **admin 3002**, **Metro/Expo 8081**.

---

## 0. Seed test data first

```bash
# from repo root — 10 educators (pending) + 10 students, realistic data
pnpm --filter @mole/backend db:seed:test

# change the counts anytime
SEED_EDUCATORS=25 SEED_STUDENTS=40 pnpm --filter @mole/backend db:seed:test
```

Seeded logins use OTP with these phone numbers (any OTP if your SMS provider is in test/mock mode):
- Educators: `+919000000001 … +9190000000NN` (all in **pending** approval state)
- Students: `+918000000001 … +8000000NN`

---

## 1. Run everything under PM2

Install PM2 once:

```bash
pnpm add -g pm2   # or: npm i -g pm2
```

For sharing with others, run **production builds** (more stable than dev servers). Build first:

```bash
# backend
pnpm --filter @mole/backend build            # compiles to backend/dist
pnpm --filter @mole/backend db:migrate        # prisma migrate deploy
# web (Next.js) — bakes NEXT_PUBLIC_* at build time, so set the API URL now (see §2)
NEXT_PUBLIC_API_URL="http://<SHARE_HOST>:3000" pnpm --filter @mole/web build
# admin (Vite)
pnpm --filter @mole/admin build               # outputs admin/dist
```

Create `ecosystem.config.js` in the repo root:

```js
module.exports = {
  apps: [
    {
      name: 'mole-backend',
      cwd: './backend',
      script: 'dist/index.js',
      env: { NODE_ENV: 'production', PORT: '3000' },
    },
    {
      name: 'mole-web',
      cwd: './web',
      // next start binds 0.0.0.0 by default; -H is explicit
      script: 'node_modules/.bin/next',
      args: 'start -p 3001 -H 0.0.0.0',
    },
    {
      name: 'mole-admin',
      cwd: './admin',
      // serve the built SPA; --host exposes it on the LAN
      script: 'node_modules/.bin/vite',
      args: 'preview --port 3002 --host 0.0.0.0',
    },
  ],
};
```

Start / manage:

```bash
pm2 start ecosystem.config.js
pm2 status
pm2 logs mole-backend      # tail a service
pm2 restart mole-web
pm2 stop all
pm2 save                   # persist across reboots
pm2 startup                # (optional) run on boot
```

> **Dev-server variant:** to run hot-reloading dev servers under PM2 instead of
> builds, point each `script` at the workspace command, e.g. backend
> `pnpm dev`, web `pnpm dev` (already `next dev -p 3001`), admin `pnpm dev`
> (add `--host` to the vite dev script). Fine for solo use; prefer builds when
> others are connecting.

---

## 2. Point the frontends at a shareable backend URL

The web app reads `NEXT_PUBLIC_API_URL` (default `http://localhost:3000`) — this is **baked in at build time**, so rebuild web whenever it changes. The admin app calls `/api/v1` and relies on Vite's dev proxy; for `vite preview` you should instead point it at the backend via an env the admin reads, or keep admin on the same host as the backend and use a reverse proxy. Simplest for testing: set the backend URL to whatever address your testers will use (`<SHARE_HOST>` below) and rebuild.

Mobile reads its API base from `EXPO_PUBLIC_API_URL` (see §4) — set it to the same `<SHARE_HOST>:3000`.

`<SHARE_HOST>` is:
- **Same machine:** `localhost`
- **Other people on your LAN:** your **Windows host LAN IP** (see §3)
- **Internet:** the tunnel URL (see §3, option B)

---

## 3. Let other people reach your machine (WSL2)

WSL2 runs in a NAT'd VM with its **own IP** (172.x.x.x). Windows forwards `localhost`
into WSL automatically, but **other machines on your LAN cannot reach WSL services by
default** — you must forward the ports from the Windows host into WSL.

### Option A — LAN sharing via Windows portproxy (free, same network only)

1. **Find your Windows LAN IP** (Windows PowerShell): `ipconfig` → the IPv4 of your Wi‑Fi/Ethernet adapter, e.g. `192.168.1.20`. This is `<SHARE_HOST>`.
2. **Find your WSL IP** (in WSL): `hostname -I` → e.g. `172.28.x.x`.
3. **Add port forwarding** (Windows PowerShell **as Administrator**), one per port:
   ```powershell
   netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=3000 connectaddress=<WSL_IP> connectport=3000
   netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=3001 connectaddress=<WSL_IP> connectport=3001
   netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=3002 connectaddress=<WSL_IP> connectport=3002
   ```
4. **Open the Windows Firewall** for those inbound ports (Administrator PowerShell):
   ```powershell
   New-NetFirewallRule -DisplayName "Mole 3000-3002" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 3000-3002
   ```
5. Testers open `http://<SHARE_HOST>:3001` (web) / `:3002` (admin). API is `http://<SHARE_HOST>:3000`.

> **Gotcha:** the WSL IP **changes on every WSL restart**. Re-run step 3 after a
> reboot (list current rules with `netsh interface portproxy show all`, reset with
> `netsh interface portproxy reset`). Script it if you do this often.

### Option B — Public HTTPS via a tunnel (easiest; works off your network)

No portproxy/firewall needed. Each service gets a public URL.

```bash
# Cloudflare Tunnel (no account needed for quick tunnels)
cloudflared tunnel --url http://localhost:3000     # backend
cloudflared tunnel --url http://localhost:3001     # web
# …or ngrok
ngrok http 3000
```

Then rebuild web/mobile with `NEXT_PUBLIC_API_URL` / `EXPO_PUBLIC_API_URL` set to the
**backend** tunnel URL. Best for sharing with people outside your LAN or on mobile data.

---

## 4. Mobile app + Android emulator on WSL

The Android emulator needs hardware acceleration, which is painful inside WSL. **Run the
emulator on Windows; run Metro/Expo in WSL.** The cleanest bridge across the WSL↔Windows
boundary is Expo's tunnel.

### Setup
1. Install **Android Studio on Windows**, create an AVD (Pixel + a recent system image), and start it.
2. Install **Expo Go** on the emulator (Play Store), or use a dev build.
3. In WSL, set the API base and start Expo with a tunnel:
   ```bash
   # EXPO_PUBLIC_API_URL must be reachable from the emulator — use a tunnel URL (§3B)
   EXPO_PUBLIC_API_URL="https://<backend-tunnel-host>" \
     pnpm --filter @mole/mobile start --tunnel
   ```
4. In the emulator's Expo Go, open the tunnel URL Expo prints (or scan the QR).

### Why `--tunnel`
Metro binds to the **WSL** IP on port 8081, which the Windows emulator can't reach over
LAN by default. `--tunnel` routes through Expo's relay so it works regardless of the
WSL/Windows network split. (LAN mode would also require a portproxy for 8081 like §3A.)

### adb from WSL (optional)
To drive the Windows emulator with `adb` from WSL, run the adb **server on Windows** and
tell WSL to use it:
```bash
# in WSL
export ADB_SERVER_SOCKET=tcp:$(ip route | awk '/default/ {print $3}'):5037
adb devices    # should list the Windows emulator
```
(The `default` route gateway is the Windows host from inside WSL2.) Otherwise you don't
need adb at all — Expo Go loads the bundle over the tunnel.

### Simpler alternative
Run a **physical Android phone** on the same Wi‑Fi with Expo Go and `--tunnel` — no
emulator, no acceleration issues.

---

## Troubleshooting: "my phone can't connect"

Work through these in order — the first is the most common cause.

1. **Is the phone on mobile data (cellular)?** Then a LAN IP (`192.168.x.x` /
   `10.x` / `172.x`) is **unreachable by definition** — those are private
   addresses that don't exist on the internet. portproxy + firewall cannot fix
   this. **Use a tunnel (Option B) — it's the only thing that works off your
   Wi-Fi.** This is almost always the issue when "it works on my laptop but not
   my phone on mobile data."

2. **Is the service bound to all interfaces, not just loopback?** In WSL:
   ```bash
   ss -tlnp | grep -E ':300[0-2]'
   ```
   `*:3000` / `0.0.0.0:3000` = reachable. `127.0.0.1:3000` = **loopback only**,
   unreachable from outside even with portproxy. Dev servers that bind loopback
   need a host flag: web `next dev` already binds `0.0.0.0`; admin needs
   `vite --host`; backend Express binds all interfaces by default.

3. **Does the portproxy `connectaddress` match the CURRENT WSL IP?** WSL2's IP
   changes on every restart. Check the live IP with `hostname -I`, then on
   Windows:
   ```powershell
   netsh interface portproxy show all      # connectaddress must equal the current WSL IP
   netsh interface portproxy reset          # clear stale rules, then re-add (see §3A)
   ```

4. **Firewall** — confirm the inbound rule is for the **Private** profile if
   you're on a home/Wi-Fi network (Windows classifies networks; a rule scoped to
   "Public" won't apply on a "Private" network, and vice-versa).

### Fastest reliable path for phone testing (any network)

```bash
# 1. one tunnel for the backend — copy the https URL it prints
cloudflared tunnel --url http://localhost:3000

# 2a. browser (web/admin) on the phone: give each its own tunnel too
cloudflared tunnel --url http://localhost:3001    # web
cloudflared tunnel --url http://localhost:3002    # admin
#     then rebuild web with NEXT_PUBLIC_API_URL=<backend-tunnel-url> and reload

# 2b. mobile app on the phone:
EXPO_PUBLIC_API_URL="<backend-tunnel-url>" pnpm --filter @mole/mobile start --tunnel
```

No portproxy, no firewall, no WSL-IP juggling — works on cellular and Wi-Fi alike.

## Quick reference

| Task | Command |
|------|---------|
| Seed test data | `pnpm --filter @mole/backend db:seed:test` |
| Start all (PM2) | `pm2 start ecosystem.config.js` |
| Logs | `pm2 logs mole-backend` |
| WSL IP | `hostname -I` |
| Windows LAN IP | `ipconfig` (PowerShell) |
| Forward a port to WSL | `netsh interface portproxy add v4tov4 listenport=<P> connectaddress=<WSL_IP> connectport=<P>` |
| Public tunnel | `cloudflared tunnel --url http://localhost:3000` |
| Mobile on emulator | `pnpm --filter @mole/mobile start --tunnel` |
