# Cloudflare Workers and Alternatives

## Why Cloudflare Workers Are the Default Choice

Cloudflare operates ASN 13335   one of the most widely trusted and least-blocked networks globally. It serves traffic for millions of legitimate websites. An ISP cannot selectively block Cloudflare's IP ranges without also breaking banking apps, government portals, and news sites. This makes Cloudflare the most resilient proxy layer for ISP bypass scenarios.

**Free plan limits (2026)**:
- 100,000 requests/day
- 10ms CPU per request
- Deployed globally to 300+ edge locations
- Custom domain support

---

## Option 1: Cloudflare Workers (Recommended)

### Setup Steps

1. Go to [workers.cloudflare.com](https://workers.cloudflare.com)
2. Create a new Worker
3. Paste the script below
4. Set environment variable `ORIGIN` to `https://your-project.supabase.co`
5. Deploy
6. (Optional) Add a custom route: `api.yourproduct.com/*`

### Universal Proxy Worker

```javascript
export default {
  async fetch(request, env) {
    const ORIGIN = env.ORIGIN || "https://your-project.supabase.co";

    const originUrl = new URL(ORIGIN);
    const requestUrl = new URL(request.url);

    // Rewrite hostname to origin
    requestUrl.hostname = originUrl.hostname;
    requestUrl.protocol = originUrl.protocol;
    requestUrl.port = originUrl.port || "";

    // Clone headers and set correct host
    const headers = new Headers(request.headers);
    headers.set("host", originUrl.hostname);

    const newRequest = new Request(requestUrl.toString(), {
      method: request.method,
      headers: headers,
      body: ["GET", "HEAD"].includes(request.method) ? null : request.body,
      redirect: "manual",
    });

    // WebSocket passthrough for Realtime
    if (request.headers.get("Upgrade") === "websocket") {
      return fetch(requestUrl.toString(), {
        method: request.method,
        headers: headers,
        body: request.body,
      });
    }

    const response = await fetch(newRequest);

    // Remove security headers that would block cross-origin usage
    const responseHeaders = new Headers(response.headers);
    responseHeaders.delete("x-frame-options");

    return new Response(response.body, {
      status: response.status,
      statusText: response.statusText,
      headers: responseHeaders,
    });
  },
};
```

### wrangler.toml (for CLI deployment)

```toml
name = "supabase-proxy"
main = "src/index.js"
compatibility_date = "2024-01-01"

[vars]
ORIGIN = "https://your-project-id.supabase.co"

[[routes]]
pattern = "api.yourproduct.com/*"
zone_name = "yourproduct.com"
```

---

## Option 2: AWS CloudFront

Best for teams already on AWS. CloudFront distributes from 450+ edge locations.

### Setup via AWS Console

1. Go to **CloudFront → Create Distribution**
2. **Origin Domain**: `your-project.supabase.co`
3. **Origin Protocol Policy**: `HTTPS Only`
4. **Viewer Protocol Policy**: `Redirect HTTP to HTTPS`
5. **Cache Policy**: `CachingDisabled` (for API proxying)
6. **Origin Request Policy**: `AllViewer`   forwards all headers including `Authorization`
7. **Alternate Domain Names (CNAME)**: `api.yourproduct.com`
8. Add SSL certificate via ACM (free)

### CloudFront for WebSockets

WebSockets require a specific cache behavior:

1. Create a new **Cache Behavior** for path `/realtime/*`
2. Set **Cache Policy**: `CachingDisabled`
3. Set **Origin Request Policy**: `AllViewer`
4. Under **Response headers**: allow `Upgrade` and `Connection`

> Note: CloudFront WebSocket support requires HTTP/1.1 upgrades. Test thoroughly with Supabase Realtime before going to production.

---

## Option 3: Nginx on VPS

Control your own proxy. Best when you want full control over headers, rate limiting, and logging.

### Full Production Configuration

```nginx
# /etc/nginx/sites-available/supabase-proxy
upstream supabase_origin {
    server your-project.supabase.co:443;
    keepalive 32;
}

server {
    listen 80;
    server_name api.yourproduct.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name api.yourproduct.com;

    # SSL   use Let's Encrypt (certbot)
    ssl_certificate /etc/letsencrypt/live/api.yourproduct.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.yourproduct.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options nosniff always;

    # Proxy all requests
    location / {
        proxy_pass https://your-project.supabase.co;
        proxy_set_header Host your-project.supabase.co;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_ssl_server_name on;
        proxy_ssl_protocols TLSv1.2 TLSv1.3;
        proxy_buffering off;

        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;
    }

    # Health check endpoint
    location /health {
        return 200 "OK";
        add_header Content-Type text/plain;
    }
}
```

### Install and Certify

```bash
sudo apt install nginx certbot python3-certbot-nginx -y
sudo certbot --nginx -d api.yourproduct.com
sudo nginx -t && sudo systemctl reload nginx
```

---

## Option 4: Vercel Rewrites (HTTP Only   No Realtime)

For Next.js apps that do NOT use Supabase Realtime, Vercel rewrites are the simplest option.

```javascript
// next.config.js
module.exports = {
  async rewrites() {
    return [
      {
        source: "/supabase/:path*",
        destination: "https://your-project.supabase.co/:path*",
      },
    ];
  },
};
```

**Critical Limitation**: Vercel's edge network does NOT proxy WebSocket upgrades. Any `supabase.channel()` or `.subscribe()` call will fail silently if routed through Vercel rewrites.

---

## Option 5: Deno Deploy / Deno Fresh

For teams using Deno:

```typescript
// proxy.ts
Deno.serve(async (req) => {
  const ORIGIN = "https://your-project.supabase.co";
  const url = new URL(req.url);
  const targetUrl = new URL(ORIGIN + url.pathname + url.search);

  const headers = new Headers(req.headers);
  headers.set("host", new URL(ORIGIN).hostname);

  const response = await fetch(targetUrl.toString(), {
    method: req.method,
    headers,
    body: req.body,
  });

  return new Response(response.body, {
    status: response.status,
    headers: response.headers,
  });
});
```

---

## Comparison Table

| Method | Cost | WebSocket | Latency | Setup Time | Control |
|:-------|:-----|:----------|:--------|:-----------|:--------|
| Cloudflare Workers | Free (100k/day) | Yes | ~5ms | 15 min | Medium |
| AWS CloudFront | ~$0.01/GB | Yes (limited) | ~15ms | 45 min | High |
| Nginx on VPS | $5-10/mo VPS | Yes | ~20ms | 1 hour | Full |
| Vercel Rewrites | Free | **No** | ~10ms | 5 min | Low |
| Deno Deploy | Free (1M/mo) | Partial | ~10ms | 20 min | Medium |

---

## Frequently Asked Questions

**Q: Will Cloudflare's own IPs get blocked?**  
A: Possible but rare. Cloudflare's IPs serve millions of Indian banking, government, and news sites. Blocking them would cause massive collateral damage. If it does happen, follow the [CDN Chaining Strategy](./32_cdn_chaining_strategy.md).

**Q: Do I need a paid Cloudflare plan?**  
A: No. The free plan supports Workers with 100,000 requests/day. For most apps, this is sufficient. Workers Paid ($5/month) gives 10 million requests/day.

**Q: Can the ISP detect I'm using a proxy?**  
A: For TLS-encrypted traffic, they see the SNI (your domain name) and the destination IP (Cloudflare). They cannot see the content. If they block your specific domain, you rotate to a new one.


