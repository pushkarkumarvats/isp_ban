# Disaster Recovery Playbook

## When to Use This Playbook

This playbook is for **acute ISP block events**   when your app stops working for a segment of users without any code changes on your end.

**Symptoms that trigger this playbook:**

- User reports: "App is down" from specific ISPs (Jio, Airtel, BSNL, etc.)
- Error monitoring shows `net::ERR_NAME_NOT_RESOLVED` or `ERR_CONNECTION_REFUSED`
- Supabase works fine from developer machines but not from affected users
- Works on WiFi but not on mobile data (same phone)

---

## Phase 1: Confirm the Block (5 Minutes)

### Step 1.1: Verify with Affected Users

```
Questions to ask affected user:
1. Which ISP / carrier? (Jio, Airtel, BSNL, Vodafone)
2. WiFi or mobile data?
3. Does a VPN fix it?
4. Which region? (city/state)
```

### Step 1.2: Remote Diagnosis (Run from local machine)

```bash
# Test what ISP DNS returns
nslookup your-project.supabase.co

# Test Cloudflare DNS (gold standard)
nslookup your-project.supabase.co 1.1.1.1

# If the IPs differ, it's a DNS block
# If the IPs are the same but connection fails, it's an IP block

# Test connectivity over proxy
curl -v https://your-proxy.yourproduct.com/rest/v1/health
```

### Step 1.3: Automated Status Check Script

```bash
#!/bin/bash
# check-connectivity.sh   run on a VM in affected region

SUPABASE_URL="your-project.supabase.co"
PROXY_URL="your-proxy.yourproduct.com"

echo "=== ISP Block Diagnosis ==="
echo "Testing direct Supabase access..."
if curl -s --connect-timeout 5 -o /dev/null -w "%{http_code}" \
    "https://${SUPABASE_URL}/rest/v1/health" | grep -q "200"; then
  echo "  ✓ Direct Supabase: OK"
else
  echo "  �  Direct Supabase: BLOCKED"
fi

echo "Testing proxy access..."
if curl -s --connect-timeout 5 -o /dev/null -w "%{http_code}" \
    "https://${PROXY_URL}/rest/v1/health" | grep -q "200"; then
  echo "  ✓ Proxy: OK"
else
  echo "  �  Proxy: BLOCKED"
fi

echo "DNS check (Cloudflare DoH)..."
CF_IP=$(curl -s "https://cloudflare-dns.com/dns-query?name=${SUPABASE_URL}&type=A" \
    -H "accept: application/dns-json" | jq -r '.Answer[0].data')
ISP_IP=$(nslookup $SUPABASE_URL 2>/dev/null | awk '/Address: /{print $2}' | tail -1)

echo "  Cloudflare DNS: ${CF_IP}"
echo "  ISP DNS:        ${ISP_IP}"

if [ "$CF_IP" != "$ISP_IP" ]; then
  echo "  → DNS POISONING DETECTED"
else
  echo "  → DNS is consistent"
fi
```

---

## Phase 2: Switch to Proxy (10 Minutes)

If proxy is already deployed (see file `02_cloudflare_and_alternatives.md`), switch is instant:

### Step 2.1: Enable Proxy via Feature Flag

```typescript
// lib/config.ts   simple feature flag via env var
let SUPABASE_URL = process.env.NEXT_PUBLIC_SUPABASE_URL!
let SUPABASE_KEY = process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!

// Override with proxy if ISP ban is active
if (process.env.NEXT_PUBLIC_USE_PROXY === 'true') {
  SUPABASE_URL = process.env.NEXT_PUBLIC_SUPABASE_PROXY_URL!
}

export { SUPABASE_URL, SUPABASE_KEY }
```

In Vercel dashboard: add `NEXT_PUBLIC_USE_PROXY = true` → Redeploy.  
Total time: ~2 minutes.

### Step 2.2: Redeploy with Proxy URL

```bash
# Vercel
vercel env add NEXT_PUBLIC_SUPABASE_URL production
# Enter: https://your-proxy.yourproduct.com
vercel --prod

# Railway
railway variables set VITE_API_URL=https://your-proxy.yourproduct.com
railway up

# Fly.io
fly secrets set SUPABASE_URL=https://your-proxy.yourproduct.com
fly deploy
```

---

## Phase 3: Deploy Standby Proxy (20 Minutes)

If no proxy was pre-deployed, deploy now:

### Step 3.1: Deploy Cloudflare Worker (Fastest)

```bash
# Clone the worker template
git clone https://github.com/yourorg/cf-proxy worker
cd worker

# Set secrets
npx wrangler secret put SUPABASE_URL
# Enter: https://your-project.supabase.co

# Deploy
npx wrangler deploy

# Add custom domain
npx wrangler custom-domains add your-proxy.yourproduct.com
```

Full Worker code is in `02_cloudflare_and_alternatives.md`.

### Step 3.2: Quick Nginx Proxy on Hetzner

```bash
# Create VPS (shortest path   use Hetzner Cloud API or UI)
# CX11 in Bangalore = 3.49 EUR/month

ssh root@NEW_VPS_IP

apt update && apt install -y nginx certbot python3-certbot-nginx

cat > /etc/nginx/conf.d/supabase-proxy.conf << 'EOF'
server {
    listen 80;
    server_name your-proxy.yourproduct.com;
    
    location / {
        proxy_pass https://your-project.supabase.co;
        proxy_set_header Host your-project.supabase.co;
        proxy_ssl_server_name on;
    }
}
EOF

systemctl restart nginx

certbot --nginx -d your-proxy.yourproduct.com --non-interactive --agree-tos -m admin@yourproduct.com

systemctl reload nginx
echo "Proxy ready at https://your-proxy.yourproduct.com"
```

---

## Phase 4: IP Rotation (30-60 Minutes)

If the proxy IP itself is blocked:

```bash
# Hetzner: Assign floating IP via API
curl -X POST "https://api.hetzner.cloud/v1/floating_ips" \
  -H "Authorization: Bearer $HETZNER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"home_location": "blr", "type": "ipv4"}' | jq '.floating_ip.ip'

# Assign to server
curl -X POST "https://api.hetzner.cloud/v1/floating_ips/${FLOATING_IP_ID}/actions/assign" \
  -H "Authorization: Bearer $HETZNER_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"server\": ${SERVER_ID}}"

# Update DNS record in Cloudflare
curl -X PATCH "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/dns_records/${RECORD_ID}" \
  -H "Authorization: Bearer $CF_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"type\": \"A\", \"name\": \"your-proxy\", \"content\": \"${NEW_IP}\", \"ttl\": 60, \"proxied\": false}"
```

---

## Phase 5: Warm Cache and Notify (Ongoing)

### Cache warmup script:

```bash
#!/bin/bash
# warm-cache.sh
PROXY="https://your-proxy.yourproduct.com"

# Warm common API endpoints
endpoints=(
  "/rest/v1/products?select=id,name&limit=50"
  "/rest/v1/categories?select=*"
  "/rest/v1/settings?select=*"
)

for endpoint in "${endpoints[@]}"; do
  status=$(curl -s -w "%{http_code}" -o /dev/null "${PROXY}${endpoint}" \
    -H "apikey: ${SUPABASE_ANON_KEY}")
  echo "Warmed ${endpoint}: HTTP ${status}"
done
```

### Status page update:

```bash
# Statuspage.io API
curl -X POST "https://api.statuspage.io/v1/pages/${PAGE_ID}/incidents" \
  -H "Authorization: OAuth ${STATUSPAGE_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "incident": {
      "name": "Connectivity issues for some ISP users",
      "status": "investigating",
      "body": "We are investigating connectivity issues reported by users on some ISPs. A workaround is being deployed.",
      "impact_override": "minor"
    }
  }'
```

---

## Monitoring: Automated Detection

Set up a synthetic monitor that runs from affected ISP regions:

```yaml
# UptimeRobot / Better Uptime config
monitors:
  - name: "Direct Supabase (India)"
    url: "https://your-project.supabase.co/rest/v1/health"
    interval: 1  # minute
    locations: ["Mumbai", "Bangalore"]
    alert_threshold: 2  # Alert after 2 failures
    alert_channels: ["slack-engineering", "pagerduty"]

  - name: "Proxy (India)"
    url: "https://your-proxy.yourproduct.com/rest/v1/health"
    interval: 1
    locations: ["Mumbai", "Bangalore"]
```

**Cloudflare Worker health check:**

```typescript
// health.worker.ts   runs every minute via Cron Trigger
export default {
  async scheduled(event: ScheduledEvent, env: Env, ctx: ExecutionContext) {
    const checks = [
      { name: 'supabase-direct', url: `https://${env.SUPABASE_HOST}/rest/v1/health` },
      { name: 'supabase-proxy', url: `${env.PROXY_URL}/rest/v1/health` },
    ]
    
    for (const check of checks) {
      const ok = await fetch(check.url).then(r => r.ok).catch(() => false)
      if (!ok) {
        await notifySlack(env.SLACK_WEBHOOK, `🚨 ${check.name} is DOWN`)
      }
    }
  },
}

async function notifySlack(webhook: string, text: string) {
  await fetch(webhook, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ text }),
  })
}
```

---

## Runbook

```
T+0   Block detected (monitoring alerts or user report)
T+5   Confirm block type (DNS / IP / DPI)
T+10  Activate pre-deployed proxy (Vercel env var change + redeploy)
T+20  Verify proxy is working for affected users
T+30  If proxy also blocked: deploy new Worker on fresh subdomain
T+60  If Worker blocked: deploy VPS proxy in different datacenter
T+90  If VPS IP blocked: rotate to floating IP
T+2h  Post incident summary, update runbook with new learnings
```

---

## Post-Incident Checklist

- [ ] Document which ISP, region, and block type was observed
- [ ] Document which mitigation resolved it and how long it took
- [ ] Check if proxy URL was already known or if a new one was needed
- [ ] Add more monitoring coverage for affected regions
- [ ] Keep final IPs from this incident as spare capacity
- [ ] Review internal runbook   was anything missing?

---

## Pre-Event Preparation Checklist

Run this before any ISP block event:

- [ ] Cloudflare Worker proxy deployed at `proxy.yourproduct.com`
- [ ] Worker is already live and tested (not hypothetical)
- [ ] App already supports `SUPABASE_PROXY_URL` fallback env var
- [ ] One-click redeployment path established (Vercel env UI or CLI script)
- [ ] Health check monitoring from Mumbai and Bangalore regions
- [ ] Slack alerts configured for `proxy.yourproduct.com` downtime
- [ ] VPS with floating IP in Bangalore/Mumbai pre-provisioned ($5/mo)
- [ ] DNS TTL set to 60 seconds (not 3600) for fast propagation


