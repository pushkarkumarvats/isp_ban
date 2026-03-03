# DNS-Level Strategies

## Overview

DNS (Domain Name System) is Layer 1 of the ISP blocking stack. Before a browser even connects to a server, it performs a DNS lookup. If this lookup is poisoned, all subsequent mitigation becomes irrelevant on the client side. DNS-level strategies focus on ensuring your own domain resolves correctly and your service's reachability does not depend on a single DNS path.

---

## Understanding DNS Poisoning in Detail

Indian ISPs intercept DNS queries on UDP port 53. Since UDP is connectionless and the ISP controls the network, they can inject a fake response faster than the real DNS server   a classic "race condition" attack.

```
Normal DNS:
Client → UDP port 53 → 8.8.8.8 → Returns 203.0.113.10 (real Supabase IP)

Poisoned DNS:
Client → UDP port 53 → ISP intercepts → Returns 0.0.0.0 (sinkhole)
         ↑ ISP injects fake answer before Google can respond
```

---

## Strategy 1: DNS-over-HTTPS (DoH)

DoH encrypts DNS queries inside HTTPS traffic. The ISP sees only a TLS connection to `1.1.1.1` or `8.8.8.8`   not the actual DNS query.

**Problem**: This helps your developers. It does NOT help your users unless you push a mobile/desktop app that configures DoH, or you build a browser extension.

**Browser-level DoH** (Chrome):
- Settings → Privacy → Use secure DNS → With: Cloudflare (1.1.1.1)

This is a development trick, not a production solution.

---

## Strategy 2: Your Domain with Cloudflare DNS (Authoritative DNS)

When you host your app on your own domain (`api.yourproduct.com`) and point it via Cloudflare, your domain uses Cloudflare's authoritative name servers. The ISP cannot poison queries for your domain unless they wholesale block Cloudflare's NS servers   which would break thousands of other sites.

### Setup

1. Register a domain from Namecheap / GoDaddy / Porkbun
2. In the registrar, set nameservers to:
   ```
   ns1.cloudflare.com
   ns2.cloudflare.com
   ```
3. In Cloudflare DNS, add your A/CNAME records with the orange cloud proxy enabled
4. Cloudflare now handles DNS resolution for your domain   ISP cannot poison it

---

## Strategy 3: CNAME Flattening + Low TTL

Set your DNS TTL (Time To Live) to 60–120 seconds. This ensures that when you change where your domain points (failover), clients re-resolve within 1–2 minutes.

```
# Cloudflare DNS settings
api.yourproduct.com  CNAME  your-worker.workers.dev  TTL: 60
```

For apex domains (e.g., `yourproduct.com` without subdomain), use Cloudflare's CNAME Flattening or an ALIAS record.

---

## Strategy 4: Multi-Provider DNS Failover

Use two DNS providers simultaneously. If one provider's IP range is blocked, the other continues serving responses.

### Route 53 Health Check Failover

```json
// Route 53 DNS Failover Record
{
  "Name": "api.yourproduct.com",
  "Type": "A",
  "SetIdentifier": "Primary",
  "Failover": "PRIMARY",
  "HealthCheckId": "YOUR_HEALTH_CHECK_ID",
  "TTL": 60,
  "ResourceRecords": [{ "Value": "104.21.x.x" }]  // Cloudflare IP
}

{
  "Name": "api.yourproduct.com",
  "Type": "A",
  "SetIdentifier": "Secondary",
  "Failover": "SECONDARY",
  "TTL": 60,
  "ResourceRecords": [{ "Value": "52.x.x.x" }]  // AWS CloudFront IP
}
```

Health Check config:
- Protocol: HTTPS
- Path: `/health`
- Interval: 30 seconds
- Failure threshold: 2 consecutive failures → trigger failover

---

## Strategy 5: NS1 or Dyn DNS with Traffic Steering

Enterprise-grade DNS providers (NS1, Dyn, Constellix) offer:
- Real-time traffic steering based on geography, latency, or health
- Millisecond TTL changes
- Anycast DNS (multiple PoPs answering your DNS queries)

For Indian users specifically, you can route `api.yourproduct.com` to your Cloudflare Worker endpoint. For users elsewhere, route to your primary.

---

## Strategy 6: Split DNS (Internal vs External)

For apps with both corporate/VPN users and public users:

```
External DNS (public):
  api.yourproduct.com → Cloudflare Worker (bypasses ISP)

Internal DNS (corporate VPN):
  api.yourproduct.com → 10.0.0.5 (internal load balancer, direct)
```

This avoids unnecessary proxy overhead for internal traffic.

---

## Strategy 7: DNS Redundancy for Your Own Infrastructure

If you are self-hosting:

1. **Primary NS**: Cloudflare (free, Anycast, DDoS protected)
2. **Secondary NS**: Hurricane Electric (free secondary DNS)
3. **Monitoring**: UptimeRobot or Checkly pinging DNS resolution every 1 minute

```bash
# Test your domain from multiple DNS resolvers
dig api.yourproduct.com @1.1.1.1
dig api.yourproduct.com @8.8.8.8
dig api.yourproduct.com @9.9.9.9
dig api.yourproduct.com @208.67.222.222  # OpenDNS
```

---

## DNS Configuration Checklist

- [ ] Domain DNS hosted on Cloudflare (not registrar's default DNS)
- [ ] Proxy enabled (orange cloud) on all public-facing records
- [ ] TTL set to 60–120 seconds for quick failover
- [ ] Health check endpoint exists (`/health` returns HTTP 200)
- [ ] Route 53 or secondary DNS failover configured
- [ ] DNS propagation tested from multiple resolvers
- [ ] No hardcoded IP addresses anywhere in client code
- [ ] DNSSEC enabled on your domain (prevents poisoning of YOUR records)

---

## DNSSEC: Protecting Your Own Records

While DNSSEC does not help your users resolve Supabase's blocked domain, it protects your own domain from being poisoned by a third party. Enable DNSSEC in Cloudflare (one click) and add the DS record to your registrar.

```
# Cloudflare auto-generates DNSSEC records
# Copy the DS record from Cloudflare → paste into registrar's DNSSEC section
DS record: 2371 13 2 <hash>
```


