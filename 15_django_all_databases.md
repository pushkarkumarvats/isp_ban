# Django With All Database Types

## The Django Advantage

Django runs entirely on the server. Database connections happen server-side over TCP   not through the browser's DNS resolver. ISP DNS poisoning does NOT affect Django's database connections directly.

**When Django IS affected**: When your Django server is deployed on an IP that the ISP blocks (rare, typically only for residential IPs or very targeted blocking). Most cloud VPS/servers are unaffected.

**When you DO need a proxy**: When Django itself is deployed on a shared IP range that gets blocked, or when your Django API serves a JavaScript frontend that makes additional direct calls to external services.

---

## Django + PostgreSQL (Standard)

No ISP bypass needed for the Django-to-PostgreSQL connection since it uses direct TCP, not HTTP DNS resolution through the browser.

```python
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('DB_NAME', 'mydb'),
        'USER': os.environ.get('DB_USER', 'postgres'),
        'PASSWORD': os.environ.get('DB_PASSWORD'),
        'HOST': os.environ.get('DB_HOST', 'localhost'),  # Private IP or hostname
        'PORT': os.environ.get('DB_PORT', '5432'),
    }
}
```

**Security note**: Always put PostgreSQL in a private subnet. Never expose port 5432 to the public internet.

---

## Django + Supabase (via supabase-py)

For complete guide see [../supabase_django.md](../supabase_django.md). Summary:

```python
# settings.py / .env
SUPABASE_URL = os.environ.get("SUPABASE_URL")
SUPABASE_PROXY_URL = os.environ.get("SUPABASE_PROXY_URL", "")
SUPABASE_KEY = os.environ.get("SUPABASE_KEY")
```

```python
# services/supabase_client.py
import os
from supabase import create_client, Client

def get_supabase_client() -> Client:
    url = os.environ.get("SUPABASE_PROXY_URL") or os.environ.get("SUPABASE_URL")
    key = os.environ.get("SUPABASE_KEY")
    return create_client(url, key)

supabase: Client = get_supabase_client()
```

### Supabase as Django Backend (Postgres via postrgres://)

You can also connect directly to Supabase's PostgreSQL using Django's ORM (bypasses supabase-py entirely):

```python
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'postgres',
        'USER': 'postgres',
        'PASSWORD': os.environ.get('SUPABASE_DB_PASSWORD'),
        'HOST': os.environ.get('SUPABASE_DB_HOST'),  # db.your-project.supabase.co
        'PORT': '5432',
        # If ISP blocks TCP on 5432, use connection pooler (port 6543)
        # 'PORT': '6543',
    }
}
```

**ISP bypass for connection pooler**: Supabase's connection pooler (PgBouncer) runs on port 6543. If port 5432 is blocked but port 6543 is not, switch to the pooler. If both are blocked, use an SSH tunnel through a clean VPS.

---

## Django + MySQL

```python
# settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': os.environ['MYSQL_DB'],
        'USER': os.environ['MYSQL_USER'],
        'PASSWORD': os.environ['MYSQL_PASSWORD'],
        'HOST': os.environ['MYSQL_HOST'],  # Private IP
        'PORT': '3306',
        'OPTIONS': {
            'ssl': {'ssl-mode': 'REQUIRED'},
        }
    }
}
```

---

## Django + MongoDB

```python
# settings.py (using djongo or mongoengine)
DATABASES = {
    'default': {
        'ENGINE': 'djongo',
        'NAME': os.environ['MONGO_DB_NAME'],
        'ENFORCE_SCHEMA': False,
        'CLIENT': {
            'host': os.environ['MONGO_URI'],  # mongodb+srv://... private
        }
    }
}
```

---

## Django REST API Behind Nginx (ISP-Safe Deployment)

When your Django API serves a JavaScript frontend, users make API calls from the browser to your Django API. Your API must be behind a CDN/proxy so the ISP cannot block its IP.

### Nginx + Cloudflare Setup

```
JS App → Cloudflare (orange-cloud) → yourproduct.com/api/* → Nginx → Gunicorn → Django
```

```nginx
# /etc/nginx/sites-available/django
upstream django {
    server 127.0.0.1:8000;  # Gunicorn
}

server {
    listen 443 ssl http2;
    server_name api.yourproduct.com;

    ssl_certificate /etc/letsencrypt/live/api.yourproduct.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.yourproduct.com/privkey.pem;

    location / {
        proxy_pass http://django;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

### Gunicorn Production Command

```bash
gunicorn myproject.wsgi:application \
  --bind 127.0.0.1:8000 \
  --workers 4 \
  --timeout 120 \
  --access-logfile /var/log/gunicorn/access.log \
  --error-logfile /var/log/gunicorn/error.log
```

---

## Django Channels (WebSocket) Behind Proxy

Django Channels requires ASGI. Daphne or Uvicorn act as the ASGI server.

```bash
daphne -b 127.0.0.1 -p 8001 myproject.asgi:application
```

Nginx WebSocket proxy:

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

## Django + Redis Cache (Session & Cache)

Redis for caching in a multi-region ISP bypass setup:

```python
# settings.py
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.redis.RedisCache",
        "LOCATION": os.environ.get("REDIS_URL", "redis://127.0.0.1:6379/1"),
    }
}

SESSION_ENGINE = "django.contrib.sessions.backends.cache"
SESSION_CACHE_ALIAS = "default"
```

Upstash Redis (free tier, globally distributed):

```bash
REDIS_URL=rediss://default:your-password@your-endpoint.upstash.io:6379
```

---

## Environment Variables Checklist

```bash
# .env.production
SECRET_KEY=your-django-secret-key
DEBUG=False
ALLOWED_HOSTS=api.yourproduct.com,yourproduct.com

# Database
DB_HOST=10.0.0.5           # Private IP   not public
DB_NAME=mydb
DB_USER=myuser
DB_PASSWORD=...

# Supabase (if using)
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_PROXY_URL=https://your-worker.workers.dev
SUPABASE_KEY=your-key

# Redis
REDIS_URL=rediss://...

# Your public API domain (behind Cloudflare)
CORS_ALLOWED_ORIGINS=https://yourproduct.com
```


