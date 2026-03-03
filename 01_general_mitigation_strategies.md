# General Mitigation Strategies

## Core Architecture Principle

Every production app exposed to ISP blocking must follow this topology:

```
User → CDN Edge → WAF → Load Balancer → App Server → Private DB
```

No IP that handles actual business logic or data should ever be directly reachable from the public internet. The CDN edge is the only public surface area.

---

## The Golden Rules

| Rule | Reason |
|:-----|:--------|
| Never expose origin server IP | Direct IP can be blocked independently of the domain |
| Always terminate TLS at CDN | Hides SNI from ISP; CDN absorbs DPI |
| Separate frontend & backend deployments | Allows independent failover |
| Keep database in private subnet | DB should never have a public IP |
| Use failover DNS (two providers) | If one DNS provider's IPs are blocked, the other serves |
| Maintain secondary provider standby | Hot or warm standby at a different cloud/CDN |
| Automate redeployment | Manual recovery under pressure is error-prone |

---

## Strategy 1: CDN Proxying (Most Effective)

Route all traffic through a CDN that sits between your users and your origin. The CDN terminates the TLS connection using your domain's certificate, then makes a fresh request to the origin.

**What the ISP sees**: A connection to `api.yourproduct.com` resolving to Cloudflare's IP `104.21.x.x`  
**What Supabase/your backend sees**: A request from the CDN's IP

### Implementation Checklist
- [ ] Register a domain you control
- [ ] Add domain to Cloudflare (free plan works)
- [ ] Set DNS record: `api.yourproduct.com CNAME your-worker.workers.dev`
- [ ] Enable Cloudflare proxy (orange cloud)   hides origin IP
- [ ] Create Worker or Page Rule to forward to real origin
- [ ] Set `SSL/TLS` to `Full (Strict)` in Cloudflare dashboard

---

## Strategy 2: Reverse Proxy on Your Own Server

If you have a VPS (DigitalOcean, Linode, Vultr, Hetzner), it can proxy requests to blocked services.

**Requirement**: Your VPS IP must not be on the ISP's blocklist. IPs from Hetzner Germany or DigitalOcean Frankfurt are typically clean.

```nginx
# /etc/nginx/sites-available/proxy
server {
    listen 443 ssl http2;
    server_name api.yourproduct.com;

    ssl_certificate /etc/letsencrypt/live/api.yourproduct.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.yourproduct.com/privkey.pem;

    location / {
        proxy_pass https://blocked-origin.example.com;
        proxy_set_header Host blocked-origin.example.com;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_ssl_server_name on;

        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 86400;
    }
}
```

---

## Strategy 3: Serverless Edge Functions as Proxy

Platforms like Cloudflare Workers, Vercel Edge Functions, and Deno Deploy run at the edge (close to users globally) and can act as transparent proxies.

**Advantages**:
- No server to manage
- Free tier sufficient for most use cases
- Distributed globally   low latency
- Easy to deploy and update

**Cloudflare Worker Proxy (Universal)**:
```javascript
export default {
  async fetch(request, env) {
    const ORIGIN = env.ORIGIN_URL; // e.g. https://your-project.supabase.co

    const url = new URL(request.url);
    const originUrl = new URL(ORIGIN);
    url.hostname = originUrl.hostname;
    url.protocol = originUrl.protocol;
    url.port = originUrl.port;

    const modifiedRequest = new Request(url.toString(), {
      method: request.method,
      headers: request.headers,
      body: request.body,
      redirect: "manual",
    });
    modifiedRequest.headers.set("host", originUrl.hostname);

    // Handle WebSocket upgrades
    if (request.headers.get("Upgrade") === "websocket") {
      return fetch(url.toString(), {
        method: request.method,
        headers: modifiedRequest.headers,
        body: request.body,
      });
    }

    return fetch(modifiedRequest);
  },
};
```

---

## Strategy 4: Multi-CDN Failover

No single CDN is immune to blocking. A multi-CDN strategy uses DNS failover (via a DNS provider like NS1 or Amazon Route 53) to switch between CDN providers automatically.

```
Primary:   api.yourproduct.com → Cloudflare Worker
Fallback:  api.yourproduct.com → AWS CloudFront (if Cloudflare blocked)
Emergency: api.yourproduct.com → Vercel Edge (last resort)
```

Use Route 53 Health Checks + DNS Failover routing policies to automate this.

---

## Strategy 5: Separation of Frontend and Backend

Never bundle frontend and backend on the same IP/domain.

| Layer | Deploy To | Why |
|:------|:----------|:----|
| Frontend (HTML/CSS/JS) | Vercel / Netlify / Cloudflare Pages | CDN-native, globally distributed |
| API / Backend | Railway / Render / Fly.io + Nginx | Separate IP from frontend |
| Database | Private subnet (Supabase / RDS / PlanetScale) | Never public-facing |
| File Storage | R2 / S3 + CDN | Separate domain for assets |

---

## Strategy 6: Keep a Hot Standby

A hot standby is a second deployment at a different provider, continuously receiving traffic (or kept warm with health checks). When the primary is blocked, DNS TTL (set to 60 seconds) ensures failover within 1 minute.

```
DNS with 60s TTL:
  api.yourproduct.com A → 203.0.113.10  (Cloudflare Worker, primary)
  Failover record      → 198.51.100.20  (AWS CloudFront, standby)
```

---

## Implementation Priority Matrix

For most teams:

1. **Day 1**: Cloudflare Worker proxy for blocked third-party services   free, takes 30 minutes
2. **Week 1**: Custom domain + Cloudflare proxy for your own backend
3. **Month 1**: Multi-CDN DNS failover
4. **Quarter 1**: Multi-region hot standby

---

## What NOT to Do

- Do not hardcode `*.supabase.co`, `*.firebase.com`, or any blocked hostname in client-side code
- Do not rely on `localhost` proxy for production (obvious)
- Do not use HTTP (non-TLS) for any proxied traffic   ISP can then read it
- Do not set DNS TTL to 86400 (24 hours)   failover will be slow
- Do not expose your VPS IP in error messages or headers


