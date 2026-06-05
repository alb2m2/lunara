# LUNARA — Architecture & Project Context

> This document is the single source of truth for the Lunara project.
> Claude Code should read this file at the start of every session before making any decisions.

---

## Project Overview

**Lunara** is a private, invite-only streaming service built on self-hosted infrastructure.
- Brand: Lunara — Premium Streaming
- Domain: `lunara.video` — registered on Cloudflare ✅
- Model: discrete, no public indexing (`noindex` on all pages)
- Access: by direct link only — no public signup, no SEO
- Legal entity: existing LLC with US Bank business account
- Language: English primary, Spanish toggle (Next.js i18n)

---

## Server — hmatamoros

| Property | Value |
|---|---|
| Hostname | hmatamoros |
| OS | Ubuntu 24.04.4 LTS |
| Kernel | 6.17.0-29-generic |
| CPU | Intel i5-10600K @ 4.1GHz (12 cores) |
| RAM | 16 GB DDR4 |
| GPU | AMD RDNA4 (XFX) — GFX12 / VCN v5.0, ~8GB VRAM |
| Network | 1 Gbps up/down |
| SSH | `ssh matamoros-local` (user: amatamoros) |
| Runtime | Docker (all services) |

### GPU Transcoding
- GPU: AMD RDNA4 (XFX) — GFX12 / VCN v5.0
- Method: VAAPI (AMD on Linux)
- Devices required: `/dev/dri/renderD128` (VAAPI) AND `/dev/dri/card0` (OpenCL — required for tone mapping)
- Kernel requirement: HWE 6.17+ — RDNA4 is not supported on 6.8 (already installed)
- GRUB parameter required: `amdgpu.dc=0` (already set on server)
- Tone mapping: **VPP (VAAPI)** — do NOT use OpenCL tone mapping, it requires card0 in the container and is unstable on RDNA4. Set `EnableTonemapping=false`, `EnableVppTonemapping=true`.
- Quick Sync (Intel iGPU) available as software fallback
- Estimated simultaneous streams with VAAPI (VCN v5.0):
  - 1080p transcode: 30–50
  - 4K → 1080p transcode: 10–20
  - 4K direct play: unlimited (client-side decode)

### Existing Services
- Jellyfin Personal — already running in Docker (do not modify)
- Other services may exist — do not touch containers not defined in this document

---

## Media Storage

| Path on host | Mount in container | Access |
|---|---|---|
| `/mnt/data/media/Movies` | `/media/Movies` | read-only |
| `/mnt/data/media/TV` | `/media/TV` | read-only |

- Base path on host: `/mnt/data/media` (ZFS dataset `tank/media`, 3.62 TB mirror)
- Verify or create subdirectories before first deploy: `mkdir -p /mnt/data/media/Movies /mnt/data/media/TV`
- Personal Jellyfin mounts `/mnt/data/media` directly — Lunara mounts subdirs read-only, no conflict
- Personal media is NOT mounted in the client Jellyfin instance
- Never mount personal media paths into Lunara containers

---

## Phase 1 Stack

All services run in Docker on `hmatamoros` unless noted otherwise.

### Services

| Service | Purpose | Subdomain | Port |
|---|---|---|---|
| Jellyfin (clients) | Streaming for paying clients | `stream.lunara.video` | 8098 (host), 8096 (internal) |
| Wizarr | User onboarding, invitations, expiration | `invite.lunara.video` | 5690 |
| Nginx Proxy Manager | SSL, reverse proxy, subdomain routing | — | 8880/8881 (via SSH tunnel) |
| Telegram Bot | Client support — Luna | t.me/airtorrent_bot | — |
| Uptime Kuma | Service monitoring + Telegram alerts | internal | 3002 |
| BTCPay Server | Crypto payment processing (Phase 2) | `pay.lunara.video` | 49392 |

### External Services

| Service | Purpose | Notes |
|---|---|---|
| Vercel | Hosts Next.js website | Connected to GitHub repo |
| Stripe | Primary payment processor | Neutral statement descriptor |
| Resend | Transactional email | Sender: `noreply@lunara.video` |
| Telegram Bot | Client support | Answer questions only |
| Cloudflare | Domain registrar + DNS | `lunara.video` |
| Backblaze B2 | Config and database backups | Daily via Backrest (plan: matamoros-daily) |

---

## Network & DNS

### Ingress Architecture — Oracle Cloud VPS + SSH Reverse Tunnel ✅ LIVE

```
Client → Oracle VPS 129.153.159.140 :80/:443
          ↓ nginx TCP stream proxy (localhost:8880/8881)
          SSH reverse tunnel (autossh, lunara-tunnel.service on hmatamoros)
          ↓
          NPM on hmatamoros :8880/:8881 → services (lunara Docker network)
```

| Component | Role | Details |
|---|---|---|
| Oracle Cloud VM.Standard.E2.1.Micro | Public IP 129.153.159.140, nginx TCP proxy | Free forever (Always Free) |
| `lunara-tunnel.service` | autossh reverse tunnel — hmatamoros → Oracle VPS | Systemd service, auto-restart |
| NPM (nginx-proxy-manager-lunara) | SSL via Let's Encrypt, routing to services | Ports 8880/8881/8882 on host |

- SSH: `ssh lunara-relay` (user: ubuntu, key: ~/.ssh/claude_agent)
- Tunnel key on hmatamoros: `~/.ssh/lunara_tunnel` (dedicated keypair)
- Real server and home IPs are never exposed to clients
- DNS A records on Cloudflare → 129.153.159.140
- Oracle Security List + NSG: ports 22, 80, 443 open; source port range must be blank (All)

**Restart tunnel if down:**
```bash
ssh matamoros-local "sudo systemctl restart lunara-tunnel"
```

**Check tunnel health:**
```bash
ssh matamoros-local "systemctl status lunara-tunnel"
ssh lunara-relay "ss -tlnp | grep 888"  # should show 8880, 8881 listening
```

### Subdomains
```
lunara.video          → Vercel (Next.js website)
stream.lunara.video   → Jellyfin clients
invite.lunara.video   → Wizarr
pay.lunara.video      → BTCPay Server
```

### SSL
- Nginx Proxy Manager handles SSL via Let's Encrypt for all server subdomains
- Vercel handles SSL for `lunara.video` automatically

---

## Jellyfin — Client Instance

### Configuration
- Separate instance from personal Jellyfin
- Different port, different config directory, different database
- Users are hidden from each other (privacy enforced at Jellyfin level)
- VAAPI hardware transcoding enabled via `/dev/dri/renderD128`

### User Templates (one per plan)
Claude Code must create three user policy templates in Jellyfin:

| Template | Max Bitrate | Max Resolution | Downloads | Screens |
|---|---|---|---|---|
| `plan_basic` | 8 Mbps | 1080p | No | 1 |
| `plan_standard` | 25 Mbps | 4K | No | 2 |
| `plan_premium` | unlimited | 4K HDR | Yes | 4 |

- Wizarr applies the correct template at user creation based on plan purchased
- Limits are enforced server-side — client cannot bypass them

### Docker Compose snippet
```yaml
jellyfin-clients:
  image: jellyfin/jellyfin:latest
  container_name: jellyfin-clients
  group_add:
    - render
    - video
  environment:
    - JELLYFIN_PublishedServerUrl=https://stream.lunara.video
  devices:
    - /dev/dri/renderD128:/dev/dri/renderD128
  volumes:
    - ./jellyfin-clients/config:/config
    - ./jellyfin-clients/cache:/cache
    - /mnt/data/media/Movies:/media/Movies:ro
    - /mnt/data/media/TV:/media/TV:ro
  ports:
    - "8098:8096"
  networks:
    - lunara
  restart: unless-stopped
```

- Port 8096 is occupied by personal Jellyfin — client instance runs on 8098
- `group_add: [render, video]` required for VAAPI access on Linux
- `network_mode: host` replaced with bridge network `lunara` to avoid port 8096 conflict
- NPM proxies `stream.lunara.video` → `http://jellyfin-clients:8096` (container-to-container, internal)

### VAAPI Configuration (post first-start)

1. Open Jellyfin admin: `http://[server-ip]:8098/web/#/dashboard`
2. Dashboard → Administration → Playback → Transcoding
3. Hardware acceleration: **Video Acceleration API (VAAPI)**
4. VA API Device: `/dev/dri/renderD128`
5. Enable codecs: H.264, HEVC, AV1 (all supported by VCN v5.0)
6. Enable Tone Mapping (for HDR → SDR when client doesn't support HDR)
7. Save and restart Jellyfin if prompted

### User Policy Templates

Jellyfin has no named template system natively. Workflow with Wizarr:
- Create one test user per plan via admin UI, configure limits, verify enforcement
- Wizarr passes Jellyfin user policy parameters via API at invitation creation time
- Configure three Wizarr invitation types (`basic` / `standard` / `premium`), each mapped to correct policy

---

## Wizarr

- Manages invitations with expiration dates
- One invitation template per plan (basic, standard, premium)
- Each template maps to the corresponding Jellyfin user policy
- When Stripe webhook fires, Next.js API calls Wizarr API to generate invitation
- Invitation link sent to client via Resend email

---

## Plans & Pricing

| Plan | Monthly | Quarterly | Annual | Quality | Screens | Downloads |
|---|---|---|---|---|---|---|
| Basic | TBD | TBD | TBD | 1080p | 1 | No |
| Standard | TBD | TBD | TBD | 4K | 2 | No |
| Premium | TBD | TBD | TBD | 4K HDR | 4 | Yes |

> Pricing amounts to be defined by owner before launch.

---

## Payment Flow

### Stripe
- Primary processor
- Statement descriptor: neutral (e.g. "DIGITAL SERVICES") — not "Lunara"
- Products: Basic, Standard, Premium
- Prices: monthly, quarterly, annual per product
- Funds deposited to LLC US Bank business account
- Webhook: `POST /api/webhooks/stripe` on Next.js (Vercel)

### BTCPay Server
- Secondary processor for crypto
- Accepts: Bitcoin (BTC), Monero (XMR)
- Wallet setup: part of initial deployment
- Self-hosted on `hmatamoros`
- Webhook: `POST /api/webhooks/btcpay` on Next.js (Vercel)

### Payment Policies
- **Grace period:** 7 days after failed payment before Wizarr disables account. Stripe retries automatically during this period.
- **Refund window:** 48 hours from access activation (not from payment date)
- **Plan upgrade:** immediate, prorated via Stripe
- **Plan downgrade:** applied at next billing cycle

### Full Payment Flow
```
Client visits lunara.video (direct link only)
→ Selects plan and billing cycle
→ Pays via Stripe or BTCPay
→ Webhook fires to Next.js API
→ Next.js calls Wizarr API with correct plan template
→ Wizarr creates Jellyfin user with plan limits
→ Wizarr generates invitation link with expiration
→ Resend sends welcome email with link to client
→ Client registers and gets access
```

---

## Next.js Website (Vercel)

### Purpose
- Private landing page — not a public marketing site
- `noindex` on all pages — invisible to search engines
- No public contact form
- Access only via direct link shared by owner

### Pages
- `/` — Hero + plans + pricing
- `/checkout` — Stripe / BTCPay payment flow
- `/success` — post-payment confirmation
- `/login` — redirect to `stream.lunara.video`

### API Routes
- `/api/webhooks/stripe` — handles Stripe events, calls Wizarr
- `/api/webhooks/btcpay` — handles BTCPay events, calls Wizarr
- `/api/invite` — internal route to create Wizarr invitation

### Tech Stack
- Framework: Next.js (App Router)
- Styling: Tailwind CSS
- Payments: Stripe SDK + BTCPay REST API
- Email: Resend SDK
- i18n: Next.js built-in (English primary, Spanish toggle)
- Deploy: Vercel (auto-deploy from GitHub)

### Design
- Dark background — deep space aesthetic
- Color palette: Lunara Violet `#4b3d8f`, Stellar Lilac `#9b85d4`, Moonlight `#c8bcf0`
- Typography: Cormorant Garamond (display), Inter (UI)
- Hero: desolate desert landscape with dark violet overlay
- Tone: premium, minimal, cinematic

---

## Email — Resend

- Sender domain: `lunara.video` — verified on Resend ✅
- Sender address: `noreply@lunara.video`
- Triggered emails:
  - Welcome + invitation link (on successful payment)
  - Expiration warning (7 days before access ends)
  - Payment failed notice
  - Refund confirmation

---

## Telegram Bot

- Purpose: client support only
- Scope: answer questions, no account management actions
- Client-facing — linked from welcome email and success page
- Bot hosted on `hmatamoros` in Docker

---

## Monitoring — Uptime Kuma

- Monitors all internal services: Jellyfin clients, Wizarr, Nginx, BTCPay
- Alerts via Telegram when any service goes down
- Internal access only (not exposed publicly)

---

## Backup — Rclone + Backblaze B2

### What is backed up
- Jellyfin clients config and database
- Wizarr database
- BTCPay Server data
- Nginx Proxy Manager config
- Docker compose files and `.env` files

### What is NOT backed up
- Media files (tank) — no backup needed per owner decision

### Schedule
- Daily automated backup via cron + Rclone
- Destination: Backblaze B2 bucket `lunara-backups`
- B2 account setup is part of initial deployment

---

## Live Credentials & Keys

All secrets in `~/lunara/credentials.md` on server and in `.tokens.db` (`ha-project/.tokens.db`).

| Service | .tokens.db key | Notes |
|---|---|---|
| Stripe Live | `lunara-stripe-live` | `sk_live_...` — switch from test before launch |
| Stripe Webhook | `lunara-stripe-live-webhook` | `whsec_...` — Live webhook secret |
| Wizarr API key | `lunara-wizarr-api` | `X-API-Key` header for Next.js webhook |
| Resend | `lunara-resend` | `re_...` — lunara.video verified |
| NPM admin | `lunara-npm` | http://192.168.1.200:8882 |
| Jellyfin admin | `lunara-jellyfin` | user: root, http://192.168.1.200:8098 |
| Telegram bot | `lunara-telegram-bot` | @airtorrent_bot (display name: Luna) |
| Uptime Kuma | `lunara-uptime-kuma` | user: root, http://192.168.1.200:3002 |
| Wizarr admin | `lunara-wizarr` | user: root, https://invite.lunara.video |

---

## Plan Enforcement

Plans are enforced automatically via cron (`*/5 * * * *` on hmatamoros):

```bash
/home/amatamoros/lunara/enforce_plans.py
```

| Plan | wizard_bundle_id | Streams | Bitrate | Downloads | Expiry |
|---|---|---|---|---|---|
| Basic | 1 | 1 | 8 Mbps | No | Per billing cycle |
| Standard | 2 | 2 | 25 Mbps | No | Per billing cycle |
| Premium | 3 | 4 | Unlimited | Yes | Per billing cycle |
| Basic Unlimited | 4 | 1 | 8 Mbps | No | Never — manual invite only |

**Basic Unlimited** — tier para invitados especiales, familia o promos sin fecha de vencimiento. No pasa por Stripe. Mismos límites técnicos que Basic. Para crear invitación de este tipo usar `wizard_bundle_id=4` y `duration=36500` (100 años) via API.

- Users with no plan in Wizarr get **Basic as default**
- Admin users (`IsAdministrator: True`) are always skipped
- Exempt users list: `EXEMPT_USERS = {"root", "em_admin"}` in the script
- Add/remove exempt users with `/lunara-exempt add|remove|list <username>`

---

## Wizarr — Key Notes

- Wizard bundle IDs: 1=Basic, 2=Standard, 3=Premium (in Wizarr SQLite DB)
- After wizard completion, redirects to `https://stream.lunara.video` (patched routes.py)
- Patch is volume-mounted: `./wizarr-patches/wizard_routes.py:/app/app/blueprints/wizard/routes.py:ro`
- If Wizarr image updates to a new major version, re-apply the patch
- Manual invitations via API (to include wizard_bundle_id + max_active_sessions):
  ```bash
  curl -X POST https://invite.lunara.video/api/invitations \
    -H "X-API-Key: [key]" \
    -d '{"server_ids":[1],"library_ids":[2,4],"expires_in_days":7,"duration":"30","wizard_bundle_id":2,"max_active_sessions":2}'
  ```

---

## Jellyfin — Branding

- Server name: "Lunara" (via API)
- CSS: custom header `✦ Lunara` — configured in Dashboard → Administration → Branding
- Title tag: "Lunara" — via patched `index.html` (volume-mounted :ro)
- Favicon: ✦ star in violet on dark bg — embedded SVG data URL in index.html
- Login disclaimer: "Lunara — Premium Streaming"
- Patch persists via: `./jellyfin-clients/index.html.patched:/jellyfin/jellyfin-web/index.html:ro`

---

## Stripe — Live Configuration

- Account: `acct_1TeSNL0nJk2QlJyO`
- Webhook: `we_1Teeqw0nJk2QlJyO4issac52` → `https://www.lunara.video/api/webhooks/stripe`
- Events: `checkout.session.completed`, `invoice.payment_succeeded`, `invoice.payment_failed`, `charge.refunded`
- Customer Portal: `bpc_1TeigT0nJk2QlJyOuOqsJ30Y` — accessible at `lunara.video/manage`
- 18 live prices: 3 plans × 3 cycles × 2 regions (LATAM/USA)
- Renewal flow: `invoice.payment_succeeded` → find Wizarr user by email → extend access
- Cancellation: automatic — Wizarr expires user at end of paid period

---

## Telegram Bot — Luna

- Bot: @airtorrent_bot (display name: Luna, icon: ✦ purple star)
- Token: in `.tokens.db` key `lunara-telegram-bot`
- Admin Telegram ID: 5025498874 (Alberto)
- Container: `lunara-telegram-bot` on hmatamoros
- Deep link: `t.me/airtorrent_bot?start=welcome` (auto-triggers welcome message)
- Trilingüe: English / Español / Português

**Commands:**
- `/start` / `/help` → welcome message with instructions
- `/request` → search Jellyseerr, show results, create request as `luna-bot` (Pending — admin approves)
- `/cancel` → cancel active request flow

**Message routing:**
- Client message → forwarded to Alberto with client name + ID
- Alberto replies to forwarded message → bot sends back to client

**Jellyseerr integration:**
- URL: `http://192.168.1.200:5055`
- API key: `.tokens.db` key `lunara-jellyseerr`
- Requests created as user `luna-bot` (id=4, no auto-approve) → go to Pending
- TV shows: `seasons: "all"` — requests all seasons
- Admin approves manually in Jellyseerr UI

---

## Backup

- Provider: Backrest + Backblaze B2 (existing plan `matamoros-daily`)
- `~/lunara/` included in backup path
- Schedule: 3am daily, retention 7d/4w/12m
- Backup status: homepage widget on hmatamoros

---

## Phase 2 (Future)

Do not implement in Phase 1. Document only for planning purposes.

| Addition | Purpose | Subdomain |
|---|---|---|
| Jellyseerr | Client content requests | `requests.lunara.video` |
| Authentik | Centralized identity provider (SSO) | `auth.lunara.video` |

### Phase 2 Notes
- Wizarr will create users in Authentik instead of directly in Jellyfin
- Authentik will provide SSO for both Jellyfin and Jellyseerr
- Client login experience stays the same — one set of credentials for all services
- Jellyseerr login will use Jellyfin credentials natively until Authentik is added

### Phase 3 (Hardware Scale)
- Trigger: sustained load above 50 simultaneous streams
- Action: evaluate additional server or dedicated transcoding node
- Current AMD RDNA4 (VCN v5.0) handles Phase 1 and most of Phase 2 comfortably

---

## Capacity Planning

| Scenario | Simultaneous Streams | Status |
|---|---|---|
| Phase 1 target | 25 | Comfortable |
| Phase 1 max | 50 | Manageable |
| Phase 2 growth | 50–80 | GPU handles it |
| Phase 3 trigger | 80+ sustained | Evaluate new hardware |

---

## Security Notes

- Jellyfin client instance: users cannot see other users
- Media mounted read-only — clients cannot modify files
- No personal media exposed to client instance
- BTCPay: self-hosted, no third-party custody of crypto
- Stripe: neutral descriptor protects brand discretion
- Website: noindex, no public links, direct access only
- Oracle VPS relay: hides home/server IP — clients only see 129.153.159.140

---

## Project Structure on Server

```
~/lunara/
├── docker-compose.yml
├── .env                          # secrets — never commit to git
├── .env.template                 # placeholder values — safe to commit
├── credentials.md                # all service credentials (local only)
├── enforce_plans.py              # plan enforcement cron (every 5 min)
├── jf_token.txt                  # Jellyfin API token for cron script
├── jellyfin-clients/
│   ├── config/                   # Jellyfin config (SSD)
│   ├── cache/                    # transcode cache (SSD)
│   └── index.html.patched        # Lunara branding (volume-mounted :ro)
├── npm/
│   ├── data/                     # NPM proxy hosts, SSL config
│   └── letsencrypt/              # Let's Encrypt certs (auto-renewed by NPM)
├── wizarr/
│   └── database/                 # SQLite DB with users, invitations
├── wizarr-patches/
│   └── wizard_routes.py          # Patched: redirects to stream.lunara.video (volume-mounted :ro)
├── telegram-bot/
│   ├── bot.py                    # Luna support bot
│   ├── Dockerfile
│   └── data/sessions.json        # admin↔client message routing
├── uptime-kuma/
│   └── data/
└── btcpay/                       # BTCPay — Phase 2, deploy separately
```

---

## Full docker-compose.yml (Phase 1)

```yaml
# ~/lunara/docker-compose.yml

networks:
  lunara:
    driver: bridge

services:

  # Reverse proxy + SSL termination
  # Traffic: Oracle VPS :80/443 → WireGuard wg-relay → hmatamoros :8880/:8881 → NPM → services
  # Admin UI: http://192.168.1.200:8882 (LAN) or ssh -L 8882:localhost:8882 matamoros-local
  nginx-proxy-manager:
    image: jc21/nginx-proxy-manager:latest
    container_name: nginx-proxy-manager-lunara
    volumes:
      - ./npm/data:/data
      - ./npm/letsencrypt:/etc/letsencrypt
    ports:
      - "8880:80"    # HTTP  — Oracle VPS TCP-proxies :80  → here via WireGuard
      - "8881:443"   # HTTPS — Oracle VPS TCP-proxies :443 → here via WireGuard
      - "8882:81"    # Admin UI — LAN access only
    networks:
      - lunara
    restart: unless-stopped

  # Jellyfin client instance — port 8098 on host (8096 taken by personal Jellyfin)
  jellyfin-clients:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin-clients
    group_add:
      - render
      - video
    environment:
      - JELLYFIN_PublishedServerUrl=https://stream.lunara.video
    devices:
      - /dev/dri/renderD128:/dev/dri/renderD128
    volumes:
      - ./jellyfin-clients/config:/config
      - ./jellyfin-clients/cache:/cache
      - /mnt/data/media/Movies:/media/Movies:ro
      - /mnt/data/media/TV:/media/TV:ro
    ports:
      - "8098:8096"
    networks:
      - lunara
    restart: unless-stopped

  # Wizarr — invitation management and user onboarding
  wizarr:
    image: ghcr.io/wizarrrr/wizarr:latest
    container_name: wizarr
    environment:
      - APP_URL=https://invite.lunara.video
    volumes:
      - ./wizarr/database:/data/database
    networks:
      - lunara
    restart: unless-stopped

  # Uptime Kuma — Lunara service monitoring (port 3002 on host to avoid potential conflicts)
  uptime-kuma-lunara:
    image: louislam/uptime-kuma:latest
    container_name: uptime-kuma-lunara
    volumes:
      - ./uptime-kuma/data:/app/data
    ports:
      - "3002:3001"
    networks:
      - lunara
    restart: unless-stopped

  # BTCPay Server — complex multi-container stack (nbxplorer + bitcoin node or external)
  # Deploy separately via its own compose in ./btcpay/ following https://docs.btcpayserver.org/Docker/
  # After setup: NPM proxy pay.lunara.video → http://localhost:49392
```

---

## .env.template

The `.env` file on the server is minimal — Telegram bot token is in `docker-compose.yml` directly (as env var in the service). All payment/email secrets live in Vercel environment variables (not on the server).

---

## Coexistence with Existing Services

hmatamoros runs a personal stack in `~/ha-project/`. Lunara runs in `~/lunara/`. They must not interfere.

### Ports in use — do not rebind

| Port | Existing personal service |
|---|---|
| 80 | Homepage |
| 3001 | reserved |
| 4533 | Navidrome |
| 5055 | Jellyseerr |
| 8080 | qBittorrent |
| 8095 | Music Assistant (web UI) |
| 8096 | Jellyfin (personal) |
| 8097 | Music Assistant (stream server) |
| 8123 | Home Assistant production |
| 8124 | Home Assistant testing |
| 8443 | Nextcloud |
| 8888 | Vaultwarden |
| 9000 | Portainer |
| 51820 | WireGuard |
| 51821 | wg-easy admin |

### Lunara ports on host

| Port | Lunara service | Notes |
|---|---|---|
| 8098 | Jellyfin clients | direct debug access; NPM is the production path |
| 3002 | Uptime Kuma Lunara | internal monitoring |
| 49392 | BTCPay Server | NPM proxies pay.lunara.video here |

### Rules
- Never modify containers in `~/ha-project/`
- Never touch `gluetun-qbt` — it is the qBittorrent kill switch
- Mount `/mnt/data/media` subdirs only — never the full path

---

## Pre-Deploy Checklist

### AirVPN panel (airvpn.org/client)
- [ ] Config Generator → generate WireGuard config for a second device slot
- [ ] Port Forwarding → add port 80
- [ ] Port Forwarding → add port 443
- [ ] Note the public IP assigned to the second device — used for all Cloudflare A records

### Cloudflare DNS
- [ ] Register `lunara.video` (or confirm registered)
- [ ] A record: `stream.lunara.video` → AirVPN public IP (gluetun-lunara device)
- [ ] A record: `invite.lunara.video` → same AirVPN public IP
- [ ] A record: `pay.lunara.video` → same AirVPN public IP
- [ ] `lunara.video` → Vercel (configured automatically by Vercel on project creation)

### Server
- [ ] `mkdir -p ~/lunara`
- [ ] Copy `.env.template` → `.env`, fill in WireGuard keys
- [ ] Verify or create media subdirectories:
  ```bash
  ls /mnt/data/media/
  # if Movies/ and TV/ don't exist:
  mkdir -p /mnt/data/media/Movies /mnt/data/media/TV
  ```
- [ ] Verify render group: `getent group render`
- [ ] Verify VAAPI device: `ls -la /dev/dri/`
- [ ] Verify kernel (must be HWE 6.17+): `uname -r`

### First boot
- [ ] `cd ~/lunara && docker compose up -d gluetun-lunara`
- [ ] Verify VPN tunnel: `docker logs gluetun-lunara | grep "VPN is up"`
- [ ] `docker compose up -d` (remaining services)
- [ ] Access NPM admin via SSH tunnel:
  ```bash
  ssh -L 9081:localhost:81 matamoros-local
  # browse to http://localhost:9081
  # default credentials: admin@example.com / changeme
  ```
- [ ] Configure proxy hosts in NPM and request Let's Encrypt SSL for all subdomains
- [ ] Configure VAAPI in Jellyfin admin (see VAAPI Configuration section above)
- [ ] Connect Wizarr to Jellyfin API and create three invitation templates

---

## Infrastructure Philosophy

### Phase 1 — Single Server
All services run on `hmatamoros`. This is intentional and sufficient for Phase 1.

The critical separation is already in place:
- **Vercel** hosts the website and payment flow — always up, independent of the server
- If `hmatamoros` goes down, clients can still visit `lunara.video` and pay
- Payments queue and access is provisioned once the server is back up
- Uptime Kuma alerts the owner immediately via Telegram if any service fails

### When to Add a Second Machine
- Sustained load above 40 simultaneous streams
- Server requires frequent maintenance windows
- More than 50 active clients
- Desire to separate payment/onboarding stack from streaming stack

---

## Deployment Order (Phase 1)

Claude Code should follow this sequence:

1. Register `lunara.video` on Cloudflare
2. Generate second AirVPN WireGuard config (second device slot) — add ports 80 and 443 to port forwarding
3. Point Cloudflare DNS A records to AirVPN public IP of the gluetun-lunara device
4. Deploy gluetun-lunara + Nginx Proxy Manager (docker compose up)
5. Deploy Jellyfin clients container with VAAPI and read-only media mounts (port 8098)
6. Configure Jellyfin: user privacy, three plan templates, hardware transcoding
7. Deploy Wizarr and connect to Jellyfin API
8. Configure Wizarr invitation templates per plan
9. Deploy BTCPay Server and configure BTC + XMR wallets
10. Set up Backblaze B2 bucket and Rclone backup schedule
11. Deploy Uptime Kuma and configure Telegram alerts
12. Deploy Telegram support bot
13. Create Next.js project, connect to Vercel
14. Implement Stripe products and prices (Basic, Standard, Premium × 3 cycles)
15. Implement webhook handlers (Stripe → Wizarr, BTCPay → Wizarr)
16. Implement Resend transactional emails
17. Configure i18n (English primary, Spanish toggle)
18. Set all pages to noindex
19. End-to-end test: payment → invitation → registration → stream

---

*Last updated: June 2026 — Phase 1 complete*
*Owner: Lunara LLC — Alberto Matamoros*
