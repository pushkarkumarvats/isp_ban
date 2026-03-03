# VPS and IP Rotation

## Why VPS IP Rotation Exists as a Strategy

When you proxy traffic through your own VPS, the VPS's public IP becomes the only point of failure. If the ISP blocks that IP (either targeted or as collateral from blocking a netblock), your proxy is down. IP rotation means you prepare or automate switching to a new IP before or immediately after a block.

---

## The IP Rotation Lifecycle

```
Phase 1: Normal Operation
  User → api.yourproduct.com (A record: 203.0.113.10) → VPS → Origin

Phase 2: IP Blocked by ISP
  User → api.yourproduct.com → Connection timeout
  Monitoring detects failure within 30 seconds

Phase 3: Rotation
  New VPS or new IP provisioned (manual or automated)
  DNS A record updated: 203.0.113.10 → 198.51.100.20
  TTL 60s ensures propagation in 1 minute

Phase 4: Restored
  User → api.yourproduct.com (A record: 198.51.100.20) → New VPS → Origin
```

---

## Strategy 1: Manual IP Rotation (Simple)

Maintain a list of VPS instances across providers. When one IP is blocked, point DNS to the next.

### Recommended VPS Providers (IP Diversity)

| Provider | Location Options | Starting Price | Notes |
|:---------|:----------------|:--------------|:-------|
| Hetzner | Germany, Finland, US | $4/mo | Clean IP ranges, rarely blocked |
| Contabo | Germany, US, Singapore | $6/mo | Large IP pools |
| Vultr | 25 locations worldwide | $6/mo | Easy snapshot/clone |
| DigitalOcean | Multiple regions | $6/mo | Good API for automation |
| OVH/SoYouStart | France, US, Canada | $5/mo | ASN separate from major clouds |

### Quick Swap Procedure

```bash
# Step 1: Create new VPS (e.g., Vultr API)
curl -X POST "https://api.vultr.com/v2/instances" \
  -H "Authorization: Bearer $VULTR_API_KEY" \
  -d '{"region":"sgp","plan":"vc2-1c-1gb","os_id":387}'

# Step 2: Deploy Nginx proxy on new VPS via cloud-init or Ansible
ansible-playbook deploy-proxy.yml -i new-vps-ip,

# Step 3: Update DNS A record (Cloudflare API)
curl -X PUT "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records/$RECORD_ID" \
  -H "Authorization: Bearer $CF_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"type":"A","name":"api.yourproduct.com","content":"NEW_IP","ttl":60}'
```

---

## Strategy 2: Automated IP Rotation with Monitoring

Set up continuous monitoring. On failure, automatically provision a new VPS and update DNS.

### Monitoring + Rotation Script

```python
# monitor_and_rotate.py
import requests
import time
import subprocess

PROXY_URL = "https://api.yourproduct.com/health"
CHECK_INTERVAL = 30  # seconds
FAILURE_THRESHOLD = 3

failures = 0

def check_health():
    try:
        response = requests.get(PROXY_URL, timeout=10)
        return response.status_code == 200
    except:
        return False

def rotate_ip():
    print("[ALERT] Proxy unreachable. Rotating IP...")
    # Provision new VPS via Terraform or cloud API
    subprocess.run(["terraform", "apply", "-auto-approve"], cwd="./infra")
    # Update DNS record
    subprocess.run(["python", "update_dns.py"])
    print("[INFO] IP rotation complete.")

while True:
    if not check_health():
        failures += 1
        print(f"[WARN] Health check failed ({failures}/{FAILURE_THRESHOLD})")
        if failures >= FAILURE_THRESHOLD:
            rotate_ip()
            failures = 0
    else:
        failures = 0
    time.sleep(CHECK_INTERVAL)
```

---

## Strategy 3: Terraform-Managed VPS Pool

Use Infrastructure as Code to pre-provision standby VPS instances. On failure, the standby is already warm and ready.

### Terraform Config (Hetzner Example)

```hcl
# main.tf
terraform {
  required_providers {
    hcloud = {
      source  = "hetznercloud/hcloud"
      version = "~> 1.45"
    }
  }
}

provider "hcloud" {
  token = var.hcloud_token
}

resource "hcloud_server" "proxy" {
  count       = 2  # Primary + standby
  name        = "proxy-${count.index}"
  server_type = "cx11"
  image       = "ubuntu-22.04"
  location    = count.index == 0 ? "fsn1" : "nbg1"

  user_data = file("cloud-init.yaml")
}

output "proxy_ips" {
  value = hcloud_server.proxy[*].ipv4_address
}
```

```yaml
# cloud-init.yaml
#cloud-config
packages:
  - nginx
  - certbot
  - python3-certbot-nginx

runcmd:
  - |
    cat > /etc/nginx/sites-available/proxy << 'EOF'
    server {
        listen 80;
        location /health { return 200 "OK"; }
        location / {
            proxy_pass https://your-project.supabase.co;
            proxy_set_header Host your-project.supabase.co;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
    }
    EOF
  - ln -s /etc/nginx/sites-available/proxy /etc/nginx/sites-enabled/
  - nginx -t && systemctl reload nginx
```

---

## Strategy 4: Floating IP / Elastic IP

Cloud providers offer "floating IPs" (known as Elastic IPs on AWS)   IP addresses that are not tied to a specific server instance. You can reassign a floating IP from one server to another in under 5 seconds via API, without changing your DNS.

### Hetzner Floating IP Example

```bash
# Assign floating IP to standby server instantly
curl -X POST "https://api.hetzner.cloud/v1/floating_ips/$FLOATING_IP_ID/actions/assign" \
  -H "Authorization: Bearer $HCLOUD_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"server": NEW_SERVER_ID}'
```

DNS remains pointing to the floating IP   no DNS propagation delay.

---

## Strategy 5: Multiple IPs on One Server (IP Aliasing)

Some VPS providers allow multiple public IPs on a single server. Configure Nginx to listen on all IPs. If one IP gets blocked, update DNS to the second IP   no new server provisioned.

```bash
# Check available IPs on server
ip addr show

# Nginx: listen on all IP aliases
server {
    listen 203.0.113.10:443 ssl;
    listen 198.51.100.20:443 ssl;
    server_name api.yourproduct.com;
    ...
}
```

---

## IP Selection Guidelines

**Choose IPs from these characteristics for clean, unblocked IPs:**
- European datacenters (Germany, Netherlands, Finland)
- Not shared with known spammers or content farms
- Not on Spamhaus or MX Toolbox blocklists
- Check: https://mxtoolbox.com/blacklists.aspx

**Avoid:**
- US datacenters (more likely to be on bulk blocklists)
- Cheapest VPS providers (often recycle flagged IPs)
- IPs previously used for cryptocurrency mining

---

## IP Rotation Playbook (Emergency)

```
T+0:00  Monitoring alert fires   proxy unreachable from 3 checks
T+0:30  Confirm block from multiple networks (use down.com or whatsmydns.net)
T+1:00  Switch DNS to standby IP (pre-provisioned) OR reassign floating IP
T+2:00  Verify new IP resolves from Indian ISP networks
T+5:00  Deploy fresh Nginx config to new VPS
T+10:00 Run smoke tests: auth, DB query, storage upload, realtime
T+15:00 Post status update to users
T+60:00 Investigate original IP block   report to abuse registry if targeted
```


