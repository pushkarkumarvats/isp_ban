# GraphQL Systems

## Overview

GraphQL APIs have a single endpoint (`/graphql`)   this makes proxying simpler than REST. The ISP bypass strategies are the same as REST, but GraphQL has specific considerations for subscriptions (WebSockets).

**Affected GraphQL platforms**: Apollo Server, Hasura, Graphene (Python), NestJS GraphQL, WPGraphQL, Pothos.

---

## The Three GraphQL Request Types

| Type | Transport | ISP Bypass Method |
|:-----|:----------|:-----------------|
| Query | HTTP POST | Next.js rewrite, Nginx proxy, CF Worker |
| Mutation | HTTP POST | Same as Query |
| Subscription | WebSocket | CF Worker (WebSocket support required) |

---

## Solution 1: Cloudflare Worker for GraphQL

A single Worker handles queries, mutations, and subscriptions.

```javascript
// cloudflare-worker.js
export default {
  async fetch(request, env) {
    const GRAPHQL_ORIGIN = env.GRAPHQL_ORIGIN; // https://api.yourcompany.com

    const url = new URL(request.url);
    const originUrl = new URL(GRAPHQL_ORIGIN);
    url.hostname = originUrl.hostname;
    url.protocol = originUrl.protocol;
    url.port = originUrl.port || '';

    const headers = new Headers(request.headers);
    headers.set('host', originUrl.hostname);

    // WebSocket (subscriptions)
    if (request.headers.get('Upgrade') === 'websocket') {
      return fetch(url.toString(), {
        method: request.method,
        headers,
        body: request.body,
      });
    }

    // HTTP (queries and mutations)
    return fetch(new Request(url.toString(), {
      method: request.method,
      headers,
      body: request.body,
    }));
  },
};
```

---

## Solution 2: Apollo Client with Dynamic URI

```typescript
// lib/apolloClient.ts
import { ApolloClient, InMemoryCache, split, HttpLink } from '@apollo/client'
import { GraphQLWsLink } from '@apollo/client/link/subscriptions'
import { createClient } from 'graphql-ws'
import { getMainDefinition } from '@apollo/client/utilities'

const GRAPHQL_HTTP_URL = process.env.NEXT_PUBLIC_GRAPHQL_PROXY_URL 
  || process.env.NEXT_PUBLIC_GRAPHQL_URL
  || 'https://your-api.com/graphql'

const GRAPHQL_WS_URL = (GRAPHQL_HTTP_URL)
  .replace('https://', 'wss://')
  .replace('http://', 'ws://')

const httpLink = new HttpLink({
  uri: GRAPHQL_HTTP_URL,
  credentials: 'include',
})

const wsLink = new GraphQLWsLink(
  createClient({
    url: GRAPHQL_WS_URL,
    connectionParams: {
      authToken: () => localStorage.getItem('auth_token'),
    },
  })
)

// Send subscriptions over WS, queries/mutations over HTTP
const splitLink = split(
  ({ query }) => {
    const definition = getMainDefinition(query)
    return definition.kind === 'OperationDefinition' 
      && definition.operation === 'subscription'
  },
  wsLink,
  httpLink
)

export const apolloClient = new ApolloClient({
  link: splitLink,
  cache: new InMemoryCache(),
})
```

```bash
# .env.production
NEXT_PUBLIC_GRAPHQL_PROXY_URL=https://your-worker.workers.dev/graphql
```

---

## Solution 3: Apollo Server Proxy

If you control the Apollo Server, add a standalone proxy endpoint:

```typescript
// server.ts (Apollo Server with Express)
import express from 'express'
import { createProxyMiddleware } from 'http-proxy-middleware'
import { ApolloServer } from '@apollo/server'
import { expressMiddleware } from '@apollo/server/express4'

const app = express()

// Proxy external blocked GraphQL API
app.use('/proxy/graphql', createProxyMiddleware({
  target: 'https://blocked-graphql-api.com',
  changeOrigin: true,
  pathRewrite: { '^/proxy/graphql': '/graphql' },
  on: {
    proxyReq: (proxyReq) => {
      proxyReq.setHeader('host', 'blocked-graphql-api.com')
    }
  }
}))

// Your own Apollo Server
const server = new ApolloServer({ typeDefs, resolvers })
await server.start()
app.use('/graphql', expressMiddleware(server))

app.listen(4000)
```

---

## Solution 4: Hasura with Cloudflare

Hasura Cloud (hosted) uses `*.hasura.app` domain   potentially blocked.

```javascript
// CF Worker for Hasura
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    url.hostname = env.HASURA_HOSTNAME;  // your-app.hasura.app

    const headers = new Headers(request.headers);
    headers.set('host', env.HASURA_HOSTNAME);
    headers.set('x-hasura-admin-secret', env.HASURA_ADMIN_SECRET);

    if (request.headers.get('Upgrade') === 'websocket') {
      return fetch(url.toString(), { method: request.method, headers });
    }

    return fetch(new Request(url.toString(), { 
      method: request.method, headers, body: request.body 
    }));
  },
};
```

### Hasura Client with Proxy

```typescript
// lib/hasura.ts
import { GraphQLClient } from 'graphql-request'

const HASURA_URL = process.env.NEXT_PUBLIC_HASURA_PROXY_URL 
  || process.env.NEXT_PUBLIC_HASURA_URL

export const hasuraClient = new GraphQLClient(HASURA_URL!, {
  headers: {
    'x-hasura-role': 'user',
  },
})
```

---

## Solution 5: Next.js API Route for GraphQL Proxy

```typescript
// app/api/graphql/route.ts
import { NextRequest, NextResponse } from 'next/server'

const GRAPHQL_ORIGIN = process.env.GRAPHQL_ORIGIN  // Server-side env var

export async function POST(req: NextRequest) {
  const body = await req.text()
  
  const response = await fetch(`${GRAPHQL_ORIGIN}/graphql`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': req.headers.get('Authorization') || '',
    },
    body,
  })

  const data = await response.json()
  return NextResponse.json(data, { status: response.status })
}
```

Client uses `/api/graphql`   same origin, no ISP block.

---

## Disable Introspection in Production

Always disable GraphQL introspection in production to prevent schema leakage through proxies:

```typescript
// Apollo Server
const server = new ApolloServer({
  typeDefs,
  resolvers,
  introspection: process.env.NODE_ENV !== 'production',
})
```

```javascript
// Hasura
// Set HASURA_GRAPHQL_DISABLE_INTROSPECTION=true in environment
```

---

## GraphQL Subscriptions via SSE (Alternative to WebSocket)

If WebSocket proxy is complex, GraphQL over SSE is simpler and works through standard HTTP proxies:

```typescript
// Apollo Client with SSE transport
import { ApolloClient, InMemoryCache } from '@apollo/client'
import { Client, createClient } from 'graphql-sse'
import { GraphQLSSELink } from '@apollo/client/link/subscriptions'

const sseClient = createClient({
  url: `${process.env.NEXT_PUBLIC_GRAPHQL_PROXY_URL}/graphql/stream`,
})

export const apolloClient = new ApolloClient({
  link: new GraphQLSSELink(sseClient),
  cache: new InMemoryCache(),
})
```

SSE works through any HTTP proxy including Vercel rewrites   unlike WebSocket.


