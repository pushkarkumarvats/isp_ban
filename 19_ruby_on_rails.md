# Ruby on Rails

## Overview

Rails is a server-side framework   its database connections use ActiveRecord over TCP and are not affected by browser DNS poisoning. The ISP bypass strategy for Rails is identical to Django/Laravel: put Nginx + Cloudflare in front of your Rails app, and proxy any browser-accessible external API calls through Rails controllers or middleware.

---

## Standard Production Topology

```
Browser → Cloudflare (api.yourproduct.com) → Nginx → Puma (Rails) → PostgreSQL (private)
```

---

## Solution 1: Nginx + Puma + Cloudflare

### Nginx Config (Rails)

```nginx
# /etc/nginx/sites-available/rails
upstream rails_app {
    server 127.0.0.1:3000;
    keepalive 32;
}

server {
    listen 80;
    server_name api.yourproduct.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name api.yourproduct.com;

    ssl_certificate /etc/letsencrypt/live/api.yourproduct.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.yourproduct.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    root /var/www/rails_app/public;
    try_files $uri @rails;

    # WebSocket for ActionCable
    location /cable {
        proxy_pass http://rails_app;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 86400s;
    }

    location @rails {
        proxy_pass http://rails_app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

### Puma Config

```ruby
# config/puma.rb
max_threads_count = ENV.fetch("RAILS_MAX_THREADS") { 5 }
min_threads_count = ENV.fetch("RAILS_MIN_THREADS") { max_threads_count }
threads min_threads_count, max_threads_count

worker_timeout 3600 if ENV.fetch("RAILS_ENV", "development") == "development"

port ENV.fetch("PORT") { 3000 }
environment ENV.fetch("RAILS_ENV") { "development" }
pidfile ENV.fetch("PIDFILE") { "tmp/pids/server.pid" }
workers ENV.fetch("WEB_CONCURRENCY") { 2 }
preload_app!
```

---

## Solution 2: Rails as Proxy for Supabase (faraday)

```ruby
# app/controllers/supabase_proxy_controller.rb
require 'faraday'

class SupabaseProxyController < ApplicationController
  skip_before_action :verify_authenticity_token
  before_action :setup_cors

  SUPABASE_URL = ENV['SUPABASE_URL']
  SUPABASE_KEY = ENV['SUPABASE_ANON_KEY']

  def proxy
    path = params[:path]
    query_string = request.query_string.present? ? "?#{request.query_string}" : ""
    target_url = "#{SUPABASE_URL}/#{path}#{query_string}"

    conn = Faraday.new do |f|
      f.adapter Faraday.default_adapter
    end

    headers = {
      'apikey'        => SUPABASE_KEY,
      'Authorization' => request.headers['Authorization'] || "Bearer #{SUPABASE_KEY}",
      'Content-Type'  => 'application/json',
      'Prefer'        => request.headers['Prefer'] || '',
    }

    body = request.raw_post unless request.get?

    response = conn.run_request(
      request.method.downcase.to_sym,
      target_url,
      body,
      headers
    )

    render json: JSON.parse(response.body), status: response.status
  rescue JSON::ParserError
    render plain: response.body, status: response.status
  end

  private

  def setup_cors
    headers['Access-Control-Allow-Origin'] = request.headers['Origin'] || '*'
    headers['Access-Control-Allow-Methods'] = 'GET, POST, PUT, PATCH, DELETE, OPTIONS'
    headers['Access-Control-Allow-Headers'] = 'Content-Type, Authorization, apikey'
    head :ok if request.options?
  end
end
```

### Routes

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # Proxy for Supabase
  match 'supabase/*path', to: 'supabase_proxy#proxy', via: :all

  # Health check
  get '/health', to: proc { [200, {}, ['OK']] }

  # API routes
  namespace :api do
    namespace :v1 do
      resources :posts
      resources :users
    end
  end
end
```

---

## Solution 3: Rails API-Only Mode (Best for SPA + Rails)

```ruby
# config/application.rb
module YourApp
  class Application < Rails::Application
    config.api_only = true  # No views, cookies, sessions (lighter)
  end
end
```

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
  include ActionController::HttpAuthentication::Token::ControllerMethods
  before_action :authenticate!

  private

  def authenticate!
    token = request.headers['Authorization']&.split(' ')&.last
    # Verify JWT token from Supabase Auth (server-side verification)
    # Using jwt gem: decoded = JWT.decode(token, supabase_jwt_secret)
    head :unauthorized unless token.present?
  end
end
```

---

## Solution 4: ActionCable Behind Cloudflare (WebSocket)

ActionCable uses WebSockets. Cloudflare supports WebSocket proxying   no extra configuration needed if Cloudflare is set to "Full" SSL and WebSockets are enabled in the Cloudflare dashboard.

```ruby
# config/cable.yml
production:
  adapter: redis
  url: <%= ENV.fetch("REDIS_URL") { "redis://localhost:6379/1" } %>
  channel_prefix: yourapp_production
```

```javascript
// React/Vue client
import { createConsumer } from "@rails/actioncable"

// Uses your Cloudflare-proxied domain   not blocked
const consumer = createConsumer("wss://api.yourproduct.com/cable")
```

---

## Multi-Region Rails Deployment

### Fly.io (Recommended for Rails)

```toml
# fly.toml
app = "your-rails-app"
primary_region = "bom"  # Mumbai

[build]
  dockerfile = "Dockerfile"

[env]
  RAILS_ENV = "production"
  RAILS_LOG_TO_STDOUT = "true"

[[services]]
  internal_port = 3000
  protocol = "tcp"

[[vm]]
  memory = "256mb"
  cpu_kind = "shared"
  cpus = 1
```

```bash
# Deploy to Mumbai primary
fly deploy
flyctl regions add sin  # Singapore as secondary
```

### Database Connection Pooling (PgBouncer on Fly)

```bash
flyctl postgres attach --app your-rails-app your-postgres-app
```

---

## CORS Configuration

```ruby
# Gemfile
gem 'rack-cors'
```

```ruby
# config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins ENV.fetch('ALLOWED_ORIGINS', 'http://localhost:3000').split(',')
    resource '*',
      headers: :any,
      methods: %i[get post put patch delete options head],
      credentials: true
  end
end
```


