# Web3 / Blockchain Apps   RPC Proxy and Fallback

## The Problem

Blockchain apps call RPC endpoints directly from the browser:

```javascript
const provider = new ethers.providers.JsonRpcProvider('https://mainnet.infura.io/v3/YOUR_KEY')
```

This exposes:
1. **API key in source code** (`YOUR_KEY` is visible in DevTools)
2. **Blocked RPCs**   `mainnet.infura.io`, `eth-mainnet.alchemyapi.io` may be blocked by ISPs
3. **Single point of failure**   one provider down = app broken

---

## The Correct Architecture

```
Browser → your-rpc.yourproduct.com → Backend RPC Proxy → Infura / Alchemy / QuickNode
```

---

## Cloudflare Worker: Universal RPC Proxy

This single Worker handles Ethereum JSON-RPC for any EVM chain:

```typescript
// rpc-proxy.worker.ts
interface Env {
  INFURA_KEY: string
  ALCHEMY_ETH_KEY: string
  QUICKNODE_URL: string
}

const ENDPOINTS: Record<string, string[]> = {
  ethereum: [
    'https://mainnet.infura.io/v3/{INFURA_KEY}',
    'https://eth-mainnet.g.alchemy.com/v2/{ALCHEMY_KEY}',
  ],
  polygon: [
    'https://polygon-mainnet.infura.io/v3/{INFURA_KEY}',
    'https://polygon-mainnet.g.alchemy.com/v2/{ALCHEMY_KEY}',
  ],
  bsc: [
    'https://bsc-dataseed.binance.org',
    'https://bsc-dataseed1.defibit.io',
  ],
  solana: [
    'https://api.mainnet-beta.solana.com',
    'https://rpc.ankr.com/solana',
  ],
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    if (request.method === 'OPTIONS') {
      return new Response(null, {
        headers: corsHeaders(request.headers.get('Origin') ?? '*'),
      })
    }
    
    const url = new URL(request.url)
    const chain = url.pathname.slice(1) || 'ethereum'  // /ethereum, /polygon, etc.
    
    const endpoints = (ENDPOINTS[chain] || ENDPOINTS.ethereum).map((e) =>
      e.replace('{INFURA_KEY}', env.INFURA_KEY).replace('{ALCHEMY_KEY}', env.ALCHEMY_ETH_KEY)
    )
    
    const body = await request.text()
    
    for (const endpoint of endpoints) {
      try {
        const response = await fetch(endpoint, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body,
        })
        
        if (response.ok) {
          return new Response(response.body, {
            status: response.status,
            headers: {
              'Content-Type': 'application/json',
              ...corsHeaders(request.headers.get('Origin') ?? '*'),
            },
          })
        }
      } catch {
        // Try next endpoint
      }
    }
    
    return new Response(JSON.stringify({ error: 'All RPC endpoints failed' }), {
      status: 503,
      headers: { 'Content-Type': 'application/json' },
    })
  },
}

function corsHeaders(origin: string) {
  return {
    'Access-Control-Allow-Origin': origin,
    'Access-Control-Allow-Methods': 'POST, OPTIONS',
    'Access-Control-Allow-Headers': 'Content-Type',
  }
}
```

Deploy with custom domain:

```toml
# wrangler.toml
name = "rpc-proxy"
compatibility_date = "2024-01-01"

[vars]
# Set INFURA_KEY and ALCHEMY_ETH_KEY as secrets:
# npx wrangler secret put INFURA_KEY

[[routes]]
pattern = "rpc.yourproduct.com/*"
zone_name = "yourproduct.com"
```

---

## Frontend: ethers.js with Proxy

```typescript
// lib/web3.ts
import { ethers } from 'ethers'

// Use your proxy instead of direct RPC
const RPC_URL = process.env.NEXT_PUBLIC_RPC_URL || 'https://rpc.yourproduct.com/ethereum'

export const provider = new ethers.JsonRpcProvider(RPC_URL)

// For multiple chains
export const providers: Record<string, ethers.JsonRpcProvider> = {
  ethereum: new ethers.JsonRpcProvider(RPC_URL),
  polygon: new ethers.JsonRpcProvider(RPC_URL.replace('/ethereum', '/polygon')),
  bsc: new ethers.JsonRpcProvider(RPC_URL.replace('/ethereum', '/bsc')),
}
```

---

## Frontend: wagmi + viem

```typescript
// lib/wagmi-config.ts
import { createConfig, http } from 'wagmi'
import { mainnet, polygon, bsc } from 'wagmi/chains'

const RPC_BASE = process.env.NEXT_PUBLIC_RPC_URL ?? 'https://rpc.yourproduct.com'

export const config = createConfig({
  chains: [mainnet, polygon, bsc],
  transports: {
    [mainnet.id]: http(`${RPC_BASE}/ethereum`),
    [polygon.id]: http(`${RPC_BASE}/polygon`),
    [bsc.id]: http(`${RPC_BASE}/bsc`),
  },
})
```

---

## Solana: Proxy with @solana/web3.js

```typescript
// lib/solana.ts
import { Connection } from '@solana/web3.js'

const RPC_URL = process.env.NEXT_PUBLIC_SOLANA_RPC ?? 'https://rpc.yourproduct.com/solana'

export const connection = new Connection(RPC_URL, 'confirmed')
```

Node.js backend proxy for Solana:

```javascript
// server/solana-proxy.js
const express = require('express')
const httpProxy = require('http-proxy')

const router = express.Router()
const proxy = httpProxy.createProxyServer()

const SOLANA_ENDPOINTS = [
  'https://api.mainnet-beta.solana.com',
  'https://rpc.ankr.com/solana',
  'https://solana-mainnet.core.chainstack.com',
]

let currentEndpoint = 0

router.use('/solana', (req, res) => {
  proxy.web(req, res, {
    target: SOLANA_ENDPOINTS[currentEndpoint % SOLANA_ENDPOINTS.length],
    changeOrigin: true,
  })
  
  proxy.on('error', () => {
    currentEndpoint++
    res.status(503).json({ error: 'RPC endpoint failed, rotated to next' })
  })
})

module.exports = router
```

---

## WebSocket RPC (eth_subscribe)

Some features require WebSocket RPCs (real-time block subscriptions):

```typescript
// lib/ws-provider.ts
const WS_RPC = process.env.NEXT_PUBLIC_WS_RPC ?? 'wss://ws.yourproduct.com/ethereum'

export function createWsProvider() {
  const ws = new ethers.WebSocketProvider(WS_RPC)
  
  ws.on('block', (blockNumber) => {
    console.log('New block:', blockNumber)
  })
  
  return ws
}
```

WebSocket RPC proxy (Cloudflare Worker with WebSocket):

```typescript
// ws-rpc-proxy.worker.ts
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const upgradeHeader = request.headers.get('Upgrade')
    
    if (upgradeHeader === 'websocket') {
      const [client, server] = Object.values(new WebSocketPair())
      
      const upstreamWs = new WebSocket('wss://mainnet.infura.io/ws/v3/' + env.INFURA_KEY)
      
      // Bidirectional relay
      client.accept()
      
      upstreamWs.addEventListener('message', (e) => client.send(e.data))
      upstreamWs.addEventListener('close', () => client.close())
      client.addEventListener('message', (e) => upstreamWs.send(e.data))
      client.addEventListener('close', () => upstreamWs.close())
      
      return new Response(null, { status: 101, webSocket: client })
    }
    
    return new Response('Expected WebSocket', { status: 400 })
  },
}
```

---

## Rate Limit and Auth

Protect your proxy from abuse:

```typescript
// Cloudflare Worker rate limiting
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const ip = request.headers.get('CF-Connecting-IP') ?? 'unknown'
    const key = `rpc:${ip}:${new Date().toISOString().slice(0, 13)}`  // per-hour key
    
    const count = await env.KV.get(key)
    const requests = parseInt(count ?? '0')
    
    if (requests > 1000) {  // 1000 RPC calls/hour per IP
      return new Response('Rate limit exceeded', { status: 429 })
    }
    
    await env.KV.put(key, String(requests + 1), { expirationTtl: 3600 })
    
    // ... proxy logic
  },
}
```

---

## RPC Provider Comparison

| Provider | Free Tier | WebSocket | Best For |
|:---------|:----------|:----------|:---------|
| Infura | 100k req/day | Yes | Ethereum mainnet |
| Alchemy | 300M compute units | Yes | Development |
| QuickNode | 10M req/month | Yes | Production |
| Ankr | 500 req/s | Yes | Multi-chain |
| Public RPCs | Unlimited | Sometimes | Fallback only |

Always test fallback chains before going to production.


