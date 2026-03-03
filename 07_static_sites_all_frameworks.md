# Static Sites   All Frameworks

## Overview

Static sites are the easiest tier to protect from ISP blocking. Since all assets (HTML, CSS, JS, images) are served from a CDN by default, you simply need to ensure your CDN is not blocked and your API calls go through a proxy.

**Affected static site generators**: Next.js static export, React (CRA), Vue CLI, Svelte/SvelteKit static, Astro, Nuxt static, Gatsby, Hugo, Jekyll, Eleventy, plain HTML/CSS.

---

## The Problem for Static Sites

Static files themselves are rarely blocked. The blocking happens when:
1. The JS on the static site makes direct API calls to `*.supabase.co`, `*.firebase.com`, or other blocked backends
2. The CDN serving the static files is itself blocked (rare)

---

## Solution 1: Multi-CDN for Static Asset Delivery

Deploy your static build to multiple CDN providers. DNS failover switches between them.

### Deploy to All Three Simultaneously

```bash
# Build once
npm run build

# Deploy to Vercel
vercel deploy --prod

# Deploy to Netlify
netlify deploy --prod --dir=dist

# Deploy to Cloudflare Pages
wrangler pages deploy dist --project-name your-site
```

### DNS Failover

```
yourproduct.com → Vercel Edge (primary)
Failover:       → Netlify CDN
Emergency:      → Cloudflare Pages
```

---

## Solution 2: Proxy API Calls Through Cloudflare Worker

For static sites that call a backend (Supabase, Firebase, etc.), create a Cloudflare Worker that proxies all API requests. Update the static site's API base URL to point to the worker.

### React (CRA / Vite)

```javascript
// src/lib/api.js
const API_BASE = import.meta.env.VITE_API_PROXY_URL 
  || import.meta.env.VITE_API_URL;

// Instead of: fetch("https://your-project.supabase.co/rest/v1/...")
// Use:        fetch(`${API_BASE}/rest/v1/...`)
export const supabase = createClient(API_BASE, ANON_KEY);
```

```bash
# .env.production
VITE_API_PROXY_URL=https://your-worker.workers.dev
```

### Vue 3 (Vite)

```javascript
// src/composables/useSupabase.js
import { createClient } from '@supabase/supabase-js'

const url = import.meta.env.VITE_SUPABASE_PROXY_URL 
  || import.meta.env.VITE_SUPABASE_URL

export const supabase = createClient(url, import.meta.env.VITE_SUPABASE_ANON_KEY)
```

### Svelte / SvelteKit (Static Mode)

```javascript
// src/lib/supabase.js
import { createClient } from '@supabase/supabase-js'

const url = import.meta.env.PUBLIC_SUPABASE_PROXY_URL 
  || import.meta.env.PUBLIC_SUPABASE_URL

export const supabase = createClient(url, import.meta.env.PUBLIC_SUPABASE_ANON_KEY)
```

### Astro

```javascript
// src/lib/supabase.ts
import { createClient } from "@supabase/supabase-js";

const supabaseUrl = import.meta.env.PUBLIC_SUPABASE_PROXY_URL 
  ?? import.meta.env.PUBLIC_SUPABASE_URL;

export const supabase = createClient(
  supabaseUrl,
  import.meta.env.PUBLIC_SUPABASE_ANON_KEY
);
```

---

## Solution 3: IPFS Mirror (Censorship-Resistant)

Static sites can be pinned to IPFS (InterPlanetary File System) as an emergency mirror. IPFS is a peer-to-peer protocol   there is no central server to block.

```bash
# Install IPFS CLI
npm install -g ipfs-deploy

# Deploy to IPFS via Pinata
ipfs-deploy dist/ -p pinata

# Access via any IPFS gateway
# https://ipfs.io/ipfs/QmYour...Hash
# https://your-project.on.fleek.co
```

**Limitation**: IPFS gateways (`ipfs.io`, `cloudflare-ipfs.com`) can themselves be blocked. For guaranteed access, users would need the IPFS desktop app. This is an extreme fallback, not a primary strategy.

---

## Solution 4: Object Storage + CDN

Self-host your static files on S3-compatible object storage with a CDN in front.

```bash
# Deploy to Cloudflare R2 (S3 compatible, free 10GB)
aws s3 sync dist/ s3://your-bucket/ \
  --endpoint-url https://your-account.r2.cloudflarestorage.com \
  --delete

# Set R2 custom domain: yourproduct.com → R2 bucket
```

This gives you full control over the CDN domain and origin IP.

---

## Solution 5: Domain Rotation Strategy

Prepare multiple domain names pointing to the same static site deployment. If one domain gets blocked (rare but possible), flip DNS or provide users with the alternate URL.

```
Primary:   yourproduct.com
Mirror A:  yourproduct.in
Mirror B:  yourproduct.io
Emergency: yourproduct-app.pages.dev (Cloudflare, hard to block)
```

Store the list of mirrors in a `manifest.json` served from a decentralized source (GitHub raw, IPFS). A service worker can check the manifest and switch domains automatically.

---

## Framework-Specific Build Outputs

| Framework | Build Command | Output Dir | Static Export Config |
|:----------|:-------------|:-----------|:--------------------|
| React (Vite) | `npm run build` | `dist/` | Default |
| React (CRA) | `npm run build` | `build/` | Default |
| Next.js | `next build` | `.next/` or `out/` | `output: 'export'` in next.config.js |
| Vue CLI | `npm run build` | `dist/` | Default |
| Vue (Vite) | `npm run build` | `dist/` | Default |
| Nuxt | `nuxt generate` | `.output/public/` | Default static |
| SvelteKit | `npm run build` | `build/` | `adapter-static` |
| Astro | `npm run build` | `dist/` | Default |
| Gatsby | `gatsby build` | `public/` | Default |

---

## Environment Variable Strategy

For static sites, environment variables are baked into the JS bundle at build time. You cannot change them without rebuilding. Plan for this:

```bash
# Development
VITE_API_URL=https://your-project.supabase.co

# Production (normal regions, no ISP block)
VITE_API_URL=https://your-project.supabase.co

# Production India fallback (ISP block active)
VITE_API_URL=https://your-worker.workers.dev

# Rebuild and redeploy with proxy URL takes < 5 minutes on Vercel/Netlify
```

**Better approach**: Always build production with the proxy URL. The proxy adds <5ms latency and eliminates the need to redeploy under pressure.


