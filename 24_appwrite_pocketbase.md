# Appwrite and PocketBase

## Overview

Both Appwrite and PocketBase are self-hosted, open-source backends-as-a-service. Because you host them yourself, you have full control over the IP, domain, and CDN setup. This makes ISP bypass significantly easier than with Supabase or Firebase   you simply put your own Nginx + Cloudflare in front.

---

## Appwrite: Self-Hosted Setup

Appwrite runs as a Docker stack. Default port is 80/443.

### Deploy Appwrite Behind Cloudflare

```yaml
# docker-compose.yml (Appwrite)
version: '3'

services:
  appwrite:
    image: appwrite/appwrite:latest
    environment:
      - _APP_ENV=production
      - _APP_DOMAIN=appwrite.yourproduct.com
      - _APP_DOMAIN_TARGET=appwrite.yourproduct.com
      - _APP_OPENSSL_KEY_V1=your-key
      - _APP_DB_HOST=mariadb
      - _APP_DB_USER=appwrite
      - _APP_DB_PASS=your-db-password
      - _APP_DB_SCHEMA=appwrite
      - _APP_REDIS_HOST=redis
    ports:
      - "80:80"
      - "443:443"
    networks:
      - appwrite
```

Enable Cloudflare proxy for `appwrite.yourproduct.com`. Appwrite now routes through Cloudflare   ISP bypass automatic.

### Appwrite Client with Custom Endpoint

```typescript
// src/lib/appwrite.ts
import { Client, Account, Databases, Storage } from 'appwrite'

const client = new Client()
  .setEndpoint(
    import.meta.env.VITE_APPWRITE_PROXY_URL 
      || import.meta.env.VITE_APPWRITE_ENDPOINT
  )
  .setProject(import.meta.env.VITE_APPWRITE_PROJECT_ID)

export const account = new Account(client)
export const databases = new Databases(client)
export const storage = new Storage(client)
```

```bash
# .env.production
VITE_APPWRITE_ENDPOINT=https://appwrite.yourproduct.com/v1
VITE_APPWRITE_PROJECT_ID=your-project-id
```

Since Appwrite is self-hosted, `appwrite.yourproduct.com` is already your domain. No separate proxy needed if Cloudflare is already in front.

---

## Appwrite: Restrict Direct Port Access

Never expose Appwrite's Docker ports to the public internet. Use Nginx as the entry point.

```nginx
# /etc/nginx/sites-available/appwrite
server {
    listen 443 ssl http2;
    server_name appwrite.yourproduct.com;

    ssl_certificate /etc/letsencrypt/live/appwrite.yourproduct.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/appwrite.yourproduct.com/privkey.pem;

    # Only allow Cloudflare IPs
    include /etc/nginx/cloudflare-ips.conf;
    deny all;

    location / {
        proxy_pass http://127.0.0.1:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $http_cf_connecting_ip;
        proxy_set_header X-Forwarded-Proto https;
        
        # WebSocket for Appwrite Realtime
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

```bash
# /etc/nginx/cloudflare-ips.conf (keep updated)
allow 103.21.244.0/22;
allow 103.22.200.0/22;
allow 104.16.0.0/13;
allow 104.24.0.0/14;
allow 108.162.192.0/18;
allow 131.0.72.0/22;
allow 141.101.64.0/18;
allow 162.158.0.0/15;
allow 172.64.0.0/13;
allow 173.245.48.0/20;
allow 188.114.96.0/20;
allow 190.93.240.0/20;
allow 197.234.240.0/22;
allow 198.41.128.0/17;
allow 2400:cb00::/32;
```

---

## Appwrite IP Rotation Automation

```bash
#!/bin/bash
# rotate-appwrite-ip.sh
# When current server IP is blocked, provision new server

NEW_SERVER_IP=$(terraform output -raw new_server_ip)
CLOUDFLARE_ZONE_ID=$CF_ZONE_ID
RECORD_ID=$CF_RECORD_ID

# Update DNS
curl -X PUT "https://api.cloudflare.com/client/v4/zones/$CLOUDFLARE_ZONE_ID/dns_records/$RECORD_ID" \
  -H "Authorization: Bearer $CF_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"type\":\"A\",\"name\":\"appwrite.yourproduct.com\",\"content\":\"$NEW_SERVER_IP\",\"ttl\":60}"

echo "IP rotated to $NEW_SERVER_IP"
```

---

## PocketBase: Self-Hosted Setup

PocketBase is a single Go binary   no Docker required.

```bash
# Download and run PocketBase
wget https://github.com/pocketbase/pocketbase/releases/latest/download/pocketbase_linux_amd64.zip
unzip pocketbase_linux_amd64.zip
./pocketbase serve --http="127.0.0.1:8090"
```

### Systemd Service

```ini
# /etc/systemd/system/pocketbase.service
[Unit]
Description=PocketBase
After=network.target

[Service]
Type=simple
User=pocketbase
WorkingDirectory=/opt/pocketbase
ExecStart=/opt/pocketbase/pocketbase serve --http=127.0.0.1:8090
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
systemctl enable pocketbase
systemctl start pocketbase
```

### Nginx in Front of PocketBase

```nginx
server {
    listen 443 ssl http2;
    server_name pb.yourproduct.com;

    ssl_certificate /etc/letsencrypt/live/pb.yourproduct.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/pb.yourproduct.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8090;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto https;

        # PocketBase Realtime (SSE)
        proxy_read_timeout 86400;
        proxy_buffering off;
        proxy_cache off;
    }
}
```

### PocketBase JS Client

```typescript
// src/lib/pocketbase.ts
import PocketBase from 'pocketbase'

const pb = new PocketBase(
  import.meta.env.VITE_POCKETBASE_URL || 'https://pb.yourproduct.com'
)

export default pb
```

Since `pb.yourproduct.com` is your own domain, routing through Cloudflare is instant. No additional proxy needed.

---

## PocketBase Realtime (SSE) Through Cloudflare

PocketBase uses Server-Sent Events (SSE) for realtime, not WebSockets. SSE works through Cloudflare without any special Worker configuration.

```typescript
// Subscribe to realtime changes
const unsubscribe = await pb.collection('posts').subscribe('*', (e) => {
  console.log(e.action, e.record)
})

// Unsubscribe when done
unsubscribe()
```

SSE passes through standard HTTP/2 connections   Cloudflare handles it natively.

---

## Self-Hosted vs Managed   Decision Matrix

| Feature | Appwrite Self-Hosted | PocketBase Self-Hosted | Supabase (Managed) |
|:--------|:--------------------|:----------------------|:-------------------|
| ISP Bypass | Full control | Full control | Requires proxy |
| Setup time | 1-2 hours | 15 minutes | None |
| Maintenance | Medium | Low | None |
| Scale | Container scaling | Single binary (limited) | Auto-scale |
| Cost | VPS ($5-20/mo) | VPS ($5/mo) | Free tier + paid |
| Realtime | WebSocket | SSE | WebSocket (needs proxy) |


