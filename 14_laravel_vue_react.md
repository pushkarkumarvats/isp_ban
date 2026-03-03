# Laravel + Vue / React Frontend

## Architecture

```
Vue/React SPA → /api/* (same origin or CDN proxy) → Laravel (blade, API) → DB
```

Laravel is a PHP framework that runs entirely server-side. The PHP code making database calls is never affected by ISP DNS poisoning. The browser-side JS (Vue/React) that makes API calls to external services (Supabase, Firebase) is affected.

**Key principle**: In a Laravel + Vue/React stack, route all blocked-service calls through Laravel. Never call blocked services from the frontend directly.

---

## Solution 1: Laravel as API Proxy

### Laravel Route   Simple HTTP Proxy

```php
<?php
// routes/api.php
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Route;

// Proxy all Supabase REST calls through Laravel
Route::any('/supabase/{path}', function (Request $request, string $path) {
    $supabaseUrl = config('services.supabase.url');
    $supabaseKey = config('services.supabase.anon_key');
    
    $targetUrl = $supabaseUrl . '/' . $path . ($request->getQueryString() 
        ? '?' . $request->getQueryString() 
        : '');

    $response = Http::withHeaders([
        'apikey'        => $supabaseKey,
        'Authorization' => $request->header('Authorization', "Bearer {$supabaseKey}"),
        'Content-Type'  => 'application/json',
        'Prefer'        => $request->header('Prefer', ''),
    ])->send(
        $request->method(),
        $targetUrl,
        ['body' => $request->getContent()]
    );

    return response($response->body(), $response->status())
        ->header('Content-Type', 'application/json');
})->where('path', '.*');
```

### config/services.php

```php
<?php
return [
    'supabase' => [
        'url'          => env('SUPABASE_URL'),
        'anon_key'     => env('SUPABASE_ANON_KEY'),
        'service_key'  => env('SUPABASE_SERVICE_KEY'),
    ],
];
```

### .env

```bash
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_KEY=your-service-key
```

---

## Solution 2: Dedicated Laravel Service Class

```php
<?php
// app/Services/SupabaseService.php
namespace App\Services;

use Illuminate\Support\Facades\Http;
use Illuminate\Http\Client\Response;

class SupabaseService
{
    private string $url;
    private string $key;

    public function __construct()
    {
        $this->url = config('services.supabase.url');
        $this->key = config('services.supabase.service_key');
    }

    private function client(): \Illuminate\Http\Client\PendingRequest
    {
        return Http::withHeaders([
            'apikey'        => $this->key,
            'Authorization' => "Bearer {$this->key}",
            'Content-Type'  => 'application/json',
        ])->baseUrl($this->url);
    }

    public function select(string $table, string $columns = '*'): array
    {
        $response = $this->client()
            ->get("/rest/v1/{$table}", ['select' => $columns]);

        return $response->json();
    }

    public function insert(string $table, array $data): array
    {
        $response = $this->client()
            ->withHeaders(['Prefer' => 'return=representation'])
            ->post("/rest/v1/{$table}", $data);

        return $response->json();
    }

    public function update(string $table, array $data, array $filters): array
    {
        $query = http_build_query(array_map(fn($v) => "eq.{$v}", $filters));
        $response = $this->client()
            ->withHeaders(['Prefer' => 'return=representation'])
            ->patch("/rest/v1/{$table}?{$query}", $data);

        return $response->json();
    }

    public function delete(string $table, array $filters): array
    {
        $query = http_build_query(array_map(fn($v) => "eq.{$v}", $filters));
        return $this->client()
            ->delete("/rest/v1/{$table}?{$query}")
            ->json();
    }
}
```

Register in AppServiceProvider:

```php
// app/Providers/AppServiceProvider.php
use App\Services\SupabaseService;

public function register(): void
{
    $this->app->singleton(SupabaseService::class);
}
```

Usage in controller:

```php
<?php
// app/Http/Controllers/PostController.php
use App\Services\SupabaseService;

class PostController extends Controller
{
    public function __construct(private SupabaseService $supabase) {}

    public function index(): \Illuminate\Http\JsonResponse
    {
        $posts = $this->supabase->select('posts', 'id,title,created_at');
        return response()->json($posts);
    }

    public function store(Request $request): \Illuminate\Http\JsonResponse
    {
        $post = $this->supabase->insert('posts', $request->validated());
        return response()->json($post, 201);
    }
}
```

---

## Solution 3: Vue 3 / React Frontend (Inertia.js)

If using Inertia.js, all data fetching happens server-side in Laravel controllers. The frontend never makes direct API calls.

```php
<?php
// app/Http/Controllers/DashboardController.php
use Inertia\Inertia;

class DashboardController extends Controller
{
    public function index(SupabaseService $supabase): \Inertia\Response
    {
        // Server-side   no ISP blocking
        $posts = $supabase->select('posts');
        
        return Inertia::render('Dashboard', [
            'posts' => $posts,
        ]);
    }
}
```

```vue
<!-- resources/js/Pages/Dashboard.vue -->
<script setup>
defineProps({ posts: Array })
</script>

<template>
  <div v-for="post in posts" :key="post.id">{{ post.title }}</div>
</template>
```

No client-side API calls. No ISP bypass needed.

---

## Solution 4: Vue SPA (Separate from Laravel)

If your Vue/React app is a separate SPA that calls Laravel's API:

```javascript
// Vue src/lib/api.js
const API_BASE = import.meta.env.VITE_API_URL // Points to Laravel

export async function fetchPosts() {
  // Calls Laravel, which calls Supabase internally
  return fetch(`${API_BASE}/api/posts`).then(r => r.json())
}
```

Laravel URL should be your proxy-protected domain: `https://api.yourproduct.com` behind Cloudflare.

---

## CORS for Separate Vue/React Frontend

```php
<?php
// config/cors.php
return [
    'paths'               => ['api/*'],
    'allowed_methods'     => ['*'],
    'allowed_origins'     => [
        'https://yourproduct.com',
        'http://localhost:5173',
    ],
    'allowed_origins_patterns' => [],
    'allowed_headers'     => ['*'],
    'exposed_headers'     => [],
    'max_age'             => 0,
    'supports_credentials' => true,
];
```

---

## Laravel + Cloudflare Worker (Fallback)

If your Laravel server's IP gets blocked (rare), route through Cloudflare Worker to your Laravel server:

```javascript
// CF Worker
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    url.hostname = env.LARAVEL_ORIGIN; // e.g., your VPS IP or origin domain
    
    const newReq = new Request(url, request);
    newReq.headers.set('host', env.LARAVEL_ORIGIN);
    return fetch(newReq);
  }
}
```


