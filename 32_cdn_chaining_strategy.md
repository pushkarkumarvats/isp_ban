# CDN Chaining Strategy

## Concept

CDN chaining layers multiple CDNs between the user and your origin. Each CDN adds a hop that:
1. Hides your origin IP/ASN from end users and ISPs
2. Provides an independent fallback if one CDN is blocked
3. Distributes traffic for better latency and cost efficiency

```
User → Cloudflare → BunnyCDN → AWS CloudFront → Origin (Supabase / VPS)
```

An ISP that blocks CloudFront must also block Cloudflare to break your app   highly unlikely.

---

## Why Chain CDNs?

| Problem | CDN Chain Solution |
|:--------|:-----------------|
| ISP blocks CloudFront | Cloudflare catches it |
| ISP blocks Cloudflare | Practically impossible at scale |
| Origin IP leaked | Three hops hide the origin |
| One CDN is slow | Route to next via failover |
| ASN blocking | Multiple ASNs in chain |

---

## Layer 1: Cloudflare (Front)

Cloudflare is always the outermost layer. It handles:
- DDoS mitigation
- TLS termination
- WAF (Web Application Firewall)
- Page Rules / Transform Rules

```
DNS: api.yourproduct.com → CNAME → api-cdn.yourproduct.com (Cloudflare)
```

**Cloudflare Worker for routing logic:**

```typescript
// route-to-next-cdn.worker.ts
interface Env {
  BUNNY_URL: string      // https://yourdomain.b-cdn.net
  CLOUDFRONT_URL: string // https://d1234567890.cloudfront.net
  ORIGIN_URL: string     // https://your-vps.example.com
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url)
    
    // Route static assets to BunnyCDN (cheaper bandwidth)
    if (url.pathname.startsWith('/assets/') || url.pathname.startsWith('/_next/static/')) {
      const bunnyUrl = env.BUNNY_URL + url.pathname + url.search
      return fetch(bunnyUrl, { headers: request.headers })
    }
    
    // Route API calls to CloudFront (with origin shield)
    if (url.pathname.startsWith('/api/')) {
      const cfUrl = env.CLOUDFRONT_URL + url.pathname + url.search
      return fetch(new Request(cfUrl, request))
    }
    
    // All other traffic to origin
    const originUrl = env.ORIGIN_URL + url.pathname + url.search
    return fetch(new Request(originUrl, request))
  },
}
```

---

## Layer 2: BunnyCDN (Assets Layer)

BunnyCDN is a cost-efficient CDN with global PoPs. Use it for static assets.

**Pull Zone configuration:**

```
Pull Zone Name: yourproduct-assets
Origin URL: https://your-origin.yourproduct.com
Hostname: assets.yourproduct.com

BunnyCDN dashboard settings:
- Enable "Perma-Cache"
- Cache Everything: ON
- TTL: 31536000 (1 year for hashed assets)
```

**Nginx origin for BunnyCDN fallback:**

```nginx
# /etc/nginx/conf.d/assets.conf
server {
    listen 443 ssl http2;
    server_name assets-origin.yourproduct.com;

    # Only allow BunnyCDN IP ranges as cache fill
    # BunnyCDN IPs: https://bunnycdn.com/api/system/edgeserverlist
    allow 23.111.8.0/22;
    allow 185.152.64.0/22;
    deny all;

    location / {
        root /var/www/assets;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

---

## Layer 3: AWS CloudFront (API/Dynamic Layer)

Use CloudFront with "Origin Shield" for API caching and as an additional hop for dynamic content.

**CloudFront distribution configuration (Terraform):**

```hcl
# cloudfront.tf
resource "aws_cloudfront_distribution" "api" {
  enabled = true
  
  origin {
    domain_name = "api-origin.yourproduct.com"
    origin_id   = "api-origin"
    
    # Origin Shield in ap-south-1 (Mumbai) for India users
    origin_shield {
      enabled              = true
      origin_shield_region = "ap-south-1"
    }
    
    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }
  
  default_cache_behavior {
    allowed_methods        = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "api-origin"
    viewer_protocol_policy = "redirect-to-https"
    
    forwarded_values {
      query_string = true
      headers      = ["Authorization", "Accept", "Content-Type"]
      cookies { forward = "none" }
    }
    
    min_ttl     = 0
    default_ttl = 0    # Don't cache API responses by default
    max_ttl     = 0
  }
  
  # India-optimized price class
  price_class = "PriceClass_200"  # US, Europe, Asia
  
  restrictions {
    geo_restriction { restriction_type = "none" }
  }
  
  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.main.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }
}
```

---

## Hiding Origin ASN

Your origin server's ASN must never appear in public BGP tables. Three ways:

### 1. Cloudflare Tunnel (Best   No Exposed IP)

```bash
# Install cloudflared on your VPS
curl -L --output cloudflared.deb \
  https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
dpkg -i cloudflared.deb

# Create a tunnel   no inbound port needed
cloudflared tunnel login
cloudflared tunnel create my-origin
cloudflared tunnel route dns my-origin api-origin.yourproduct.com

# Start the tunnel (routes all Cloudflare traffic to localhost:8080)
cloudflared tunnel run --url http://localhost:8080 my-origin
```

Your VPS public IP is never used. All traffic flows through the Cloudflare Tunnel.

### 2. Firewall Allowlist (IP-Based)

```bash
# ufw rules   allow only Cloudflare and BunnyCDN IPs
ufw default deny incoming

# Cloudflare IPs (https://www.cloudflare.com/ips/)
for ip in 103.21.244.0/22 103.22.200.0/22 103.31.4.0/22 104.16.0.0/13 104.24.0.0/14 \
          108.162.192.0/18 131.0.72.0/22 141.101.64.0/18 162.158.0.0/15 172.64.0.0/13 \
          173.245.48.0/20 188.114.96.0/20 190.93.240.0/20 197.234.240.0/22 198.41.128.0/17; do
  ufw allow from $ip to any port 443
done

# Allow SSH from your IP
ufw allow from YOUR_IP proto tcp to any port 22

ufw enable
```

---

## CDN Failover Logic (Frontend)

```typescript
// lib/cdn-failover.ts
const CDN_URLS = [
  process.env.NEXT_PUBLIC_CDN_PRIMARY ?? 'https://api.yourproduct.com',
  process.env.NEXT_PUBLIC_CDN_FALLBACK ?? 'https://api-fallback.yourproduct.com',
]

export async function fetchWithFailover(path: string, options?: RequestInit): Promise<Response> {
  for (const base of CDN_URLS) {
    try {
      const controller = new AbortController()
      const timeout = setTimeout(() => controller.abort(), 8000)
      
      const response = await fetch(`${base}${path}`, {
        ...options,
        signal: controller.signal,
      })
      
      clearTimeout(timeout)
      
      if (response.ok) return response
    } catch {
      // Continue to next CDN
    }
  }
  
  throw new Error('All CDN endpoints unreachable')
}
```

---

## Chain Summary

| Layer | Provider | Purpose | Cost |
|:------|:---------|:--------|:-----|
| Front | Cloudflare Free | DDoS, TLS, WAF | Free |
| Assets | BunnyCDN | Static files, cheap bandwidth | ~$0.01/GB |
| API | AWS CloudFront | API caching, origin shield | ~$0.009/10k req |
| Origin | Hetzner / DigitalOcean | App server | $5-20/mo |

Total cost for 1M monthly requests + 100GB bandwidth: ~$8-15/month.


