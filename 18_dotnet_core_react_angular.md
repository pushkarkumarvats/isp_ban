# .NET Core + React / Angular

## Overview

.NET Core (ASP.NET Core) is Microsoft's cross-platform web framework. It runs on Windows (Azure, IIS) or Linux (Docker, VMs). Like all server-side frameworks, its database connections are not affected by browser-side DNS poisoning.

**Unique advantage**: .NET has YARP (Yet Another Reverse Proxy)   a production-grade reverse proxy library that integrates natively into ASP.NET Core.

---

## Solution 1: ASP.NET Core with YARP Reverse Proxy

YARP lets you add a full reverse proxy to any ASP.NET Core app in <50 lines of code.

### Install

```bash
dotnet add package Yarp.ReverseProxy
```

### Program.cs

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add YARP
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

// CORS for React/Angular frontend
builder.Services.AddCors(opts => opts.AddDefaultPolicy(policy =>
    policy.WithOrigins(
            builder.Configuration.GetSection("AllowedOrigins").Get<string[]>()
                ?? new[] { "http://localhost:3000", "http://localhost:4200" }
          )
          .AllowAnyMethod()
          .AllowAnyHeader()
          .AllowCredentials()
));

var app = builder.Build();
app.UseCors();
app.MapReverseProxy();
app.Run();
```

### appsettings.json

```json
{
  "AllowedOrigins": [
    "https://yourproduct.com",
    "http://localhost:3000",
    "http://localhost:4200"
  ],
  "ReverseProxy": {
    "Routes": {
      "supabase-route": {
        "ClusterId": "supabase",
        "Match": { "Path": "/supabase/{**catch-all}" },
        "Transforms": [
          { "PathPattern": "{**catch-all}" },
          { "RequestHeader": "host", "Set": "your-project.supabase.co" },
          { "RequestHeader": "apikey", "Set": "{SUPABASE_ANON_KEY}" }
        ]
      }
    },
    "Clusters": {
      "supabase": {
        "HttpClient": { "SslProtocols": "Tls12, Tls13" },
        "Destinations": {
          "supabase-1": {
            "Address": "https://your-project.supabase.co/"
          }
        }
      }
    }
  }
}
```

---

## Solution 2: HttpClient Proxy Controller

For finer control over what gets proxied:

```csharp
// Controllers/ProxyController.cs
using System.Net.Http.Headers;

[ApiController]
[Route("api/proxy")]
public class ProxyController : ControllerBase
{
    private readonly HttpClient _httpClient;
    private readonly IConfiguration _config;

    public ProxyController(IHttpClientFactory factory, IConfiguration config)
    {
        _httpClient = factory.CreateClient("supabase");
        _config = config;
    }

    [HttpGet("{**path}")]
    [HttpPost("{**path}")]
    [HttpPut("{**path}")]
    [HttpPatch("{**path}")]
    [HttpDelete("{**path}")]
    public async Task<IActionResult> Proxy(string path)
    {
        var supabaseUrl = _config["Supabase:Url"];
        var anonKey = _config["Supabase:AnonKey"];

        var targetUrl = $"{supabaseUrl}/{path}";
        if (Request.QueryString.HasValue)
            targetUrl += Request.QueryString.Value;

        var requestMessage = new HttpRequestMessage(
            new HttpMethod(Request.Method),
            targetUrl
        );

        // Forward headers
        foreach (var (key, value) in Request.Headers)
        {
            if (!new[] { "host", "content-length" }.Contains(key.ToLower()))
                requestMessage.Headers.TryAddWithoutValidation(key, value.ToArray());
        }

        requestMessage.Headers.TryAddWithoutValidation("apikey", anonKey);
        requestMessage.Headers.Host = new Uri(supabaseUrl!).Host;

        // Forward body for non-GET
        if (Request.ContentLength > 0)
        {
            requestMessage.Content = new StreamContent(Request.Body);
            requestMessage.Content.Headers.ContentType =
                MediaTypeHeaderValue.Parse(Request.ContentType ?? "application/json");
        }

        var response = await _httpClient.SendAsync(requestMessage);
        var content = await response.Content.ReadAsStringAsync();

        return StatusCode((int)response.StatusCode, content);
    }
}
```

Register HttpClient:

```csharp
// Program.cs
builder.Services.AddHttpClient("supabase", client =>
{
    client.BaseAddress = new Uri(builder.Configuration["Supabase:Url"]!);
    client.Timeout = TimeSpan.FromSeconds(30);
});
```

---

## Solution 3: React SPA with .NET Backend for Identity

Using Microsoft Identity / Azure AD B2C through .NET   the auth flow never hits a blocked domain from the browser:

```csharp
// Program.cs
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = builder.Configuration["Auth:Authority"];
        options.Audience = builder.Configuration["Auth:Audience"];
    });
```

```javascript
// React: MSAL configuration (points to your .NET auth endpoint instead of tenant)
const msalConfig = {
  auth: {
    clientId: process.env.REACT_APP_CLIENT_ID,
    authority: process.env.REACT_APP_AUTH_URL, // Your proxy, not Azure directly
  }
}
```

---

## Solution 4: Azure Deployment + Traffic Manager

Azure Traffic Manager provides DNS-level failover between regions:

```json
{
  "type": "Microsoft.Network/trafficManagerProfiles",
  "properties": {
    "profileStatus": "Enabled",
    "trafficRoutingMethod": "Performance",
    "endpoints": [
      {
        "name": "centralIndia",
        "type": "AzureEndpoints",
        "properties": {
          "targetResourceId": "/subscriptions/.../loadBalancers/api-lb-india",
          "endpointStatus": "Enabled",
          "priority": 1
        }
      },
      {
        "name": "southeastAsia",
        "type": "AzureEndpoints",
        "properties": {
          "targetResourceId": "/subscriptions/.../loadBalancers/api-lb-sea",
          "endpointStatus": "Enabled",
          "priority": 2
        }
      }
    ],
    "monitorConfig": {
      "protocol": "HTTPS",
      "port": 443,
      "path": "/health",
      "intervalInSeconds": 30,
      "toleratedNumberOfFailures": 2
    }
  }
}
```

---

## Angular with .NET Core (Full Stack)

```typescript
// Angular: environment.ts
export const environment = {
  production: true,
  apiUrl: 'https://api.yourproduct.com',  // .NET API behind Cloudflare
  // No direct calls to Supabase, Firebase, etc.
}
```

```typescript
// Angular: api.service.ts
@Injectable({ providedIn: 'root' })
export class ApiService {
  private base = environment.apiUrl

  constructor(private http: HttpClient) {}

  // All calls go to .NET   no blocked domains reached from browser
  getUsers() { return this.http.get(`${this.base}/api/users`) }
  createUser(user: any) { return this.http.post(`${this.base}/api/users`, user) }
}
```

---

## Minimal API Health Check

Always expose a health check for CDN/load balancer monitoring:

```csharp
// Program.cs
app.MapGet("/health", () => Results.Ok(new {
    status = "healthy",
    timestamp = DateTime.UtcNow,
    version = "1.0.0"
})).AllowAnonymous();
```


