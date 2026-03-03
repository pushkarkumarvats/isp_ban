# Nuxt.js With All Backends

## Why Nuxt Has Multiple Layers for ISP Bypass

Nuxt 3 (with Nitro) runs in two environments simultaneously:
- **Server-side (Nitro)**: Node.js or Edge, not affected by ISP DNS poisoning
- **Client-side (Vue 3)**: In the user's browser, affected by DNS poisoning

The strategy is identical to SvelteKit: handle proxied/sensitive API calls in Nuxt server routes. Use Cloudflare Worker only for WebSockets.

---

## Solution 1: Nuxt Server Routes as Proxy (useServerFetch)

```typescript
// server/api/supabase/[...path].ts
import { createClient } from '@supabase/supabase-js'

export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  
  // Server runtime config   never exposed to client
  const supabase = createClient(
    config.supabaseUrl,        // private
    config.supabaseServiceKey  // private
  )

  const path = event.context.params?.path
  const method = event.node.req.method
  const body = method !== 'GET' ? await readBody(event) : undefined
  const query = getQuery(event)

  // Direct Supabase calls from server (no ISP interference)
  const table = path?.split('/')[0]
  let dbQuery = supabase.from(table || '').select(query.select as string || '*')

  const { data, error } = await dbQuery
  
  if (error) {
    throw createError({ statusCode: 500, message: error.message })
  }

  return data
})
```

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  runtimeConfig: {
    // Private   server only
    supabaseUrl: process.env.SUPABASE_URL,
    supabaseServiceKey: process.env.SUPABASE_SERVICE_KEY,

    // Public   exposed to client
    public: {
      supabaseProxyUrl: process.env.NUXT_PUBLIC_SUPABASE_PROXY_URL || '',
      supabaseAnonKey: process.env.NUXT_PUBLIC_SUPABASE_ANON_KEY || '',
    },
  },
})
```

---

## Solution 2: nuxt-supabase Module with Proxy

Using the official `@nuxtjs/supabase` module:

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxtjs/supabase'],
  supabase: {
    url: process.env.NUXT_PUBLIC_SUPABASE_PROXY_URL || process.env.NUXT_PUBLIC_SUPABASE_URL,
    key: process.env.NUXT_PUBLIC_SUPABASE_ANON_KEY,
    redirect: false,
  },
})
```

```bash
# .env
NUXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NUXT_PUBLIC_SUPABASE_PROXY_URL=https://your-worker.workers.dev  # CF Worker
NUXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
```

The module automatically uses the proxy URL for all client-side Supabase calls. Realtime works because the CF Worker handles WebSocket upgrades.

---

## Solution 3: Nuxt Nitro Proxy (Built-in)

Nitro has a built-in proxy handler   no extra dependencies needed.

```typescript
// server/api/proxy/[...path].ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const path = event.context.params?.path || ''
  
  return proxyRequest(event, `${config.supabaseUrl}/${path}`, {
    headers: {
      host: new URL(config.supabaseUrl).hostname,
      apikey: config.supabaseAnonKey,
    },
    fetch,
  })
})
```

---

## Solution 4: Nuxt + Django REST Framework

```typescript
// server/api/django/[...slug].ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const slug = event.context.params?.slug || ''
  const query = getQuery(event)
  const method = event.method
  
  const queryString = new URLSearchParams(query as Record<string, string>).toString()
  const targetUrl = `${config.djangoApiUrl}/api/${slug}${queryString ? `?${queryString}` : ''}`

  const body = ['POST', 'PUT', 'PATCH'].includes(method) 
    ? JSON.stringify(await readBody(event)) 
    : undefined

  const response = await $fetch.raw(targetUrl, {
    method: method as any,
    headers: {
      'Content-Type': 'application/json',
      'Authorization': getHeader(event, 'Authorization') || '',
    },
    body,
  })

  return response._data
})
```

```typescript
// nuxt.config.ts
runtimeConfig: {
  djangoApiUrl: process.env.DJANGO_API_URL,  // server-side only
}
```

---

## Solution 5: Nuxt + Laravel Sanctum API

```typescript
// server/api/laravel/[...path].ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const path = event.context.params?.path || ''
  const cookies = parseCookies(event)

  return proxyRequest(event, `${config.laravelUrl}/${path}`, {
    headers: {
      'X-XSRF-TOKEN': cookies['XSRF-TOKEN'] || '',
      'Cookie': `laravel_session=${cookies['laravel_session'] || ''}`,
    },
  })
})
```

---

## Solution 6: useFetch with Server-Side Proxy

Use Nuxt's `useFetch` with server-side endpoints so the request never hits the browser:

```vue
<!-- pages/index.vue -->
<script setup lang="ts">
// This fetch goes to /api/supabase/posts   your Nuxt server route
// The Nuxt server calls Supabase server-side   ISP bypass automatic
const { data: posts, error } = await useFetch('/api/supabase/posts', {
  query: { select: 'id,title,created_at' }
})
</script>

<template>
  <div v-for="post in posts" :key="post.id">
    {{ post.title }}
  </div>
</template>
```

---

## Nuxt Deployment Options for ISP Bypass

| Deployment | ISP Safe | Cost | Notes |
|:-----------|:---------|:-----|:-------|
| Cloudflare Pages + Workers | Yes | Free | Best option; Nitro supports CF adapter |
| Vercel | Yes | Free | Good; use edge runtime |
| Fly.io (Node) | Yes | Free/low | Mumbai region available |
| Render | Yes | Free | Limited regions |
| Self-hosted VPS + Nginx | Depends | $5-10/mo | Need clean IP |

### Cloudflare Adapter (Most ISP-Resistant)

```bash
npm install @cloudflare/nitro-preset
```

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    preset: 'cloudflare-pages',
  },
})
```

The entire Nuxt app runs as a Cloudflare Worker   no separate CDN needed.


