# Vue.js + Node.js

## Architecture

```
Vue 3 SPA (Vite) → API calls → Node/Express or Cloudflare Worker → Supabase/Backend
```

Vue apps are pure client-side SPAs by default (unless using Nuxt). All API calls originate from the user's browser. Any blocked hostname must be proxied through an unblocked intermediary.

---

## Solution 1: Cloudflare Worker Proxy (Recommended for Vue SPA)

The cleanest solution for Vue SPAs with no Node backend.

### Step 1: Deploy Cloudflare Worker

See the universal worker script in [02_cloudflare_and_alternatives.md](./02_cloudflare_and_alternatives.md).

### Step 2: Configure Vue Environment

```bash
# .env.production
VITE_SUPABASE_PROXY_URL=https://your-worker.workers.dev
VITE_SUPABASE_URL=https://your-project.supabase.co

# .env.development
VITE_SUPABASE_URL=https://your-project.supabase.co
```

### Step 3: Vue Supabase Client

```typescript
// src/lib/supabase.ts
import { createClient } from '@supabase/supabase-js'

const supabaseUrl = import.meta.env.VITE_SUPABASE_PROXY_URL 
  ?? import.meta.env.VITE_SUPABASE_URL

export const supabase = createClient(
  supabaseUrl,
  import.meta.env.VITE_SUPABASE_ANON_KEY
)
```

### Usage in Vue Component

```vue
<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { supabase } from '@/lib/supabase'

const posts = ref([])

onMounted(async () => {
  const { data, error } = await supabase.from('posts').select('*')
  if (error) console.error(error)
  else posts.value = data
})
</script>

<template>
  <div v-for="post in posts" :key="post.id">{{ post.title }}</div>
</template>
```

---

## Solution 2: Vite Proxy for Development

During development, Vite's built-in proxy routes API calls through Node   bypassing ISP block at your machine level.

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  server: {
    proxy: {
      '/supabase': {
        target: 'https://your-project.supabase.co',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/supabase/, ''),
        headers: {
          host: 'your-project.supabase.co',
        },
      },
    },
  },
})
```

Note: This only works during `vite dev`. It does NOT apply to production builds.

---

## Solution 3: Vue + Node.js Backend with http-proxy-middleware

If you have a Node.js backend serving your Vue app:

```javascript
// server.mjs
import express from 'express'
import { createProxyMiddleware } from 'http-proxy-middleware'
import { fileURLToPath } from 'url'
import path from 'path'

const app = express()
const __dirname = path.dirname(fileURLToPath(import.meta.url))

// Serve Vue build
app.use(express.static(path.join(__dirname, 'dist')))

// Proxy Supabase through server
app.use('/api/supabase', createProxyMiddleware({
  target: 'https://your-project.supabase.co',
  changeOrigin: true,
  pathRewrite: { '^/api/supabase': '' },
  headers: { host: 'your-project.supabase.co' },
}))

// SPA fallback
app.get('*', (req, res) => {
  res.sendFile(path.join(__dirname, 'dist', 'index.html'))
})

app.listen(3000)
```

---

## Solution 4: Vue + Pinia Store with Proxy Awareness

Centralize all API configuration in a Pinia store to make switching between proxy and direct URL seamless:

```typescript
// stores/config.ts
import { defineStore } from 'pinia'
import { createClient } from '@supabase/supabase-js'

export const useConfigStore = defineStore('config', {
  state: () => ({
    supabaseUrl: import.meta.env.VITE_SUPABASE_PROXY_URL 
      || import.meta.env.VITE_SUPABASE_URL,
    supabaseKey: import.meta.env.VITE_SUPABASE_ANON_KEY,
  }),
  getters: {
    supabase: (state) => createClient(state.supabaseUrl, state.supabaseKey),
  },
})
```

---

## Vue Router with Auth Guard (Proxy-Aware)

```typescript
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router'
import { supabase } from '@/lib/supabase'

const router = createRouter({
  history: createWebHistory(),
  routes: [
    { path: '/', component: () => import('@/views/Home.vue') },
    { 
      path: '/dashboard', 
      component: () => import('@/views/Dashboard.vue'),
      meta: { requiresAuth: true }
    },
  ],
})

router.beforeEach(async (to) => {
  if (to.meta.requiresAuth) {
    // supabase.auth already uses proxy URL   no special handling needed
    const { data: { session } } = await supabase.auth.getSession()
    if (!session) return { path: '/login' }
  }
})

export default router
```

---

## Realtime Subscriptions with Proxy

Vue Realtime via Cloudflare Worker:

```typescript
// composables/useRealtime.ts
import { supabase } from '@/lib/supabase'
import { ref, onUnmounted } from 'vue'

export function useRealtimePosts() {
  const posts = ref([])

  const channel = supabase
    .channel('posts-changes')
    .on(
      'postgres_changes',
      { event: '*', schema: 'public', table: 'posts' },
      (payload) => {
        console.log('Change received:', payload)
        // Update posts array
        if (payload.eventType === 'INSERT') posts.value.push(payload.new)
        if (payload.eventType === 'DELETE') {
          posts.value = posts.value.filter(p => p.id !== payload.old.id)
        }
      }
    )
    .subscribe()

  onUnmounted(() => {
    supabase.removeChannel(channel)
  })

  return { posts }
}
```

This works because the Cloudflare Worker handles WebSocket upgrades. No additional configuration needed when the proxy URL is set correctly.

---

## CORS for Vue SPA with Separate API Domain

Add your Vue deployment domain to Supabase allowed origins:
- Supabase Dashboard → Settings → API → Add `https://yourproduct.com`

If using a custom proxy domain, add that too.


