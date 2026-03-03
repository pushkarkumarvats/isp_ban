# Enterprise Zero Trust Architecture

## Threat Model for Enterprise Apps

Zero Trust assumes:

1. **No trusted network**   every request is authenticated, regardless of origin
2. **No public origin**   origin server IP/ASN is never exposed to the public internet
3. **No direct DB access**   database is unreachable from the public internet
4. **Least-privilege tokens**   every service uses the minimum necessary credentials
5. **Mutual TLS (mTLS)**   services authenticate each other at the transport layer

This architecture inherently bypasses ISP bans because no traffic flows over paths that ISPs can intercept.

---

## Architecture Diagram

```
Internet Users
      │
      ▼
Cloudflare WAF + DDoS (all traffic enters here)
      │
      ▼
Cloudflare Tunnel → Load Balancer (private IP only)
      │
      ▼
API Gateway (identity-aware proxy)
      │
      ├─→ Auth Service (mTLS)
      ├─→ Application Server (mTLS)
      └─→ Internal DB (private subnet, no public interface)
```

---

## Cloudflare Access (Identity-Aware Proxy)

Cloudflare Access enforces identity verification before any request reaches your servers.

```yaml
# cloudflare-access-policy.yaml
# Configure via Cloudflare Zero Trust Dashboard → Access → Applications

Application:
  name: "Admin Dashboard"
  domain: "admin.yourproduct.com"
  session_duration: "8h"
  
  policies:
    - name: "Allow Corporate SSO"
      decision: allow
      include:
        - email_domain: yourcompany.com
        - idp_groups: ["engineering", "admin"]
      exclude:
        - email: ["contractor@yourcompany.com"]
```

```bash
# All requests to admin.yourproduct.com must pass through Cloudflare Access
# No request reaches your server without a valid Cloudflare Access JWT
```

**Validate Cloudflare Access JWT in your backend:**

```typescript
// middleware/cf-access.ts
import * as jose from 'jose'

const TEAM_DOMAIN = process.env.CF_ACCESS_TEAM_DOMAIN!  // https://yourteam.cloudflareaccess.com
const POLICY_AUD  = process.env.CF_ACCESS_POLICY_AUD!   // Application Audience Tag

async function validateCFAccessToken(token: string): Promise<jose.JWTPayload> {
  const certsUrl = `${TEAM_DOMAIN}/cdn-cgi/access/certs`
  const JWKS = jose.createRemoteJWKSet(new URL(certsUrl))
  
  const { payload } = await jose.jwtVerify(token, JWKS, {
    audience: POLICY_AUD,
    issuer: TEAM_DOMAIN,
  })
  
  return payload
}

export async function cfAccessMiddleware(req: any, res: any, next: any) {
  const token = req.headers['cf-access-jwt-assertion']
  
  if (!token) {
    return res.status(401).json({ error: 'Missing Cloudflare Access token' })
  }
  
  try {
    const payload = await validateCFAccessToken(token)
    req.user = { email: payload.email, sub: payload.sub }
    next()
  } catch {
    res.status(403).json({ error: 'Invalid or expired token' })
  }
}
```

---

## Mutual TLS (mTLS) Between Services

Services authenticate each other using client certificates:

```bash
# Generate CA and service certificates
# CA
openssl genrsa -out ca.key 4096
openssl req -new -x509 -key ca.key -sha256 -subj "/C=IN/CN=Internal CA" -days 3650 -out ca.crt

# Service certificate (e.g., for API Gateway → App Server)
openssl genrsa -out api-server.key 2048
openssl req -new -key api-server.key -out api-server.csr -subj "/C=IN/CN=api-server"
openssl x509 -req -in api-server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out api-server.crt -days 365 -sha256

# Database client certificate
openssl genrsa -out db-client.key 2048
openssl req -new -key db-client.key -out db-client.csr -subj "/C=IN/CN=db-client"
openssl x509 -req -in db-client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out db-client.crt -days 365 -sha256
```

**Nginx with mTLS (internal service):**

```nginx
# /etc/nginx/conf.d/internal-mtls.conf
server {
    listen 443 ssl http2;
    server_name internal-api.yourdomain.internal;

    ssl_certificate      /etc/nginx/certs/api-server.crt;
    ssl_certificate_key  /etc/nginx/certs/api-server.key;

    # Require client certificate
    ssl_client_certificate /etc/nginx/certs/ca.crt;
    ssl_verify_client      on;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header X-Client-Cert $ssl_client_s_dn;
        proxy_set_header X-Forwarded-For $remote_addr;
    }
}
```

**Node.js client with mTLS:**

```typescript
// lib/internal-client.ts
import https from 'https'
import fs from 'fs'
import axios from 'axios'

const internalClient = axios.create({
  baseURL: 'https://internal-api.yourdomain.internal',
  httpsAgent: new https.Agent({
    cert: fs.readFileSync('/etc/certs/db-client.crt'),
    key:  fs.readFileSync('/etc/certs/db-client.key'),
    ca:   fs.readFileSync('/etc/certs/ca.crt'),
  }),
})

export default internalClient
```

---

## Private Database (No Public Interface)

**PostgreSQL with private network only:**

```bash
# PostgreSQL configuration
# /etc/postgresql/16/main/postgresql.conf

listen_addresses = '10.0.1.10'   # Internal IP ONLY   not 0.0.0.0

# /etc/postgresql/16/main/pg_hba.conf
# TYPE  DATABASE   USER         ADDRESS          AUTH
host    all        app_user     10.0.1.0/24      scram-sha-256
hostssl all        app_user     10.0.1.0/24      cert clientcert=verify-full
# No entries for public internet addresses
```

**Supabase: Direct DB via Internal Network (Self-Hosted)**

```yaml
# docker-compose.yml for self-hosted Supabase
services:
  db:
    image: supabase/postgres:15.1.0.147
    networks:
      - internal
    # No ports: exported   DB is not accessible from host
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}

  api:
    image: supabase/gotrue:v2.138.0
    networks:
      - internal
      - external  # Only this service faces the internet
    depends_on:
      - db

networks:
  internal:
    internal: true   # No internet access
  external:
    internal: false
```

---

## VPN for Internal Access

```bash
# WireGuard VPN for developer access to internal services
# /etc/wireguard/wg0.conf (server)

[Interface]
Address    = 10.8.0.1/24
ListenPort = 51820
PrivateKey = SERVER_PRIVATE_KEY

PostUp     = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown   = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey  = DEV_PUBLIC_KEY
AllowedIPs = 10.8.0.2/32  # Developer laptop
```

```bash
# /etc/wireguard/wg0.conf (developer laptop)
[Interface]
Address    = 10.8.0.2/24
PrivateKey = DEV_PRIVATE_KEY
DNS        = 10.8.0.1

[Peer]
PublicKey  = SERVER_PUBLIC_KEY
Endpoint   = vpn.yourproduct.com:51820
AllowedIPs = 10.0.0.0/8   # Route all internal traffic through VPN
```

---

## Service Mesh with SPIFFE / SPIRE

For large clusters, use SPIRE to automatically issue and rotate mTLS certificates:

```yaml
# spire-server-config.yaml
server:
  bind_address: "0.0.0.0"
  bind_port: "8081"
  trust_domain: "yourproduct.com"
  default_svid_ttl: "1h"
  
plugins:
  DataStore:
    - sql:
        plugin_data:
          database_type: sqlite3
          connection_string: "/opt/spire/data/datastore.sqlite3"
  
  NodeAttestor:
    - k8s_psat:
        plugin_data:
          allowed_pod_label_keys: ["app"]
          allowed_node_label_keys: ["kubernetes.io/hostname"]
```

---

## Security Checklist

- [ ] No server has a public IP with open ports (use Cloudflare Tunnel)
- [ ] All database ports are closed to public internet
- [ ] mTLS between all internal services
- [ ] Cloudflare Access on all admin interfaces
- [ ] JWT validation in every service (not just in the gateway)
- [ ] Secrets managed by Vault / AWS Secrets Manager (not env files)
- [ ] All service-to-service calls use short-lived tokens (< 1 hour)
- [ ] Log all access to audit trail (CloudTrail / Cloudflare Logpush)
- [ ] Rotate all TLS certificates every 90 days (Let's Encrypt auto-renew)
- [ ] Vulnerability scanning on all container images before deploy


