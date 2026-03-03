# AI API Backends   Proxy and Fallback

## The Problem

AI API keys (`sk-...` for OpenAI, `AIza...` for Gemini, etc.) must never exist client-side. Even if an ISP doesn't block OpenAI/Anthropic, the correct architecture routes ALL AI calls through your backend. This eliminates:

1. **API key exposure** in browser DevTools
2. **ISP blocking** of `api.openai.com`, `api.anthropic.com`, `generativelanguage.googleapis.com`
3. **Rate limit abuse**   you control client access
4. **Cost control**   your backend enforces budgets

---

## Architecture

```
User → your-api.com/ai/chat → Backend (Node/Python/etc.) → OpenAI / Anthropic / Gemini
```

---

## Node.js / Express Backend Proxy

```typescript
// src/ai-proxy.ts
import express from 'express'
import OpenAI from 'openai'
import Anthropic from '@anthropic-ai/sdk'

const router = express.Router()

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY })
const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY })

// Unified endpoint   selects provider from request
router.post('/ai/chat', async (req, res) => {
  const { messages, provider = 'openai', model } = req.body
  
  // Basic rate limiting
  const userId = req.headers['x-user-id'] as string
  if (!userId) return res.status(401).json({ error: 'Unauthorized' })
  
  try {
    if (provider === 'openai') {
      const completion = await openai.chat.completions.create({
        model: model ?? 'gpt-4o-mini',
        messages,
        stream: false,
      })
      return res.json({ content: completion.choices[0].message.content })
    }
    
    if (provider === 'anthropic') {
      const message = await anthropic.messages.create({
        model: model ?? 'claude-3-haiku-20240307',
        max_tokens: 1024,
        messages,
      })
      return res.json({ content: message.content[0].type === 'text' ? message.content[0].text : '' })
    }
    
    res.status(400).json({ error: 'Unknown provider' })
  } catch (err: any) {
    res.status(500).json({ error: err.message })
  }
})

// Streaming endpoint
router.post('/ai/stream', async (req, res) => {
  const { messages, provider = 'openai' } = req.body
  
  res.setHeader('Content-Type', 'text/event-stream')
  res.setHeader('Cache-Control', 'no-cache')
  res.setHeader('Connection', 'keep-alive')
  
  if (provider === 'openai') {
    const stream = await openai.chat.completions.create({
      model: 'gpt-4o-mini',
      messages,
      stream: true,
    })
    
    for await (const chunk of stream) {
      const text = chunk.choices[0]?.delta?.content ?? ''
      if (text) res.write(`data: ${JSON.stringify({ text })}\n\n`)
    }
    
    res.write('data: [DONE]\n\n')
    res.end()
  }
})

export default router
```

---

## Python / FastAPI Backend

```python
# main.py
from fastapi import FastAPI, HTTPException, Header
from fastapi.responses import StreamingResponse
import openai
import anthropic
import os
import json
from typing import List, Optional

app = FastAPI()

openai_client = openai.AsyncOpenAI(api_key=os.environ["OPENAI_API_KEY"])
anthropic_client = anthropic.AsyncAnthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

@app.post("/ai/chat")
async def chat(payload: dict, x_user_id: Optional[str] = Header(None)):
    if not x_user_id:
        raise HTTPException(status_code=401, detail="Unauthorized")
    
    provider = payload.get("provider", "openai")
    messages = payload.get("messages", [])
    
    if provider == "openai":
        response = await openai_client.chat.completions.create(
            model=payload.get("model", "gpt-4o-mini"),
            messages=messages,
        )
        return {"content": response.choices[0].message.content}
    
    elif provider == "anthropic":
        message = await anthropic_client.messages.create(
            model=payload.get("model", "claude-3-haiku-20240307"),
            max_tokens=1024,
            messages=messages,
        )
        return {"content": message.content[0].text}
    
    raise HTTPException(status_code=400, detail="Unknown provider")


@app.post("/ai/stream")
async def stream_chat(payload: dict):
    messages = payload.get("messages", [])
    
    async def generate():
        async with openai_client.chat.completions.stream(
            model="gpt-4o-mini",
            messages=messages,
        ) as stream:
            async for chunk in stream:
                text = chunk.choices[0].delta.content or ""
                if text:
                    yield f"data: {json.dumps({'text': text})}\n\n"
        yield "data: [DONE]\n\n"
    
    return StreamingResponse(generate(), media_type="text/event-stream")
```

---

## Multi-Provider Fallback

When one AI provider is blocked or rate-limited, fall back to the next:

```typescript
// src/ai-fallback.ts
const PROVIDERS = [
  {
    name: 'openai',
    call: async (messages: any[]) => {
      const r = await openai.chat.completions.create({ model: 'gpt-4o-mini', messages })
      return r.choices[0].message.content
    },
  },
  {
    name: 'anthropic',
    call: async (messages: any[]) => {
      const r = await anthropic.messages.create({ model: 'claude-3-haiku-20240307', max_tokens: 1024, messages })
      return r.content[0].type === 'text' ? r.content[0].text : ''
    },
  },
  {
    name: 'gemini',
    call: async (messages: any[]) => {
      // google-generativeai
      const model = genAI.getGenerativeModel({ model: 'gemini-1.5-flash' })
      const prompt = messages.map((m) => m.content).join('\n')
      const result = await model.generateContent(prompt)
      return result.response.text()
    },
  },
]

export async function callWithFallback(messages: any[]): Promise<string> {
  for (const provider of PROVIDERS) {
    try {
      return await provider.call(messages)
    } catch (err: any) {
      console.warn(`Provider ${provider.name} failed:`, err.message)
    }
  }
  throw new Error('All AI providers failed')
}
```

---

## Response Caching with Redis

```typescript
import { createClient as createRedisClient } from 'redis'

const redis = createRedisClient({ url: process.env.REDIS_URL })

export async function cachedAI(messages: any[], ttl = 300): Promise<string> {
  const cacheKey = `ai:${JSON.stringify(messages)}`
  
  const cached = await redis.get(cacheKey)
  if (cached) {
    return cached
  }
  
  const response = await callWithFallback(messages)
  await redis.setEx(cacheKey, ttl, response)
  return response
}
```

---

## Frontend Client (React)

```typescript
// hooks/useAI.ts
import { useState } from 'react'

interface Message { role: 'user' | 'assistant'; content: string }

export function useAI() {
  const [loading, setLoading] = useState(false)
  const [messages, setMessages] = useState<Message[]>([])
  
  const send = async (content: string) => {
    const newMessages = [...messages, { role: 'user' as const, content }]
    setMessages(newMessages)
    setLoading(true)
    
    const res = await fetch('/api/ai/chat', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'X-User-Id': 'user-123' },
      body: JSON.stringify({ messages: newMessages }),
    })
    
    const data = await res.json()
    setMessages([...newMessages, { role: 'assistant', content: data.content }])
    setLoading(false)
  }
  
  const stream = async (content: string) => {
    const newMessages = [...messages, { role: 'user' as const, content }]
    setMessages(newMessages)
    setLoading(true)
    
    const res = await fetch('/api/ai/stream', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ messages: newMessages }),
    })
    
    const reader = res.body!.getReader()
    const decoder = new TextDecoder()
    let assistantText = ''
    
    while (true) {
      const { done, value } = await reader.read()
      if (done) break
      
      const chunk = decoder.decode(value)
      for (const line of chunk.split('\n')) {
        if (line.startsWith('data: ')) {
          const data = line.slice(6)
          if (data === '[DONE]') break
          try {
            assistantText += JSON.parse(data).text
          } catch {}
        }
      }
    }
    
    setMessages([...newMessages, { role: 'assistant', content: assistantText }])
    setLoading(false)
  }
  
  return { messages, loading, send, stream }
}
```

---

## Environment Variables

```
# .env.local / Vercel / Railway
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
GOOGLE_AI_KEY=AIza...
REDIS_URL=redis://localhost:6379

# These must NEVER have NEXT_PUBLIC_, VITE_, or PUBLIC_ prefix
```

---

## Cost Control Headers

```typescript
// Middleware to track per-user AI costs
app.use('/ai', async (req, res, next) => {
  const userId = req.headers['x-user-id']
  const usage = await redis.incr(`ai:usage:${userId}:${new Date().toISOString().slice(0, 10)}`)
  
  if (usage > 50) {  // 50 requests/day limit
    return res.status(429).json({ error: 'Daily AI limit exceeded' })
  }
  
  next()
})
```


