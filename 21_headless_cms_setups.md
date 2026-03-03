# Headless CMS Setups

## Overview

Headless CMS platforms expose a content API that your frontend (Next.js, React, Vue, etc.) calls at runtime. If that CMS API is on a blocked domain, your frontend breaks.

**Affected platforms**: Strapi (self-hosted or Strapi Cloud), Directus (self-hosted), Contentful, Sanity, Prismic, DatoCMS, Ghost.

---

## Category 1: Self-Hosted CMS (Strapi, Directus)

You control the IP and domain. Put Nginx + Cloudflare in front.

### Strapi Behind Cloudflare

```nginx
# /etc/nginx/sites-available/strapi
server {
    listen 443 ssl http2;
    server_name cms.yourproduct.com;

    ssl_certificate /etc/letsencrypt/live/cms.yourproduct.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/cms.yourproduct.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:1337;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

Strapi config:

```javascript
// config/server.js
module.exports = ({ env }) => ({
  host: env('HOST', '0.0.0.0'),
  port: env.int('PORT', 1337),
  url: env('PUBLIC_URL', 'https://cms.yourproduct.com'),
  app: {
    keys: env.array('APP_KEYS'),
  },
});
```

```bash
# .env
PUBLIC_URL=https://cms.yourproduct.com
```

### Directus Behind Cloudflare

```bash
# .env (Directus)
PUBLIC_URL=https://cms.yourproduct.com
HOST=0.0.0.0
PORT=8055
```

Same Nginx config   just change `proxy_pass` to `http://127.0.0.1:8055`.

---

## Category 2: SaaS CMS with Cloudflare Worker Proxy

For Contentful, Sanity, Prismic   you do not control their servers, so you proxy their API through a Worker.

### Contentful Proxy (Cloudflare Worker)

```javascript
// cloudflare-worker.js
export default {
  async fetch(request, env) {
    const CONTENTFUL_SPACE = env.CONTENTFUL_SPACE_ID;
    const CONTENTFUL_TOKEN = env.CONTENTFUL_ACCESS_TOKEN;
    
    const url = new URL(request.url);
    
    // Rewrite to Contentful CDN
    url.hostname = 'cdn.contentful.com';
    url.pathname = `/spaces/${CONTENTFUL_SPACE}${url.pathname}`;
    
    const newRequest = new Request(url.toString(), {
      method: 'GET',
      headers: {
        'Authorization': `Bearer ${CONTENTFUL_TOKEN}`,
        'Content-Type': 'application/json',
      },
    });

    return fetch(newRequest);
  },
};
```

### Next.js with Contentful Proxy

```typescript
// lib/contentful.ts
const CONTENTFUL_BASE = process.env.NEXT_PUBLIC_CONTENTFUL_PROXY_URL 
  || 'https://cdn.contentful.com';
const SPACE_ID = process.env.NEXT_PUBLIC_CONTENTFUL_SPACE_ID;
const ACCESS_TOKEN = process.env.NEXT_PUBLIC_CONTENTFUL_ACCESS_TOKEN;

export async function getEntries(contentType: string) {
  const url = `${CONTENTFUL_BASE}/spaces/${SPACE_ID}/entries?content_type=${contentType}`;
  
  const response = await fetch(url, {
    headers: {
      Authorization: `Bearer ${ACCESS_TOKEN}`,
    },
    next: { revalidate: 60 }  // ISR   cached at CDN
  });

  return response.json();
}
```

```bash
# .env.production
NEXT_PUBLIC_CONTENTFUL_PROXY_URL=https://your-worker.workers.dev
```

---

## Category 3: Sanity CMS Proxy

Sanity's API is at `your-project-id.api.sanity.io`. If this is blocked:

```typescript
// sanity/client.ts
import { createClient } from '@sanity/client'

export const sanity = createClient({
  projectId: process.env.NEXT_PUBLIC_SANITY_PROJECT_ID!,
  dataset: 'production',
  useCdn: false,
  apiVersion: '2024-01-01',
  // Proxy URL instead of default *.api.sanity.io
  apiHost: process.env.NEXT_PUBLIC_SANITY_PROXY_URL || undefined,
})
```

Cloudflare Worker for Sanity:

```javascript
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    url.hostname = `${env.SANITY_PROJECT_ID}.api.sanity.io`;
    
    const newReq = new Request(url.toString(), request);
    newReq.headers.set('host', url.hostname);
    return fetch(newReq);
  }
}
```

---

## Category 4: Ghost CMS (Self-Hosted)

Ghost Headless mode exposes a Content API. Self-hosted Ghost + Cloudflare:

```bash
# Install Ghost behind a domain
ghost install --url https://cms.yourproduct.com

# Ghost config
ghost config url https://cms.yourproduct.com
```

```nginx
# Ghost Nginx config (similar to Strapi)
location / {
    proxy_pass http://127.0.0.1:2368;
    proxy_set_header Host $host;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}
```

---

## Category 5: ISR / SSG   Eliminate Runtime CMS Calls

The best ISP bypass for content: pre-fetch all CMS content at build time (SSG/ISR). The user's browser never makes a runtime API call to the CMS.

### Next.js ISR (Incremental Static Regeneration)

```typescript
// app/blog/[slug]/page.tsx
export async function generateStaticParams() {
  // Runs at build time on your server   not affected by ISP
  const posts = await sanity.fetch('*[_type == "post"]{slug}');
  return posts.map(p => ({ slug: p.slug.current }));
}

export default async function BlogPost({ params }) {
  // Runs on server (ISR revalidation)   not in browser
  const post = await sanity.fetch(`*[slug.current == $slug][0]`, { slug: params.slug });
  return <article>{post.body}</article>;
}

export const revalidate = 3600; // Re-fetch from CMS every hour, server-side
```

With ISR, content is cached on Vercel's CDN. Users receive pre-rendered HTML. Zero runtime CMS API calls from the browser.

---

## CMS API Access Control

Keep CMS APIs private   only allow access from:
- Your own server IPs (Vercel, Railway, VPS)
- Cloudflare Worker IPs (for the proxy)
- NOT the public internet directly

```javascript
// Strapi middleware: IP allowlist
module.exports = (config, { strapi }) => {
  return async (ctx, next) => {
    const allowedIps = process.env.ALLOWED_IPS?.split(',') || [];
    const clientIp = ctx.request.ip;
    
    // In production, only allow Cloudflare IPs or your server IPs
    if (process.env.NODE_ENV === 'production' && !allowedIps.includes(clientIp)) {
      ctx.status = 403;
      return;
    }
    
    await next();
  };
};
```


