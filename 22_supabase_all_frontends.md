# Supabase With All Frontends

## Status: Production-Tested

The core Cloudflare Worker proxy approach for Supabase has been tested in production. See the specific guides for detailed implementations:

- [supabase_nextjs.md](../supabase_nextjs.md)   **Tested and confirmed working**
- [supabase_django.md](../supabase_django.md)   Server-side bypass

This document covers every frontend combination with Supabase.

---

## The Universal Cloudflare Worker (All Frontends)

Deploy this once   reuse for all frontend frameworks.

```javascript
// cloudflare-worker/index.js
export default {
  async fetch(request, env) {
    const SUPABASE_HOSTNAME = env.SUPABASE_HOSTNAME; // your-project-id.supabase.co

    const url = new URL(request.url);
    url.hostname = SUPABASE_HOSTNAME;
    url.protocol = 'https:';
    url.port = '';

    const headers = new Headers(request.headers);
    headers.set('host', SUPABASE_HOSTNAME);

    const newRequest = new Request(url.toString(), {
      method: request.method,
      headers: headers,
      body: ['GET', 'HEAD'].includes(request.method) ? null : request.body,
      redirect: 'manual',
    });

    // WebSocket support for Supabase Realtime
    if (request.headers.get('Upgrade') === 'websocket') {
      return fetch(url.toString(), {
        method: request.method,
        headers: headers,
        body: request.body,
      });
    }

    return fetch(newRequest);
  },
};
```

```toml
# wrangler.toml
name = "supabase-proxy"
main = "index.js"
compatibility_date = "2024-01-01"

[vars]
SUPABASE_HOSTNAME = "your-project-id.supabase.co"

[[routes]]
pattern = "api.yourproduct.com/*"
zone_name = "yourproduct.com"
```

---

## Supabase + Next.js

Full guide: [../supabase_nextjs.md](../supabase_nextjs.md)

```typescript
// lib/supabase.ts
import { createBrowserClient } from '@supabase/ssr'

export const createClient = () =>
  createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_PROXY_URL || process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  )
```

---

## Supabase + React (Vite)

```typescript
// src/lib/supabase.ts
import { createClient } from '@supabase/supabase-js'

const supabaseUrl = import.meta.env.VITE_SUPABASE_PROXY_URL 
  || import.meta.env.VITE_SUPABASE_URL

export const supabase = createClient(supabaseUrl, import.meta.env.VITE_SUPABASE_ANON_KEY)
```

```bash
# .env.production
VITE_SUPABASE_PROXY_URL=https://your-worker.workers.dev
VITE_SUPABASE_ANON_KEY=your-anon-key
```

---

## Supabase + Vue 3

```typescript
// src/plugins/supabase.ts
import { createClient, SupabaseClient } from '@supabase/supabase-js'
import { App, inject, InjectionKey, Ref, ref } from 'vue'

const SUPABASE_KEY: InjectionKey<SupabaseClient> = Symbol('supabase')

export function createSupabasePlugin() {
  const url = import.meta.env.VITE_SUPABASE_PROXY_URL 
    || import.meta.env.VITE_SUPABASE_URL
  const key = import.meta.env.VITE_SUPABASE_ANON_KEY
  const client = createClient(url, key)

  return {
    install(app: App) {
      app.provide(SUPABASE_KEY, client)
    }
  }
}

export function useSupabase() {
  const client = inject(SUPABASE_KEY)
  if (!client) throw new Error('Supabase plugin not installed')
  return client
}
```

---

## Supabase + SvelteKit

```typescript
// src/lib/supabase.ts
import { createClient } from '@supabase/supabase-js'
import { PUBLIC_SUPABASE_PROXY_URL, PUBLIC_SUPABASE_URL, PUBLIC_SUPABASE_ANON_KEY } from '$env/static/public'

export const supabase = createClient(
  PUBLIC_SUPABASE_PROXY_URL || PUBLIC_SUPABASE_URL,
  PUBLIC_SUPABASE_ANON_KEY
)
```

---

## Supabase + Flutter (Mobile)

```dart
// lib/supabase_config.dart
import 'package:supabase_flutter/supabase_flutter.dart';
import 'package:flutter_dotenv/flutter_dotenv.dart';

Future<void> initSupabase() async {
  final proxyUrl = dotenv.env['SUPABASE_PROXY_URL'];
  final directUrl = dotenv.env['SUPABASE_URL']!;
  final anonKey = dotenv.env['SUPABASE_ANON_KEY']!;

  await Supabase.initialize(
    url: proxyUrl ?? directUrl,
    anonKey: anonKey,
    // Headers passed for WebSocket connections (Realtime)
    headers: proxyUrl != null 
      ? {'x-proxy-origin': directUrl} 
      : {},
  );
}
```

```
# .env (Flutter)
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_PROXY_URL=https://your-worker.workers.dev
SUPABASE_ANON_KEY=your-anon-key
```

---

## Supabase + React Native

```typescript
// lib/supabase.ts
import { createClient } from '@supabase/supabase-js'
import { MMKV } from 'react-native-mmkv'

const storage = new MMKV()

const ExpoSecureStoreAdapter = {
  getItem: (key: string) => storage.getString(key) ?? null,
  setItem: (key: string, value: string) => storage.set(key, value),
  removeItem: (key: string) => storage.delete(key),
}

const supabaseUrl = process.env.EXPO_PUBLIC_SUPABASE_PROXY_URL 
  || process.env.EXPO_PUBLIC_SUPABASE_URL!

export const supabase = createClient(supabaseUrl, process.env.EXPO_PUBLIC_SUPABASE_ANON_KEY!, {
  auth: {
    storage: ExpoSecureStoreAdapter,
    autoRefreshToken: true,
    persistSession: true,
    detectSessionInUrl: false,
  },
})
```

---

## Supabase + Nuxt 3

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxtjs/supabase'],
  supabase: {
    url: process.env.NUXT_PUBLIC_SUPABASE_PROXY_URL || process.env.NUXT_PUBLIC_SUPABASE_URL,
    key: process.env.NUXT_PUBLIC_SUPABASE_ANON_KEY,
  },
})
```

---

## Supabase Auth: Important CORS Setup

When using a proxy domain, add it to Supabase Auth's allowed redirect URLs:

1. Supabase Dashboard → Authentication → URL Configuration
2. **Site URL**: `https://yourproduct.com`
3. **Redirect URLs**: Add `https://yourproduct.com/**`
4. **CORS**: Settings → API → Add `https://your-worker.workers.dev` if using CF Worker directly

---

## Supabase Storage with Proxy

Images served from Supabase Storage use URLs like:
`https://your-project.supabase.co/storage/v1/object/public/bucket/file.jpg`

These are also blocked. Proxy them:

```typescript
// Next.js: next.config.js
module.exports = {
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'your-worker.workers.dev',  // CF Worker proxies storage too
      },
    ],
  },
  async rewrites() {
    return [
      {
        source: '/storage/:path*',
        destination: `${process.env.SUPABASE_PROXY_URL}/storage/:path*`,
      },
    ]
  },
}
```

---

## Critical Security Warning

When using a proxy:
- **ANON KEY** in client-side code is acceptable   it is designed to be public
- **SERVICE ROLE KEY** must NEVER appear in client-side code or environment variables prefixed with `NEXT_PUBLIC_`, `VITE_`, `PUBLIC_`, or `EXPO_PUBLIC_`
- The proxy adds one hop   ensure your Worker/proxy does not log or store request bodies (which may contain passwords or tokens)


