# Anti-DPI Techniques

## What is Deep Packet Inspection (DPI)?

Basic ISP blocks work on DNS (UDP port 53)   they respond to `supabase.co` lookups with a sinkhole IP.

DPI is more sophisticated: the ISP inspects the TLS SNI (Server Name Indication) field in the TLS `ClientHello` packet to see which hostname you're connecting to   even when using custom DNS. DPI-based blocks are used by:

- State-level firewalls (GFW in China, Iran, Russia)
- Enterprise firewalls with SSL inspection
- Some ISPs that go beyond DNS blocking

---

## Layer 1: DNS over HTTPS (DoH)

Basic defense   prevents DNS-level interception:

```typescript
// Browser: Force DoH via JavaScript
// Browsers already use DoH if configured. For Node.js:
import { Resolver } from 'dns/promises'

// Use Cloudflare DoH
const resolver = new Resolver()
resolver.setServers(['1.1.1.1', '1.0.0.1'])

// Or use doh-fetch library
import dohFetch from 'doh-fetch'

const result = await dohFetch('supabase.co', {
  doh: 'https://cloudflare-dns.com/dns-query',
  type: 'A',
})
```

Configure DoH system-wide in Chrome:

```
chrome://settings/security → Use secure DNS → With: Cloudflare (1.1.1.1)
```

---

## Layer 2: TLS 1.3 (Hides Certificate Details)

TLS 1.2 exposes the certificate in cleartext. TLS 1.3 encrypts more of the handshake.

**Nginx: Force TLS 1.3:**

```nginx
server {
    listen 443 ssl http2;
    server_name api.yourproduct.com;

    ssl_protocols TLSv1.3;                        # TLS 1.3 ONLY
    ssl_ciphers   TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256;
    ssl_prefer_server_ciphers on;

    ssl_certificate     /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;
}
```

**Node.js server with TLS 1.3:**

```javascript
const tls = require('tls')
const https = require('https')
const fs = require('fs')

const server = https.createServer({
  cert: fs.readFileSync('cert.pem'),
  key:  fs.readFileSync('key.pem'),
  minVersion: 'TLSv1.3',
  ciphers: 'TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256',
}, app)
```

---

## Layer 3: HTTP/3 (QUIC Protocol)

HTTP/3 uses QUIC over UDP instead of TCP. Most ISP DPI systems are designed for TCP   QUIC traffic often passes through unfiltered.

**Nginx: Enable HTTP/3 (requires nginx-quic build):**

```nginx
server {
    listen 443 quic reuseport;    # HTTP/3 over UDP
    listen 443 ssl http2;         # HTTP/2 fallback

    server_name api.yourproduct.com;

    ssl_protocols TLSv1.3;
    ssl_certificate     /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;

    # Advertise HTTP/3 support
    add_header Alt-Svc 'h3=":443"; ma=86400' always;
}
```

**Cloudflare: HTTP/3 is enabled by default** for all zones. No configuration needed.

**Verify HTTP/3 connectivity:**

```bash
curl --http3 -I https://api.yourproduct.com
# HTTP/3 200
```

---

## Layer 4: Encrypted Client Hello (ECH)

ECH is the strongest defense against SNI-based DPI. It encrypts the entire TLS `ClientHello`, including the SNI field. DPI cannot see which hostname you're connecting to.

**ECH is controlled by DNS (HTTPS records), not your server.**

```bash
# Check if your domain has ECH enabled via Cloudflare
dig +short HTTPS api.yourproduct.com
# Should show: 1 . ech=... (base64 encrypted key)
```

**Cloudflare automatically enables ECH** for all proxied domains. Just set your domain to "Proxied" (orange cloud) in Cloudflare DNS.

**Firefox: Enable ECH in browser:**

```
about:config → network.dns.echconfig.enabled = true
              → network.dns.use_https_rr_as_altsvc = true
```

---

## Layer 5: Domain Fronting

Domain fronting uses a CDN where the outer TLS SNI (what DPI sees) is a popular, unblockable domain, while the HTTP Host header (what the CDN uses to route) is your actual domain.

> **Note:** Most major CDNs (AWS, Cloudflare) now actively prevent domain fronting. This technique is primarily used in censorship-heavy regions.

---

## Layer 6: Obfs4 / shadowsocks (Transport Obfuscation)

For extreme DPI environments (e.g., China GFW), traffic obfuscation makes HTTPS traffic look like random noise:

**shadowsocks proxy server:**

```bash
# Install shadowsocks-libev on your VPS
apt install shadowsocks-libev

# /etc/shadowsocks-libev/config.json
{
    "server": "0.0.0.0",
    "server_port": 8388,
    "password": "your-strong-password",
    "timeout": 300,
    "method": "chacha20-ietf-poly1305",
    "fast_open": true,
    "plugin": "v2ray-plugin",
    "plugin_opts": "server;tls;host=yourproduct.com"
}

systemctl enable shadowsocks-libev
systemctl start shadowsocks-libev
```

**v2ray (vmess + WebSocket + TLS):**

```json
{
  "inbounds": [{
    "port": 443,
    "protocol": "vmess",
    "settings": {
      "clients": [{ "id": "YOUR-UUID" }]
    },
    "streamSettings": {
      "network": "ws",
      "wsSettings": { "path": "/ws" },
      "tlsSettings": {
        "serverName": "yourproduct.com",
        "certificates": [{
          "certificateFile": "/etc/nginx/certs/fullchain.pem",
          "keyFile": "/etc/nginx/certs/privkey.pem"
        }]
      },
      "security": "tls"
    }
  }]
}
```

---

## Detection Ladder

Use the minimum technique needed for your threat model:

| ISP Block Type | Required Defense |
|:---------------|:----------------|
| DNS poisoning | Custom DNS (1.1.1.1) or DoH |
| SNI filtering / DPI | ECH + Cloudflare proxy |
| IP range block | CDN (change IP via Cloudflare) |
| ASN block | CDN chaining (multiple ASNs) |
| Protocol fingerprinting | HTTP/3 (QUIC, UDP not TCP) |
| Deep obfuscation (GFW) | shadowsocks + v2ray |

---

## Quick Decision Tree

```
Blocked? →
  Yes → DNS issue? (nslookup blocked) →
    Yes → Use DoH or 1.1.1.1
  No → Still blocked after custom DNS? →
    Yes → SNI filtering → Enable ECH via Cloudflare
  No → Still blocked? →
    Yes → IP blocked → Use CDN (change IP)
  No → Still blocked? →
    Yes → Extreme DPI → HTTP/3 + QUIC or shadowsocks
```

---

## Testing DPI

```bash
# Test 1: Is it DNS?
nslookup api.supabase.co 1.1.1.1
nslookup api.supabase.co 8.8.8.8
# Different results from ISP DNS vs public DNS = DNS block

# Test 2: Is it SNI?
curl -v --resolve api.supabase.co:443:$(dig +short api.supabase.co @1.1.1.1 | head -1) \
  https://api.supabase.co/
# If this works but normal curl fails = DNS block, SNI is fine

# Test 3: Is it IP?
curl -v -H "Host: api.supabase.co" \
  https://$(dig +short api.supabase.co @1.1.1.1 | head -1)/
# If this is blocked = IP-level block

# Test 4: HTTP/3
curl --http3 -v https://api.yourproduct.com/
# If this succeeds when HTTP/1.1 fails = TCP DPI, QUIC bypasses it
```


