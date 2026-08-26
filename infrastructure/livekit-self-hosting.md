# Self-Hosting LiveKit for Mole

Mole runs 1:1 tutoring video sessions on **LiveKit** (`VIDEO_PROVIDER=livekit`).
Today the SFU is **LiveKit Cloud**. This doc covers running the SFU yourself —
what it costs, which features you keep, where to host it, and how to deploy.

- **Web** renders via `@livekit/components-react` (`VideoConference`).
- **Mobile** renders via `@livekit/react-native` (native, dev/EAS build — no Expo Go).
- **Recording** is server-side **egress → S3** and is a *separate* service from the SFU.

> **TL;DR:** For video/screen-share/chat, self-hosting the SFU is cheap and easy
> (one small VM, ~$10–25/mo near your users). **Recording (egress) is the heavy
> part** — it needs its own container + Redis + a beefy CPU box. At Mole's current
> volume, self-host the SFU and either skip recording or keep it on LiveKit Cloud.

---

## 1. What runs where

LiveKit is not one process. Self-hosting means choosing which of these you run:

| Component | Purpose | Weight | Mole uses it? |
|---|---|---|---|
| **livekit-server** (SFU) | Routes audio/video/data between participants | Light (CPU per-stream) | **Yes — core** |
| **egress** | Records/composites rooms, uploads to S3 | **Heavy** (headless Chrome) | Yes — session recording |
| **ingress** | Pull RTMP/WHIP *into* a room | Medium | No |
| **SIP** | Phone dial-in/out | Medium | No |
| **Redis** | Shared state; **required** once you run egress/multi-node | Light | Only with egress |
| **TURN** | Relay media through strict NATs/firewalls | Light–medium | Recommended |

The SFU alone gives the full live experience. Egress is the only piece that
changes the cost/ops story materially.

---

## 2. Feature matrix — self-host vs cloud

### Runs on the OSS server (free, no extra services)

| Feature | Self-host | Note |
|---|---|---|
| Audio/video calls | Full | Core SFU; identical to cloud |
| Screen sharing | Full | Web + mobile, no code change |
| In-call chat | Full | LiveKit **data channel** (`publishData`) — Mole's `SessionChat` |
| Camera on/off, mute, initials avatar | Full | Client-side, server-agnostic |
| Multi-participant / group | Full | SFU scales fine at Mole's size |
| Token auth (room = booking id) | Full | Same API key/secret minting — no code change |
| Adaptive quality / simulcast | Full | Built into the OSS server |

### Needs extra self-hosted services

| Feature | Requires |
|---|---|
| **Recording (egress → S3)** | `livekit/egress` container **+ Redis** + a CPU-heavy box |
| RTMP/WHIP ingest | `livekit/ingress` container (unused by Mole) |
| Phone dial-in (SIP) | `livekit/sip` container (unused by Mole) |

### Cloud-only (no OSS equivalent)

- Global edge/relay network (auto low-latency routing worldwide)
- Managed TURN over TCP/443 that punches through strict corporate/carrier firewalls
- Dashboard, analytics, auto-scaling, managed uptime

**Bottom line for Mole:** the entire live session — video, screen share, chat,
camera-off initials — runs 100% on a self-hosted OSS server. Only **recording**
adds real setup.

---

## 3. Cost estimate — 500 sessions/month, 60 min each

Assumptions: 1:1 tutoring = **2 participants/session**, ~1.2 Mbps per video stream.

- 500 sessions × 60 min = 30,000 session-min = **60,000 participant-min/month**
- The SFU forwards each stream to the other party. Billable data is **egress
  (data OUT to the internet)** = what participants download. Data IN is free.

**Bandwidth math:**
```
2 participants × 1.2 Mbps × 3600 s ≈ 1.3 GB per session
500 sessions × 1.3 GB       ≈ 650 GB egress / month
```

### 3a. Video-only (no recording)

| Item | AWS EC2 | Budget host (e.g. Vultr/Hetzner) |
|---|---|---|
| SFU instance (2 vCPU / 4 GB), 24/7 | ~$60 (~$38 reserved) | ~$9–24 |
| Egress bandwidth ~650 GB | ~$60 (@ $0.09/GB) | **$0** (bundled 3–20 TB) |
| IP + misc | ~$5 | ~$0 |
| **Total** | **~$100–125/mo** | **~$10–25/mo** |

**Bandwidth is half the AWS bill** — and exactly where budget hosts win (they
bundle terabytes free). 650 GB is nothing against Hetzner's 20 TB allowance.

### 3b. With recording (egress → S3)

| Component | Spec | ~Cost/mo |
|---|---|---|
| SFU | 2 vCPU / 4 GB, 24/7 | $9–60 |
| Redis | t3.micro / small managed | $8–12 |
| **Egress** | CPU-heavy (headless Chrome), ~4 vCPU, autoscaled to peak concurrency | **$120–250** |
| Bandwidth | ~650 GB out | $0–60 |
| S3 storage | ~350 GB recordings (expire them) | ~$8 |
| **Total** | | **~$150–430/mo** |

Egress dominates. It composites in headless Chrome — one concurrent recording
wants ~2–4 vCPU. Cost scales with **peak concurrent recordings**, not total minutes.

### 3c. vs LiveKit Cloud (current setup)

- Connection/bandwidth bundled; ~$0.12/GB after the plan tier.
- **Recording ~$0.006/composite-min** → 30,000 min × $0.006 ≈ **$180/mo just recording**.
- All-in for this volume: **~$200–300/mo, zero ops**.

### Verdict

- **Video only:** self-host is a clear win (~$10–125 vs ~$200 cloud).
- **With recording:** self-host (~$150–430) ≈ cloud (~$200–300), but cloud wins
  on **ops** — no egress+Redis stack to run, patch, autoscale, and debug.

At 500 sessions/mo the recording ops burden usually isn't worth self-hosting.
**Recommended split: self-host the SFU, skip or keep recording on cloud.** (You
can't self-host the SFU *and* use cloud egress — egress binds to one server.
Pick one stack.)

---

## 4. Where to host (cheaper than AWS)

Bandwidth is the cost driver. Hyperscalers (AWS/GCP/Azure) charge ~$0.09/GB out;
budget hosts bundle terabytes. **But latency matters for live video** — pick a
region near your users (Mole = India).

| Host | Plan (2 vCPU/4 GB) | Bandwidth | ~$/mo | India PoP? |
|---|---|---|---|---|
| **Vultr** | 2 vCPU/4 GB | 3 TB | ~$18 | **Yes — Mumbai/Delhi** |
| **DigitalOcean** | 4 GB droplet | 4 TB | ~$24 | **Yes — Bangalore** |
| **Linode/Akamai** | 4 GB | 4 TB | ~$24 | **Yes — Mumbai** |
| **Oracle Cloud** | Ampere ARM (always-free) | 10 TB | **$0** | **Yes — Mumbai** (grab capacity) |
| **Hetzner Cloud** | CPX21 | 20 TB | ~$9 | No (EU/US) |
| **Contabo** | VPS S | 32 TB | ~$7 | No (EU/US/Asia SG) |
| **OVHcloud** | VPS Comfort | Unmetered | ~$12 | No (nearest: Singapore) |
| **AWS EC2** | c6i.large | 100 GB | ~$60 + bw | Yes — Mumbai |

**Recommendation for Mole (India users):**
1. **Try Oracle Cloud Ampere free tier (Mumbai)** — 4 vCPU / 24 GB ARM, 10 TB
   bandwidth, **$0**. LiveKit builds for ARM. If you can grab capacity, this is
   unbeatable.
2. **Otherwise Vultr Mumbai or DigitalOcean Bangalore** — ~$18–24/mo, low RTT for
   Indian students, ~5× cheaper than AWS, trivial ops.
3. **Stay on AWS EC2 (Mumbai)** only if you want everything co-located with the
   existing backend + S3 (one bill, one IAM, free network to S3). Pay the
   bandwidth premium for the ops simplicity.

> Hetzner/Contabo are the cheapest **per byte**, but EU/US-only → 150–250 ms RTT
> to India = noticeable lag on a live call. Not recommended as the primary SFU
> for Indian users.

---

## 5. Deploy guide — SFU (works on any host above)

Example: Ubuntu 22.04, 2 vCPU / 4 GB, region near your users.

### 1. Install Docker
```bash
curl -fsSL https://get.docker.com | sh
```

### 2. Generate keys + starter config
```bash
docker run --rm livekit/generate --local
# prints an API key, secret, and a sample livekit.yaml
```

### 3. `livekit.yaml`
```yaml
port: 7880
rtc:
  tcp_port: 7881
  port_range_start: 50000
  port_range_end: 60000
  use_external_ip: true      # REQUIRED on a cloud VM (server is behind a NAT'd public IP)
keys:
  <API_KEY>: <API_SECRET>
```

### 4. Run the server
```bash
docker run -d --restart unless-stopped \
  -p 7880:7880 -p 7881:7881 \
  -p 50000-60000:50000-60000/udp \
  -v $PWD/livekit.yaml:/livekit.yaml \
  livekit/livekit-server --config /livekit.yaml
```

### 5. Open the firewall — BOTH the cloud firewall and the host `ufw`
```
7880/tcp          signal (put behind TLS — see step 6)
7881/tcp          ICE/TCP fallback
50000-60000/udp   media  ← THE critical one; if UDP is closed, calls connect but show no video
```

### 6. TLS for `wss://` (browsers + mobile require a secure WebSocket)
Put **Caddy** in front — it auto-provisions Let's Encrypt:
```
livekit.yourdomain.com {
    reverse_proxy localhost:7880
}
```
Point a DNS **A record** at the server's public IP first.

### 7. Wire the backend
```env
LIVEKIT_URL=wss://livekit.yourdomain.com
LIVEKIT_API_KEY=<API_KEY>
LIVEKIT_API_SECRET=<API_SECRET>
```
Token-minting code is unchanged — same SDK, different URL/key/secret. Web and
mobile read the server URL from the token grant, so no frontend change.

### 8. Verify with two real devices on different networks
```bash
livekit-cli join-room \
  --url wss://livekit.yourdomain.com \
  --api-key <k> --api-secret <s> \
  --room test --identity tester
```

### Gotchas
- **UDP is mandatory.** Missing/closed UDP range = call connects, media silently
  fails. Always test with two devices on *different* networks (wifi + cellular).
- **`use_external_ip: true`** is required on any cloud VM.
- **Strict client NAT** (corporate/carrier) → add **TURN over TCP/443**. Enable
  LiveKit's embedded TURN in the config with a TLS cert, or run `coturn`
  alongside. Without it, some users on locked-down networks can't get media.

---

## 6. Local development

For local video-only testing (no recording), the dev-mode server is a one-liner:

```bash
docker run --rm -p 7880:7880 -p 7881:7881 -p 7882:7882/udp \
  -e LIVEKIT_KEYS="devkey: secret" \
  livekit/livekit-server --dev
```
- Server at `ws://localhost:7880`, fixed `devkey`/`secret`.
- Backend: `LIVEKIT_URL=ws://localhost:7880`, key `devkey`, secret `secret`.
- **A phone can't reach `localhost`** — use your machine's LAN IP
  (`ws://192.168.x.x:7880`) with the phone on the same wifi, and set
  `use_external_ip`/`--node-ip` to that LAN IP. A Cloudflare quick-tunnel is
  TCP-only, so WebRTC **media (UDP) dies over it** — signalling connects but no
  video. Use LAN or a host with open UDP for real device testing.

### 6a. Testing the Mole app against local LiveKit

Nothing in the app is Cloud-specific — the backend returns whatever `LIVEKIT_URL`
it's configured with. So testing self-hosted locally is just: point `.env` at the
local server and run a real session.

**Fastest loop — web, two browser windows (localhost media just works):**

1. Start the LiveKit dev server (section 6) and `pnpm backend` + `pnpm web`.
2. `backend/.env`: `LIVEKIT_URL=ws://localhost:7880`, `LIVEKIT_API_KEY=devkey`,
   `LIVEKIT_API_SECRET=secret`. Restart the backend so it reloads `.env`.
3. Log in as a **student** in one window and an **educator** in another (normal +
   incognito). Book → educator accepts → student pays → both open the session.
4. Each client hits `GET`-token on the backend, receives `ws://localhost:7880` +
   a join token, and connects. Expect two video tiles; `docker logs` on the
   livekit container shows `participant joined`.

**Real WebRTC path — phone on the same wifi:**

1. Find the host LAN IP (`hostname -I | awk '{print $1}'`), set
   `LIVEKIT_URL=ws://<LAN-IP>:7880` and `--node-ip <LAN-IP>` (or
   `use_external_ip` off + `rtc.node_ip`), restart backend.
2. Point the mobile app's API base at the same host:
   `EXPO_PUBLIC_API_URL=http://<LAN-IP>:3000` (token + media on one host).
3. Open host firewall: TCP 7880/7881 + the UDP media range.
4. Run a **dev build** (`npx expo run:android`) — LiveKit's native WebRTC module
   is absent in Expo Go.
5. Join from the phone + a laptop browser. Media flows over LAN UDP.

**Verify recording locally (egress, section 7):**

- Add the egress webhook to `livekit.yaml` so recording completion reaches the
  backend from inside Docker:
  ```yaml
  webhook:
    api_key: devkey
    urls:
      - http://host.docker.internal:3000/api/v1/webhooks/livekit
  ```
- Run a session a few seconds, end it (OTP exchange or the auto-complete cron).
  `docker logs` on the egress container shows start/stop; on stop LiveKit POSTs
  `egress_ended`, the backend finalizes the `SessionRecording` row and the MP4
  lands in the S3 bucket. Rows stuck in `processing` ⇒ the webhook isn't reaching
  the backend (check the URL host + that the backend is on 3000).

---

## 7. Recording (egress) — if you must self-host it

Egress needs the server + **Redis** + the egress container.

> **Local-disk recordings (dev).** The backend supports
> `LIVEKIT_RECORDING_STORAGE=local`: egress writes the MP4 to
> `LIVEKIT_RECORDING_DIR` (a Docker volume mapped onto the backend's
> `UPLOAD_DIR`) instead of S3, and playback is served through the `/api/v1/files`
> proxy — no S3 creds needed. Runnable stack + config live in the monorepo at
> `docker/livekit/` (`docker compose -f docker/livekit/docker-compose.yml up -d`).
> Dev only: the file proxy is unauthenticated, so the booking access-check is the
> sole gate. Prod stays `LIVEKIT_RECORDING_STORAGE=s3`.

S3-upload sketch (prod / non-local):

```yaml
# docker-compose.yml (SFU + Redis + egress)
services:
  redis:
    image: redis:7-alpine
    command: redis-server --save "" --appendonly no
    ports: ["6379:6379"]

  livekit:
    image: livekit/livekit-server
    command: --config /livekit.yaml
    volumes: ["./livekit.yaml:/livekit.yaml"]
    network_mode: host          # simplest for the UDP media range
    depends_on: [redis]

  egress:
    image: livekit/egress
    environment:
      EGRESS_CONFIG_FILE: /egress.yaml
    volumes: ["./egress.yaml:/egress.yaml"]
    cap_add: ["SYS_ADMIN"]       # headless Chrome sandbox
    network_mode: host
    depends_on: [redis, livekit]
```

`livekit.yaml` must point at Redis:
```yaml
redis:
  address: localhost:6379
```

`egress.yaml` (S3 upload target — same bucket/creds as the app's storage):
```yaml
redis:
  address: localhost:6379
api_key: <API_KEY>
api_secret: <API_SECRET>
ws_url: ws://localhost:7880
s3:
  access_key: <AWS_ACCESS_KEY_ID>
  secret: <AWS_SECRET_ACCESS_KEY>
  region: ap-south-1
  bucket: mole-media-prod
```

Notes:
- Egress is **CPU-hungry**; size the box for **peak concurrent recordings**, not
  total minutes. One recording ≈ 2–4 vCPU.
- Uploading to S3 from the same AWS region is free; from a non-AWS host, S3
  *ingress* is still free but the compute runs away from the bucket.
- The backend's egress-start code (`services/recording`, `controllers/recording`)
  is provider-agnostic — it just needs the SFU + egress reachable via the same
  API key/secret.

---

## 8. Recommendation summary

| Situation | Do this |
|---|---|
| Video/screen-share/chat only, cost-sensitive | **Self-host SFU** on Vultr/DO Mumbai (or Oracle free tier). ~$0–25/mo. |
| Want recording, low volume (~500/mo) | Keep **recording on LiveKit Cloud**, or skip it. Don't run egress yourself yet. |
| Everything co-located, ops-simplest | **AWS EC2 Mumbai** next to backend + S3. ~$125/mo. |
| Scaling to thousands of sessions | Revisit self-host egress + a bigger/dedicated SFU; bandwidth savings start to matter. |

No frontend or token-minting changes are needed to switch SFUs — only the three
`LIVEKIT_*` backend env vars.
