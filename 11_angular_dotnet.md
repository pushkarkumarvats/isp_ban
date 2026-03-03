# Angular + .NET Core

## Overview

Angular is a full TypeScript framework typically used with .NET Core (C#) or .NET 8 Web API backends. The Angular app runs in the browser (affected by ISP DNS poisoning). The .NET backend runs on a server (not affected).

**Strategy**: Never call blocked services directly from Angular. Route all calls through your .NET backend or a Cloudflare Worker proxy.

---

## Solution 1: Angular HttpClient Interceptor (Cleanest Pattern)

Create an Angular HTTP interceptor that rewrites URLs for blocked services.

```typescript
// src/app/interceptors/proxy.interceptor.ts
import { Injectable } from '@angular/core'
import {
  HttpInterceptor, HttpRequest, HttpHandler, HttpEvent
} from '@angular/common/http'
import { Observable } from 'rxjs'
import { environment } from '../../environments/environment'

@Injectable()
export class ProxyInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    // Rewrite direct Supabase calls to proxy
    if (req.url.includes('supabase.co') && environment.supabaseProxyUrl) {
      const proxyUrl = req.url.replace(
        /https:\/\/[a-z0-9]+\.supabase\.co/,
        environment.supabaseProxyUrl
      )
      const proxiedReq = req.clone({ url: proxyUrl })
      return next.handle(proxiedReq)
    }
    return next.handle(req)
  }
}
```

Register the interceptor:

```typescript
// src/app/app.module.ts
import { HTTP_INTERCEPTORS, HttpClientModule } from '@angular/common/http'
import { ProxyInterceptor } from './interceptors/proxy.interceptor'

@NgModule({
  imports: [HttpClientModule],
  providers: [
    {
      provide: HTTP_INTERCEPTORS,
      useClass: ProxyInterceptor,
      multi: true,
    },
  ],
})
export class AppModule {}
```

### Environment Configuration

```typescript
// src/environments/environment.ts (development)
export const environment = {
  production: false,
  supabaseUrl: 'https://your-project.supabase.co',
  supabaseAnonKey: 'your-anon-key',
  supabaseProxyUrl: '',  // empty = use direct URL in dev
}

// src/environments/environment.prod.ts (production)
export const environment = {
  production: true,
  supabaseUrl: 'https://your-project.supabase.co',
  supabaseAnonKey: 'your-anon-key',
  supabaseProxyUrl: 'https://your-worker.workers.dev',  // CF Worker
}
```

---

## Solution 2: .NET Core as API Proxy

Your .NET backend proxies requests server-side. Angular only calls your .NET API.

### .NET 8 Minimal API Proxy

```csharp
// Program.cs
using System.Net.Http.Headers;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddHttpClient("supabase", client => {
    client.BaseAddress = new Uri(builder.Configuration["Supabase:Url"]!);
    client.DefaultRequestHeaders.Add("apikey", builder.Configuration["Supabase:AnonKey"]);
});

builder.Services.AddCors(options => {
    options.AddPolicy("Angular", policy => {
        policy.WithOrigins("https://yourproduct.com", "http://localhost:4200")
              .AllowAnyMethod()
              .AllowAnyHeader()
              .AllowCredentials();
    });
});

var app = builder.Build();
app.UseCors("Angular");

// Proxy endpoint for Supabase REST
app.MapGet("/api/supabase/{**path}", async (string path, HttpContext ctx, IHttpClientFactory factory) => {
    var client = factory.CreateClient("supabase");
    var targetPath = $"/rest/v1/{path}{ctx.Request.QueryString}";
    
    // Forward auth header if present
    if (ctx.Request.Headers.TryGetValue("Authorization", out var auth)) {
        client.DefaultRequestHeaders.Authorization = 
            AuthenticationHeaderValue.Parse(auth!);
    }
    
    var response = await client.GetAsync(targetPath);
    var content = await response.Content.ReadAsStringAsync();
    return Results.Content(content, "application/json", statusCode: (int)response.StatusCode);
});

app.Run();
```

### appsettings.json

```json
{
  "Supabase": {
    "Url": "https://your-project.supabase.co",
    "AnonKey": "your-anon-key"
  },
  "AllowedHosts": "*"
}
```

### Angular Service

```typescript
// src/app/services/data.service.ts
import { Injectable } from '@angular/core'
import { HttpClient } from '@angular/common/http'
import { Observable } from 'rxjs'
import { environment } from '../../environments/environment'

@Injectable({ providedIn: 'root' })
export class DataService {
  private readonly apiBase = environment.apiUrl  // .NET backend URL

  constructor(private http: HttpClient) {}

  getUsers(): Observable<any[]> {
    // Goes through .NET, which calls Supabase server-side
    return this.http.get<any[]>(`${this.apiBase}/api/supabase/users`)
  }
}
```

---

## Solution 3: Angular Universal (SSR) + .NET Reverse Proxy

Angular Universal runs server-side rendering on a Node.js process. For non-Angular pages, .NET acts as a reverse proxy.

```csharp
// .NET 8 YARP Reverse Proxy setup
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));
```

```json
// appsettings.json
{
  "ReverseProxy": {
    "Routes": {
      "supabase-route": {
        "ClusterId": "supabase-cluster",
        "Match": { "Path": "/supabase/{**catch-all}" },
        "Transforms": [{ "PathPattern": "{**catch-all}" }]
      }
    },
    "Clusters": {
      "supabase-cluster": {
        "Destinations": {
          "destination1": {
            "Address": "https://your-project.supabase.co/"
          }
        }
      }
    }
  }
}
```

---

## Supabase JS Client with Angular

```typescript
// src/app/services/supabase.service.ts
import { Injectable } from '@angular/core'
import { createClient, SupabaseClient } from '@supabase/supabase-js'
import { environment } from '../../environments/environment'

@Injectable({ providedIn: 'root' })
export class SupabaseService {
  private readonly supabase: SupabaseClient

  constructor() {
    const url = environment.supabaseProxyUrl || environment.supabaseUrl
    this.supabase = createClient(url, environment.supabaseAnonKey)
  }

  get client() { return this.supabase }

  async signIn(email: string, password: string) {
    return this.supabase.auth.signInWithPassword({ email, password })
  }

  async getUser() {
    return this.supabase.auth.getUser()
  }
}
```

---

## Realtime in Angular (via CF Worker)

```typescript
// src/app/services/realtime.service.ts
import { Injectable, OnDestroy } from '@angular/core'
import { Subject } from 'rxjs'
import { SupabaseService } from './supabase.service'

@Injectable({ providedIn: 'root' })
export class RealtimeService implements OnDestroy {
  private channel: any
  public changes$ = new Subject<any>()

  constructor(private supabaseService: SupabaseService) {}

  subscribeToTable(table: string) {
    this.channel = this.supabaseService.client
      .channel(`${table}-realtime`)
      .on('postgres_changes', { event: '*', schema: 'public', table }, 
          (payload) => this.changes$.next(payload))
      .subscribe()
  }

  ngOnDestroy() {
    if (this.channel) {
      this.supabaseService.client.removeChannel(this.channel)
    }
  }
}
```

