# Lunara — Streaming Service

## Context
Read `LUNARA_CONTEXT.md` at the start of every session. It is the single source of truth.

## Quick Reference

### SSH Access
```bash
ssh matamoros-local   # hmatamoros (192.168.1.200) — server
ssh lunara-relay      # Oracle VPS (129.153.159.140) — relay
```

### Key URLs
- Landing: https://lunara.video
- Streaming: https://stream.lunara.video (Jellyfin, port 8098)
- Invitations: https://invite.lunara.video (Wizarr, port 5690)
- NPM admin: http://192.168.1.200:8882 (LAN) or `ssh -L 8882:localhost:8882 matamoros-local`
- Uptime Kuma: http://192.168.1.200:3002
- Wizarr admin: http://localhost:5690 (via tunnel)

### Services on server
```bash
cd ~/lunara && docker compose ps          # check all Lunara services
docker logs jellyfin-clients --tail 50   # Jellyfin logs
docker logs wizarr-lunara --tail 50      # Wizarr logs
docker logs lunara-telegram-bot --tail 20 # Luna bot logs
systemctl status lunara-tunnel           # SSH relay tunnel
```

## Common Operations

### Create manual invitation (with plan enforcement)
```bash
# Standard, 30 days
curl -X POST https://invite.lunara.video/api/invitations \
  -H "X-API-Key: [from .tokens.db key=lunara-wizarr-api]" \
  -H "Content-Type: application/json" \
  -d '{"server_ids":[1],"library_ids":[2,4],"expires_in_days":7,"duration":"30","unlimited":false,"allow_downloads":false,"wizard_bundle_id":2,"max_active_sessions":2}'

# Plans: bundle_id 1=Basic(1stream), 2=Standard(2streams), 3=Premium(4streams+downloads)
# Duration: "30"=monthly, "90"=quarterly, "365"=annual, "36500"=forever
```

### Trigger full invite flow (payment simulation)
```bash
curl -X POST https://www.lunara.video/api/invite \
  -H "Content-Type: application/json" \
  -H "x-internal-key: [from Vercel env INTERNAL_API_KEY]" \
  -d '{"email":"user@example.com","plan":"standard","durationDays":30}'
```

### Reset user password in Jellyfin
- Go to http://192.168.1.200:8098 → Admin → Users → [user] → Reset Password

### Exempt user from plan enforcement
```
/lunara-exempt add <username>
/lunara-exempt remove <username>
/lunara-exempt list
```

### Force plan enforcement now
```bash
ssh matamoros-local "python3 ~/lunara/enforce_plans.py"
```

### Check/restart SSH relay tunnel
```bash
ssh matamoros-local "systemctl status lunara-tunnel"
ssh matamoros-local "sudo systemctl restart lunara-tunnel"
# Verify: ssh lunara-relay "ss -tlnp | grep 888"
```

### Update Jellyfin branding CSS
```bash
JF_TOKEN=$(ssh matamoros-local "cat ~/lunara/jf_token.txt")
curl -X POST http://192.168.1.200:8098/System/Configuration/branding \
  -H "X-MediaBrowser-Token: $JF_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"LoginDisclaimer":"...","CustomCss":"..."}'
```

## Persistence Notes

These patches are volume-mounted — they survive container recreates:
- Wizarr redirect fix: `~/lunara/wizarr-patches/wizard_routes.py`
- Jellyfin branding: `~/lunara/jellyfin-clients/index.html.patched`

If Wizarr or Jellyfin update to a new major version, re-apply patches.

## Stripe (Live)
- Dashboard: dashboard.stripe.com
- Secret key: in `.tokens.db` key=`lunara-stripe-live`
- Webhook: `https://www.lunara.video/api/webhooks/stripe`
- Customer portal: `lunara.video/manage`

## Backup
- Provider: Backrest → Backblaze B2 (plan: matamoros-daily)
- Schedule: 3am daily
- Includes: `~/lunara/` + ha-project + photos + nextcloud
- Status: http://192.168.1.200 (homepage widget)
