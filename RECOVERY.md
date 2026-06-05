# Lunara — Disaster Recovery Playbook

> Estimated recovery time from total server loss: **2–3 hours**

---

## What is backed up

| Data | Location on server | B2 path | How critical |
|---|---|---|---|
| Lunara configs + DB | `~/lunara/` | `b2:matamoros-backups/data/lunara/` | Critical |
| ha-project (HA, Nextcloud config) | `~/ha-project/` | `b2:matamoros-backups/data/ha-project/` | Critical |
| Photos (Immich) | `/mnt/data/photos/` | `b2:matamoros-backups/data/photos/` | Critical |
| Nextcloud data | `/mnt/data/nextcloud/data/` | `b2:matamoros-backups/data/nextcloud/data/` | Critical |
| Media (Movies/TV) | `/mnt/data/media/` | **NOT backed up** — re-download if lost | Low |

**Backup schedule:** Daily at 3am via Backrest → Backblaze B2
**Retention:** 7 daily / 4 weekly / 12 monthly
**Verify status:** http://192.168.1.200 (homepage widget) or http://192.168.1.200:9898

> ⚠️ **Important:** Backrest runs in Docker. Only paths explicitly mounted as volumes are backed up.
> `~/lunara/` is mounted as `/data/lunara:ro` inside the Backrest container.
> If you add new data directories to back up, you must add them as volume mounts in
> `~/ha-project/docker-compose.yml` AND update the path in `backrest/config/config.json`.

---

## Scenario 1 — Server still running, single service down

```bash
ssh matamoros-local

# Check which container is down
docker ps -a | grep -v "Up "

# Restart specific service
cd ~/lunara && docker compose up -d <service>
cd ~/ha-project && docker compose up -d <service>

# Check logs
docker logs <container> --tail 50
```

**Lunara SSH tunnel down (clients can't reach stream.lunara.video):**
```bash
ssh matamoros-local "sudo systemctl restart lunara-tunnel && systemctl status lunara-tunnel"
```

---

## Scenario 2 — Server needs reboot

```bash
ssh matamoros-local "sudo reboot"
# Wait ~2 min then verify
ssh matamoros-local "docker ps | wc -l"  # should be 30+
```

All containers have `restart: unless-stopped` — they come back automatically.
Check Uptime Kuma alerts on Telegram (Luna bot) to confirm.

---

## Scenario 3 — Full server loss (hardware failure)

### Step 1 — New Ubuntu 24.04 server

Install Ubuntu 24.04 LTS. Minimum specs:
- 6 cores / 8GB RAM / 500GB SSD
- AMD GPU for VAAPI transcoding (or Intel Quick Sync as fallback)

### Step 2 — Base setup

```bash
# Create user
sudo useradd -m -s /bin/bash amatamoros
sudo usermod -aG sudo amatamoros
sudo apt update && sudo apt install -y docker.io docker-compose-plugin git curl

# ZFS (if new drives available)
sudo apt install -y zfsutils-linux
sudo zpool create tank mirror /dev/sda /dev/sdc
sudo zfs create -o mountpoint=/mnt/data/media tank/media
sudo zfs create -o mountpoint=/mnt/data/photos tank/photos
# ... etc
```

### Step 3 — Restore from B2

```bash
# Install Backrest
docker run -d --name backrest-restore \
  -v /home/amatamoros:/restore \
  -e BACKREST_CONFIG=/config/config.json \
  garethgeorge/backrest:latest

# Or use restic directly
export B2_ACCOUNT_ID=0057935c53a1ca20000000001
export B2_ACCOUNT_KEY=K005v3ewfjnTdx1FcTTnadHnlhgFdmg
export RESTIC_REPOSITORY=b2:matamoros-backups
export RESTIC_PASSWORD="PHFZCPj/HxiO2hJGzVqc/Rl4s/22Bg8kdkZ0wel+JUQ="

# List snapshots
restic snapshots

# Restore latest
restic restore latest --target /restore/
```

### Step 4 — Restore Lunara

```bash
# After restoring ~/lunara/ from B2:
cd ~/lunara

# Copy .env from credentials.md (tokens.db backup)
# Start all services
docker compose up -d

# Re-apply Wizarr patch (if container was recreated fresh)
docker cp wizarr-patches/wizard_routes.py wizarr-lunara:/app/app/blueprints/wizard/routes.py

# Re-apply Jellyfin branding
docker cp jellyfin-clients/index.html.patched jellyfin-clients:/jellyfin/jellyfin-web/index.html
```

### Step 5 — SSH tunnel to Oracle VPS

```bash
# Regenerate SSH key for tunnel
ssh-keygen -t ed25519 -f ~/.ssh/lunara_tunnel -N ''

# Add new public key to Oracle VPS
ssh -i ~/.ssh/claude_agent ubuntu@129.153.159.140 \
  "echo '$(cat ~/.ssh/lunara_tunnel.pub)' >> ~/.ssh/authorized_keys"

# Start tunnel service
sudo systemctl enable --now lunara-tunnel
```

### Step 6 — Verify everything

```bash
# Check services
docker ps --format "table {{.Names}}\t{{.Status}}"

# Check Jellyfin
curl -s http://localhost:8098/health

# Check tunnel
systemctl status lunara-tunnel
ssh lunara-relay "ss -tlnp | grep 888"

# Check from internet
curl https://stream.lunara.video/health
```

---

## Key credentials location

All credentials in two places:
1. **`.tokens.db`** on server: `~/ha-project/.tokens.db` (backed up via B2)
2. **`~/lunara/credentials.md`** (backed up via B2)

Critical keys needed for recovery:
- Backblaze B2: keyID + applicationKey (in credentials.md)
- Oracle VPS SSH: `~/.ssh/claude_agent` (Mac) — backup this key separately
- Stripe Live: in `.tokens.db` key `lunara-stripe-live`
- Wizarr API key: in `.tokens.db` key `lunara-wizarr-api`

---

## Oracle VPS (relay) — separate recovery

Oracle VPS is independent of hmatamoros. If it goes down:
1. Log into Oracle Cloud → restart instance
2. Verify nginx is running: `ssh lunara-relay "systemctl status nginx"`
3. Verify iptables: `ssh lunara-relay "sudo iptables -L INPUT -n | grep ACCEPT"`

If VPS is lost permanently → provision new VM.Standard.E2.1.Micro (Always Free), follow original setup.

---

## DNS (Cloudflare)

If Oracle VPS IP changes:
```
Cloudflare → lunara.video DNS → update A records for stream/invite/pay
to new Oracle VPS IP
```

---

*Last updated: June 2026*
