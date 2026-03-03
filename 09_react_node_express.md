# React + Node.js / Express

## Architecture Overview

```
User (Blocked ISP) → React SPA → Direct API call → BLOCKED
                         ↓
User (Fixed) → React SPA → /api/proxy/* (same origin) → Node/Express → Real Backend
```

The key: React runs in the browser (affected by DNS poisoning). Node.js/Express runs on a server (not affected). Route all sensitive API calls through your own backend, which proxies to blocked services.

---

## Option 1: Express Proxy Middleware

If your Express backend is on the same server or accessible directly (VPS), add a proxy middleware layer.

### Install

```bash
npm install http-proxy-middleware
```

### Express Setup

```javascript
// server.js
const express = require('express')
const { createProxyMiddleware } = require('http-proxy-middleware')
const app = express()

// Proxy Supabase API calls
app.use('/supabase', createProxyMiddleware({
  target: 'https://your-project.supabase.co',
  changeOrigin: true,
  pathRewrite: { '^/supabase': '' },
  secure: true,
  headers: {
    'host': 'your-project.supabase.co',
  },
  on: {
    proxyReq: (proxyReq, req) => {
      // Forward auth headers
      if (req.headers.authorization) {
        proxyReq.setHeader('Authorization', req.headers.authorization)
      }
      if (req.headers.apikey) {
        proxyReq.setHeader('apikey', req.headers.apikey)
      }
    }
  }
}))

// WebSocket proxy for Supabase Realtime
app.use('/supabase/realtime', createProxyMiddleware({
  target: 'wss://your-project.supabase.co',
  changeOrigin: true,
  ws: true,
  pathRewrite: { '^/supabase/realtime': '/realtime' },
}))

app.listen(3001, () => console.log('Proxy running on :3001'))
```

### React Client

```javascript
// src/lib/supabase.js
import { createClient } from '@supabase/supabase-js'

// In development: direct Supabase URL
// In production: your proxy URL
const supabaseUrl = process.env.REACT_APP_SUPABASE_PROXY_URL 
  || process.env.REACT_APP_SUPABASE_URL

export const supabase = createClient(
  supabaseUrl,
  process.env.REACT_APP_SUPABASE_ANON_KEY
)
```

```bash
# .env.production
REACT_APP_SUPABASE_PROXY_URL=https://api.yourproduct.com/supabase
```

---

## Option 2: React + Node API with Fetch Wrapper

Create a centralized API client in React that always routes through your Node backend.

```javascript
// src/lib/apiClient.js
const BASE_URL = process.env.REACT_APP_API_URL || '/api'

class ApiClient {
  constructor(baseUrl) {
    this.baseUrl = baseUrl
  }

  async request(path, options = {}) {
    const response = await fetch(`${this.baseUrl}${path}`, {
      ...options,
      credentials: 'include',
      headers: {
        'Content-Type': 'application/json',
        ...options.headers,
      },
    })

    if (!response.ok) {
      throw new Error(`API Error: ${response.status}`)
    }

    return response.json()
  }

  get(path) { return this.request(path) }
  post(path, data) { return this.request(path, { method: 'POST', body: JSON.stringify(data) }) }
  put(path, data) { return this.request(path, { method: 'PUT', body: JSON.stringify(data) }) }
  delete(path) { return this.request(path, { method: 'DELETE' }) }
}

export const api = new ApiClient(BASE_URL)
```

### Express Routes (server-side)

```javascript
// routes/users.js
const express = require('express')
const { createClient } = require('@supabase/supabase-js')
const router = express.Router()

// These never touch client-side   safe from DNS poisoning
const supabase = createClient(
  process.env.SUPABASE_URL,        // Direct .supabase.co URL, server-side only
  process.env.SUPABASE_SERVICE_KEY // Service role key   NEVER exposed to client
)

router.get('/users', async (req, res) => {
  const { data, error } = await supabase.from('users').select('*')
  if (error) return res.status(500).json({ error: error.message })
  res.json(data)
})

module.exports = router
```

---

## Option 3: Vite Dev Proxy (Development)

During development with Vite, proxy API calls to avoid CORS and ISP issues:

```javascript
// vite.config.js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:3001',
        changeOrigin: true,
      },
      '/supabase': {
        target: 'https://your-project.supabase.co',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/supabase/, ''),
        headers: { host: 'your-project.supabase.co' },
      },
    },
  },
})
```

---

## Option 4: Cloudflare Worker as Standalone Proxy

If you do not have a Node backend, or your Node backend is on a blocked IP, use Cloudflare Workers (see [02_cloudflare_and_alternatives.md](./02_cloudflare_and_alternatives.md)).

Update React to use the worker URL:

```javascript
// src/lib/supabase.js
const SUPABASE_URL = import.meta.env.VITE_SUPABASE_PROXY_URL 
  || import.meta.env.VITE_SUPABASE_URL

export const supabase = createClient(SUPABASE_URL, import.meta.env.VITE_SUPABASE_ANON_KEY)
```

```bash
# .env.production
VITE_SUPABASE_PROXY_URL=https://your-worker.workers.dev
```

---

## CORS Configuration

When your React frontend at `https://yourproduct.com` calls your Express backend at `https://api.yourproduct.com`:

```javascript
// server.js
const cors = require('cors')

app.use(cors({
  origin: [
    'https://yourproduct.com',
    'http://localhost:3000',  // dev
  ],
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization', 'apikey', 'x-client-info'],
}))
```

Also add your proxy domain to Supabase CORS whitelist: **Supabase Dashboard → Settings → API → Allowed CORS origins**.

---

## Monitoring Your Express Proxy

```javascript
// middleware/proxyMonitor.js
const proxyMonitor = (req, res, next) => {
  const start = Date.now()
  res.on('finish', () => {
    const duration = Date.now() - start
    console.log(`[PROXY] ${req.method} ${req.path} → ${res.statusCode} (${duration}ms)`)
    
    // Alert if proxy latency spikes (may indicate ISP interference)
    if (duration > 5000) {
      console.error(`[ALERT] Slow proxy response: ${duration}ms for ${req.path}`)
    }
  })
  next()
}
```


