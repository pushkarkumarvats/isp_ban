# WebSockets and Realtime Systems

## Why WebSockets Are the Hardest to Proxy

HTTP proxying is straightforward   a proxy makes one HTTP request and returns one HTTP response. WebSockets are different: they start as an HTTP request, then perform an "upgrade" to a persistent bidirectional connection. Most HTTP-only proxies (Vercel rewrites, basic load balancers) do not handle this upgrade correctly.

ISPs can block WebSockets by:
- Blocking the initial HTTP upgrade packet
- Injecting a TCP RST after the upgrade
- Refusing `Upgrade: websocket` headers

---

## Protocol Overview

```
Client                    ISP                    Server
  │                        │                        │
  │─── GET /realtime ────>│                        │
  │    Upgrade: websocket  │                        │
  │                        │─── (if not blocked) ──>│
  │                        │                        │ HTTP 101 Switching Protocols
  │<───────────────────────│<──────────────────────│
  │══════ WSS tunnel ══════════════════════════════│
  │       (persistent bidirectional connection)     │
```

---

## Solution 1: Cloudflare Workers (Recommended)

Cloudflare Workers fully support WebSocket proxying. The Worker handles the upgrade handshake and maintains the tunnel.

```javascript
// cloudflare-worker.js   WebSocket proxy
export default {
  async fetch(request, env) {
    const ORIGIN = env.WS_ORIGIN;  // wss://your-server.com

    const upgradeHeader = request.headers.get('Upgrade');
    
    if (upgradeHeader === 'websocket') {
      // WebSocket path
      const url = new URL(request.url);
      const originHostname = new URL(ORIGIN).hostname;
      url.hostname = originHostname;
      url.protocol = 'wss:';
      url.port = '';

      const headers = new Headers(request.headers);
      headers.set('host', originHostname);

      // Cloudflare Workers natively handle WS tunneling
      return fetch(url.toString(), {
        headers,
        method: request.method,
      });
    }

    // Regular HTTP path
    const url = new URL(request.url);
    url.hostname = new URL(ORIGIN).hostname;
    url.protocol = 'https:';
    const headers = new Headers(request.headers);
    headers.set('host', new URL(ORIGIN).hostname);
    
    return fetch(new Request(url.toString(), { 
      method: request.method, headers, body: request.body 
    }));
  },
};
```

---

## Solution 2: Nginx WebSocket Proxy

```nginx
# /etc/nginx/sites-available/ws-proxy
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}

upstream websocket_backend {
    server your-ws-server.com:443;
}

server {
    listen 443 ssl http2;
    server_name ws.yourproduct.com;

    ssl_certificate /etc/letsencrypt/live/ws.yourproduct.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/ws.yourproduct.com/privkey.pem;

    location / {
        proxy_pass https://websocket_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host your-ws-server.com;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_ssl_server_name on;
        
        # Long timeout for persistent connections
        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;
        proxy_connect_timeout 60s;
    }
}
```

---

## Solution 3: Socket.io Behind Proxy

Socket.io uses WebSockets with automatic HTTP long-polling fallback. Configure it to use your proxy URL.

```typescript
// Client
import { io } from 'socket.io-client'

const socket = io(
  process.env.NEXT_PUBLIC_SOCKETIO_PROXY_URL || process.env.NEXT_PUBLIC_SOCKETIO_URL,
  {
    transports: ['websocket', 'polling'],  // polling as fallback if WS blocked
    path: '/socket.io',
    secure: true,
    reconnection: true,
    reconnectionDelay: 1000,
    reconnectionAttempts: 10,
  }
)
```

```javascript
// Server (Node.js)
const { createServer } = require('http')
const { Server } = require('socket.io')
const express = require('express')

const app = express()
const httpServer = createServer(app)

const io = new Server(httpServer, {
  cors: {
    origin: ['https://yourproduct.com', 'http://localhost:3000'],
    credentials: true,
  },
  transports: ['websocket', 'polling'],
})

io.on('connection', (socket) => {
  console.log('Client connected:', socket.id)
  socket.on('message', (data) => io.emit('message', data))
})

httpServer.listen(3001)
```

---

## Solution 4: Supabase Realtime Through CF Worker

```typescript
// Using CF Worker proxy URL for Supabase Realtime
import { createClient } from '@supabase/supabase-js'

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_PROXY_URL!,  // CF Worker URL
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
)

// Realtime subscription   uses WebSocket internally
// CF Worker transparently proxies the WS upgrade
const channel = supabase
  .channel('room1')
  .on('broadcast', { event: 'message' }, (payload) => {
    console.log('Message:', payload)
  })
  .subscribe()
```

---

## Solution 5: Django Channels Behind Nginx + Daphne

```python
# asgi.py
import os
from django.core.asgi import get_asgi_application
from channels.routing import ProtocolTypeRouter, URLRouter
from channels.auth import AuthMiddlewareStack
from . import routing

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'myproject.settings')

application = ProtocolTypeRouter({
    "http": get_asgi_application(),
    "websocket": AuthMiddlewareStack(
        URLRouter(routing.websocket_urlpatterns)
    ),
})
```

```bash
# Run Daphne (ASGI server for Django Channels)
daphne -b 127.0.0.1 -p 8001 myproject.asgi:application
```

Nginx routes WebSockets to Daphne:

```nginx
location /ws/ {
    proxy_pass http://127.0.0.1:8001;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_read_timeout 86400;
}
```

---

## HTTP/3 as WebSocket Alternative

HTTP/3 (QUIC) runs over UDP instead of TCP. ISPs that block TCP-based WebSockets may not block UDP/HTTP3 WebStream connections.

Enable HTTP/3 in Cloudflare:
- Dashboard → Speed → Optimization → HTTP/3 (QUIC) → Enable

And in Nginx (requires nginx >= 1.25):

```nginx
server {
    listen 443 quic reuseport;  # HTTP/3
    listen 443 ssl http2;       # HTTP/2 fallback

    add_header Alt-Svc 'h3=":443"; ma=86400';  # Advertise HTTP/3
}
```

---

## Fallback: Long Polling

When WebSockets cannot be proxied, fall back to HTTP long polling:

```typescript
// Socket.io with polling fallback
const socket = io(PROXY_URL, {
  transports: ['polling'],  // Disable WebSocket, use polling only
  polling: { extraHeaders: { Authorization: `Bearer ${token}` } }
})
```

Long polling sends HTTP requests every 1-2 seconds. Less efficient but works through any HTTP proxy.

---

## WSS vs WS

Always use `wss://` (WebSocket Secure   over TLS) not `ws://`. Non-TLS WebSocket traffic:
- Can be inspected/blocked by DPI
- Is rejected by Cloudflare proxy
- Is a security risk (credentials visible in transit)


