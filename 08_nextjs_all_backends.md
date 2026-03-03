# Next.js With All Backends

## The Next.js Advantage for ISP Bypass

Next.js's App Router includes API Routes and Server Actions   server-side entry points that run on Vercel's or your own server's infrastructure. These server-side functions are never blocked by the ISP because the DNS poisoning only affects the end-user's browser, not your server. Your Next.js server makes the Supabase/backend call from a datacenter IP, which is clean.

**Architecture**:
```
User (Jio) → yourproduct.com (Vercel/Cloudflare) → API Route (server-side) → Supabase/Backend
```

The ISP sees: a request to `yourproduct.com` (your domain, not blocked)  
Supabase/backend is called server-side   not from the user's poisoned DNS

---

## Next.js + Supabase (Fully Tested   See supabase_nextjs.md)

The complete guide for Next.js + Supabase with Cloudflare Worker proxy is in [../supabase_nextjs.md](../supabase_nextjs.md). Summary:

1. Create Cloudflare Worker proxy pointing to `your-project.supabase.co`
2. Set `NEXT_PUBLIC_SUPABASE_PROXY_URL` to the worker URL in Vercel environment
3. Initialize Supabase client using the proxy URL
4. WebSockets (Realtime) work through the Worker

---

## Next.js + Node.js / Express API

### Proxy via Next.js API Route

```typescript
// app/api/proxy/[...path]/route.ts
import { NextRequest, NextResponse } from 'next/server'

const API_ORIGIN = process.env.BACKEND_API_URL // e.g. https://api.yourcompany.com

export async function GET(
  request: NextRequest,
  { params }: { params: { path: string[] } }
) {
  const path = params.path.join('/')
  const search = request.nextUrl.search
  
  const response = await fetch(`${API_ORIGIN}/${path}${search}`, {
    headers: {
      'Authorization': request.headers.get('Authorization') || '',
      'Content-Type': 'application/json',
    },
  })

  const data = await response.json()
  return NextResponse.json(data, { status: response.status })
}

export async function POST(
  request: NextRequest,
  { params }: { params: { path: string[] } }
) {
  const path = params.path.join('/')
  const body = await request.text()
  
  const response = await fetch(`${API_ORIGIN}/${path}`, {
    method: 'POST',
    headers: {
      'Authorization': request.headers.get('Authorization') || '',
      'Content-Type': request.headers.get('Content-Type') || 'application/json',
    },
    body,
  })

  const data = await response.json()
  return NextResponse.json(data, { status: response.status })
}
```

Client usage:
```typescript
// Instead of: fetch('https://api.yourcompany.com/users')
// Use:        fetch('/api/proxy/users')
const users = await fetch('/api/proxy/users').then(r => r.json())
```

---

## Next.js + Firebase

Firebase has its own set of blocked endpoints in India. Same Cloudflare Worker pattern applies.

```typescript
// lib/firebase.ts
import { initializeApp } from 'firebase/app'
import { getFirestore, connectFirestoreEmulator } from 'firebase/firestore'

const firebaseConfig = {
  // Use proxy URL for apiEndpoint if Firebase is blocked
  apiKey: process.env.NEXT_PUBLIC_FIREBASE_API_KEY,
  authDomain: process.env.NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN,
  projectId: process.env.NEXT_PUBLIC_FIREBASE_PROJECT_ID,
}

const app = initializeApp(firebaseConfig)
export const db = getFirestore(app)
```

For Firebase Auth specifically, use the `authDomain` environment variable to point to your proxy domain   Firebase supports custom auth domains.

---

## Next.js + Django (REST API)

### next.config.js with Rewrites

```javascript
// next.config.js
module.exports = {
  async rewrites() {
    return [
      {
        source: '/api/django/:path*',
        destination: `${process.env.DJANGO_API_URL}/:path*`,
      },
    ]
  },
  // Django API URL is set server-side   never in NEXT_PUBLIC
}
```

```bash
# .env.local
DJANGO_API_URL=https://your-django-server.com  # Server-side only
```

### With Django Channels (WebSocket)

Vercel rewrites do NOT proxy WebSockets. For Django Channels, use a Cloudflare Worker as the WebSocket proxy, just like Supabase Realtime.

---

## Next.js + Laravel

```javascript
// next.config.js
module.exports = {
  async rewrites() {
    return process.env.NODE_ENV === 'production'
      ? [
          {
            source: '/api/:path*',
            destination: `${process.env.LARAVEL_URL}/api/:path*`,
          },
        ]
      : []
  },
}
```

Set `LARAVEL_URL` to your Laravel server URL. Never expose it as `NEXT_PUBLIC_`   it is used server-side only.

---

## Next.js + Spring Boot

Spring Boot apps are typically deployed on AWS or GCP VMs. Same proxy pattern:

```typescript
// app/api/java/[...slug]/route.ts
export async function GET(request: NextRequest, { params }) {
  const url = `${process.env.SPRING_BASE_URL}/${params.slug.join('/')}`
  const res = await fetch(url, {
    headers: { Authorization: request.headers.get('Authorization') || '' }
  })
  return new Response(res.body, { status: res.status, headers: res.headers })
}
```

---

## Next.js + .NET Core API

```typescript
// app/api/dotnet/[...slug]/route.ts
import { NextRequest } from 'next/server'

export async function handler(request: NextRequest, { params }) {
  const targetUrl = `${process.env.DOTNET_API_URL}/${params.slug.join('/')}`
  
  const headers = new Headers()
  request.headers.forEach((value, key) => {
    if (!['host', 'connection'].includes(key.toLowerCase())) {
      headers.set(key, value)
    }
  })

  const response = await fetch(targetUrl, {
    method: request.method,
    headers,
    body: request.method !== 'GET' ? request.body : undefined,
  })

  return new Response(response.body, {
    status: response.status,
    headers: response.headers,
  })
}

export { handler as GET, handler as POST, handler as PUT, handler as DELETE }
```

---

## Next.js + GraphQL (Apollo, Hasura)

GraphQL has a single endpoint (`/graphql`), making proxy setup simple.

```javascript
// next.config.js
module.exports = {
  async rewrites() {
    return [
      {
        source: '/graphql',
        destination: process.env.GRAPHQL_ENDPOINT,
      },
    ]
  },
}
```

For Hasura with subscriptions (WebSockets), use the Cloudflare Worker proxy.

---

## Next.js + Serverless (AWS Lambda)

If your Lambda functions are behind API Gateway:

```javascript
// next.config.js
module.exports = {
  async rewrites() {
    return [
      {
        source: '/api/serverless/:path*',
        destination: 'https://your-api-id.execute-api.ap-south-1.amazonaws.com/prod/:path*',
      },
    ]
  },
}
```

The rewrite runs on Vercel's server   the ISP never sees the Lambda URL.

---

## Summary: When to Use Which Pattern

| Backend | Approach | WebSocket Support |
|:--------|:---------|:-----------------|
| Supabase | Cloudflare Worker | Yes |
| Node/Express | Next.js API Route proxy OR CF Worker | No (unless CF Worker) |
| Django | next.config rewrites (HTTP) + CF Worker (WS) | CF Worker only |
| Laravel | next.config rewrites | No |
| Spring Boot | Next.js API Route catch-all | No |
| .NET | Next.js API Route catch-all | No |
| GraphQL HTTP | next.config rewrites | N/A |
| GraphQL WS | Cloudflare Worker | Yes |


