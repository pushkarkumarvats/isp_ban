# Flask and FastAPI

## Overview

Flask and FastAPI are Python web frameworks that run on the server. Like Django, their server-to-database connections are not affected by client-side DNS poisoning. The risk is:

1. Your server's public IP being blocked (if hosted on a blocked ASN)
2. Your API serving a JavaScript frontend that calls blocked external services

Both frameworks behave identically for ISP bypass purposes.

---

## Standard Deployment: FastAPI/Flask Behind Nginx + Cloudflare

The correct production topology:

```
JS Frontend (Vercel) → api.yourproduct.com (Cloudflare) → Nginx → Uvicorn/Gunicorn → FastAPI/Flask
```

```nginx
# /etc/nginx/sites-available/fastapi
upstream fastapi {
    server 127.0.0.1:8000;
}

server {
    listen 443 ssl http2;
    server_name api.yourproduct.com;

    ssl_certificate /etc/letsencrypt/live/api.yourproduct.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.yourproduct.com/privkey.pem;

    location / {
        proxy_pass http://fastapi;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

---

## FastAPI as Proxy for Supabase

```python
# main.py
import httpx
from fastapi import FastAPI, Request, Response
from fastapi.middleware.cors import CORSMiddleware
import os

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://yourproduct.com", "http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

SUPABASE_URL = os.environ["SUPABASE_URL"]
SUPABASE_KEY = os.environ["SUPABASE_ANON_KEY"]

@app.api_route("/supabase/{path:path}", methods=["GET", "POST", "PUT", "PATCH", "DELETE"])
async def supabase_proxy(path: str, request: Request):
    target_url = f"{SUPABASE_URL}/{path}"
    if request.query_params:
        target_url += f"?{request.query_params}"

    headers = dict(request.headers)
    headers["host"] = SUPABASE_URL.split("//")[1]
    headers.setdefault("apikey", SUPABASE_KEY)

    body = await request.body()

    async with httpx.AsyncClient() as client:
        response = await client.request(
            method=request.method,
            url=target_url,
            headers=headers,
            content=body,
        )

    return Response(
        content=response.content,
        status_code=response.status_code,
        headers=dict(response.headers),
    )


@app.get("/health")
async def health():
    return {"status": "ok"}
```

---

## Flask as Proxy for Supabase

```python
# app.py
import os
import requests
from flask import Flask, request, Response
from flask_cors import CORS

app = Flask(__name__)
CORS(app, origins=["https://yourproduct.com", "http://localhost:3000"])

SUPABASE_URL = os.environ["SUPABASE_URL"]
SUPABASE_KEY = os.environ["SUPABASE_ANON_KEY"]


@app.route("/supabase/<path:path>", methods=["GET", "POST", "PUT", "PATCH", "DELETE"])
def supabase_proxy(path):
    target_url = f"{SUPABASE_URL}/{path}"
    if request.query_string:
        target_url += f"?{request.query_string.decode()}"

    headers = {k: v for k, v in request.headers if k.lower() != "host"}
    headers["host"] = SUPABASE_URL.split("//")[1]
    headers.setdefault("apikey", SUPABASE_KEY)

    resp = requests.request(
        method=request.method,
        url=target_url,
        headers=headers,
        data=request.get_data(),
        allow_redirects=False,
        timeout=30,
    )

    excluded_headers = {"content-encoding", "transfer-encoding", "connection"}
    response_headers = {
        k: v for k, v in resp.headers.items()
        if k.lower() not in excluded_headers
    }

    return Response(resp.content, resp.status_code, response_headers)


@app.get("/health")
def health():
    return {"status": "ok"}


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)
```

---

## FastAPI + Supabase Direct (Server-Side Queries)

Better than proxying: use supabase-py on the server. Frontend calls FastAPI; FastAPI calls Supabase. No proxy needed for the client.

```python
# database.py
import os
from supabase import create_client, AsyncClient
from supabase._async.client import AsyncClient as AsyncSupabase

_client: AsyncSupabase | None = None

async def get_supabase() -> AsyncSupabase:
    global _client
    if _client is None:
        url = os.environ.get("SUPABASE_PROXY_URL") or os.environ["SUPABASE_URL"]
        key = os.environ["SUPABASE_SERVICE_KEY"]
        _client = await create_client(url, key)
    return _client
```

```python
# routers/posts.py
from fastapi import APIRouter, Depends
from database import get_supabase

router = APIRouter(prefix="/posts", tags=["posts"])

@router.get("/")
async def list_posts(supabase=Depends(get_supabase)):
    result = await supabase.table("posts").select("*").execute()
    return result.data

@router.post("/")
async def create_post(post: dict, supabase=Depends(get_supabase)):
    result = await supabase.table("posts").insert(post).execute()
    return result.data[0]
```

---

## FastAPI WebSocket Proxy

For WebSocket-based features (like chat), FastAPI can proxy WebSocket connections:

```python
# websocket_proxy.py
from fastapi import FastAPI, WebSocket
import websockets
import asyncio
import os

app = FastAPI()
WS_ORIGIN = os.environ.get("WS_BACKEND_URL", "wss://your-project.supabase.co")

@app.websocket("/ws/{path:path}")
async def websocket_proxy(websocket: WebSocket, path: str):
    await websocket.accept()
    
    async with websockets.connect(f"{WS_ORIGIN}/{path}") as upstream:
        async def forward_to_upstream():
            async for message in websocket.iter_text():
                await upstream.send(message)
        
        async def forward_to_client():
            async for message in upstream:
                await websocket.send_text(message)
        
        await asyncio.gather(
            forward_to_upstream(),
            forward_to_client(),
        )
```

---

## Deployment

### FastAPI with Uvicorn

```bash
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4 --proxy-headers
```

### Flask with Gunicorn

```bash
gunicorn app:app --bind 0.0.0.0:8000 --workers 4 --timeout 120
```

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'
services:
  app:
    build: .
    environment:
      - SUPABASE_URL=${SUPABASE_URL}
      - SUPABASE_ANON_KEY=${SUPABASE_ANON_KEY}
    expose:
      - "8000"
  
  nginx:
    image: nginx:alpine
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./certs:/etc/nginx/certs
    depends_on:
      - app
```

