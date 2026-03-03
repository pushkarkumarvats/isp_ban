# Understanding ISP Bans

## What Is Happening

Since February 2026, major Indian ISPs (Jio, Airtel, Vi/Vodafone-Idea, BSNL) have been performing active DNS poisoning on `*.supabase.co` subdomains. When a user on these ISPs attempts to resolve `your-project-id.supabase.co`, the ISP's DNS resolver returns a sinkhole IP address (e.g., `0.0.0.0` or an internal ISP server) instead of the real Supabase IP. The result is silent failure   connection timeouts, auth breakage, and data loss   with no clear error message shown to the user.

---

## The 9 Types of ISP-Level Blocks

### 1. IP Address Blocking
The ISP maintains a blocklist of specific IP addresses. Any packet destined for those IPs is dropped at the router level.

- **Detection**: `ping` returns no response. `traceroute` stops at the ISP's own gateway.
- **Bypass**: Move your origin to a different IP. Use a CDN (Cloudflare, Fastly, Akamai) that allocates shared IPs not on the blocklist.

### 2. Domain Name Blocking (BGP/Route Blocking)
The ISP announces a BGP route for the target's IP prefix, effectively hijacking traffic at the network routing level.

- **Detection**: `traceroute` never reaches the destination. May see redirects to ISP error page.
- **Bypass**: Using a proxy or CDN that is NOT in the same BGP prefix.

### 3. DNS Poisoning (What Is Happening Now)
The ISP intercepts DNS queries (UDP port 53) and returns a fake answer. Even if the real IP is not blocked, the client never gets to try it.

- **Detection**:
  ```bash
  # Compare results   if they differ, you are being poisoned
  nslookup your-project.supabase.co 8.8.8.8
  nslookup your-project.supabase.co 1.1.1.1
  nslookup your-project.supabase.co   # Your ISP's default
  ```
- **Bypass options**:
  - Use DNS-over-HTTPS (DoH)   encrypts DNS query so ISP cannot intercept
  - Use DNS-over-TLS (DoT)
  - Proxy behind your own domain (the recommended approach in this guide)

### 4. ASN (Autonomous System Number) Blocking
Instead of blocking individual IPs, the ISP blocks the entire ASN   a range of IP addresses owned by one organization (e.g., AWS us-east-1, Supabase's hosting provider).

- **Detection**: Multiple services on the same cloud provider are unreachable.
- **Bypass**: CDN provider that uses a different ASN (Cloudflare ASN 13335 is rarely blocked).

### 5. Port Blocking
The ISP blocks specific TCP/UDP ports. Common targets: port 80, 443, 8080, custom ports.

- **Detection**:
  ```bash
  telnet your-project.supabase.co 443
  # Connection refused or hangs
  ```
- **Bypass**: Use an alternative port if the service supports it. Tunnel over port 443 (HTTPS).

### 6. SNI Filtering (TLS Interception)
During the TLS handshake, the `Client Hello` message contains the hostname (SNI   Server Name Indication) in plaintext. The ISP reads this and drops connections to blocked hostnames, even though the traffic is supposedly encrypted.

- **Detection**: Connection fails after TCP handshake but before data transfer.
- **Bypass**:
  - Encrypted Client Hello (ECH)   hides the SNI
  - Proxy/CDN that uses your own domain as the SNI

### 7. Deep Packet Inspection (DPI)
The ISP inspects the content of unencrypted packets or uses protocol fingerprinting to identify and block specific traffic patterns (e.g., block all WebRTC, all Tor-like patterns).

- **Detection**: HTTPS traffic often passes; specific protocols are throttled.
- **Bypass**: Use TLS 1.3, obfuscated proxies, HTTP/3 (QUIC over UDP).

### 8. Protocol Throttling
The ISP does not outright block but severely limits bandwidth for specific protocols or destinations (e.g., all video streaming, all VPN protocols).

- **Detection**: Connection succeeds but is unusably slow. `speedtest` shows normal, but specific site is slow.
- **Bypass**: CDN acceleration, HTTP/3, compression.

### 9. CDN IP Range Blocking
The ISP blocks the known IP ranges of major CDNs (e.g., Cloudflare's 104.16.0.0/12). This is aggressive and often backfires because millions of legitimate sites share these IPs.

- **Detection**: Cloudflare-hosted sites all fail.
- **Bypass**: Use a lesser-known CDN (BunnyCDN, Fastly edge, Azure CDN) or your own IP.

---

## How to Diagnose Which Block You Are Facing

```bash
# Step 1: DNS check (most common   DNS poisoning)
nslookup your-project.supabase.co 1.1.1.1
nslookup your-project.supabase.co 8.8.8.8

# Step 2: Direct IP check (rule out IP block)
# Find real IP first using a VPN
curl -I https://your-project.supabase.co

# Step 3: Traceroute (find where packets die)
tracert your-project.supabase.co   # Windows
traceroute your-project.supabase.co  # Linux/Mac

# Step 4: Port check
telnet your-project.supabase.co 443

# Step 5: SNI check
openssl s_client -connect your-project.supabase.co:443 -servername your-project.supabase.co
```

### Diagnosis Matrix

| Symptom | Likely Cause |
|:--------|:-------------|
| DNS returns wrong IP | DNS Poisoning |
| Traceroute stops at ISP hop | IP or BGP Block |
| TCP connects but TLS fails | SNI Filtering |
| Very slow, not failing | Throttling |
| Only specific ports blocked | Port Blocking |
| All Cloudflare sites fail | CDN IP Range Block |

---

## Why Individual Users Cannot Fix This Permanently

- Changing DNS to `1.1.1.1` helps IF the issue is pure DNS poisoning and the ISP allows encrypted DNS
- A VPN bypasses everything   but you cannot ship a VPN requirement to 10,000 users
- Browser extensions like Psiphon or Lantern are not viable for production apps
- The only permanent, scalable, user-transparent fix is a **server-side proxy behind your own domain**

---

## The Correct Mental Model

```
BEFORE (Broken):
User (Jio) → DNS query → Jio DNS poisons → Wrong IP → Connection fails

AFTER (Fixed):
User (Jio) → DNS query → Resolves YOUR domain → CDN/Proxy (not blocked) → Supabase
```

Your domain becomes the entry point. The ISP may block `supabase.co` but they cannot block `api.yourproduct.com` without also breaking your entire product   a much more costly move for them politically and legally.


