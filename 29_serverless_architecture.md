# Serverless Architecture

## Overview

Serverless platforms (AWS Lambda, Vercel Serverless Functions, Cloudflare Workers, Google Cloud Functions, Azure Functions) run your code in managed compute without persistent servers. They have implications for ISP bypass:

1. **Your function is IN a datacenter**   not behind an ISP. Outbound calls from your function to Supabase/Firebase are not blocked.
2. **The function URL might be blocked**   e.g., `*.lambda-url.us-east-1.amazonaws.com` could be blocked by ISPs.
3. **Always put a custom domain in front** of your serverless function.

---

## The Serverless ISP Bypass Stack

```
User (Jio) → api.yourproduct.com (Cloudflare) → Lambda Function URL
                                               OR Vercel Function
                                               OR Cloudflare Worker
```

Users never see the raw Lambda/Vercel URL. Your custom domain is the only public-facing entry point.

---

## AWS Lambda: Custom Domain Setup

### Step 1: Lambda Function URL

```python
# Lambda handler   calls Supabase server-side
import json
import os
import httpx

def handler(event, context):
    supabase_url = os.environ['SUPABASE_URL']
    supabase_key = os.environ['SUPABASE_KEY']
    
    # This call originates from AWS ap-south-1   no ISP blocking
    response = httpx.get(
        f"{supabase_url}/rest/v1/posts",
        headers={
            "apikey": supabase_key,
            "Authorization": f"Bearer {supabase_key}",
        },
        params={"select": "id,title,created_at"}
    )
    
    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": response.text,
    }
```

### Step 2: API Gateway Custom Domain

```yaml
# serverless.yml (Serverless Framework)
service: your-api

provider:
  name: aws
  runtime: python3.12
  region: ap-south-1  # Mumbai   lowest latency for India

plugins:
  - serverless-domain-manager

custom:
  customDomain:
    domainName: api.yourproduct.com
    stage: prod
    basePath: ''
    createRoute53Record: true

functions:
  api:
    handler: handler.handler
    events:
      - http:
          path: /{proxy+}
          method: ANY
    environment:
      SUPABASE_URL: ${env:SUPABASE_URL}
      SUPABASE_KEY: ${env:SUPABASE_KEY}
```

---

## Vercel Serverless Functions

Vercel functions run in a Node.js Lambda. Your Next.js API routes ARE Vercel serverless functions.

```typescript
// app/api/data/route.ts
import { createClient } from '@supabase/supabase-js'

// Server-side env vars (no NEXT_PUBLIC_ prefix)
const supabase = createClient(
  process.env.SUPABASE_URL!,        // Direct Supabase URL   from Vercel's servers
  process.env.SUPABASE_SERVICE_KEY! // Service key   never goes to browser
)

export async function GET() {
  const { data, error } = await supabase.from('posts').select('*').limit(20)
  if (error) return Response.json({ error: error.message }, { status: 500 })
  return Response.json(data)
}
```

The function runs in Vercel's datacenter (not on the user's device). Supabase is called server-side   zero ISP interference.

---

## Cloudflare Workers (Edge Serverless)

Cloudflare Workers are the recommended serverless platform for ISP bypass because:
- Deployed at 300+ edge locations globally
- Workers' IP ranges (`104.16.0.0/12`) are almost never blocked entirely
- Native WebSocket support
- Free tier is generous (100k requests/day)

```typescript
// cloudflare-worker.ts
interface Env {
  SUPABASE_URL: string
  SUPABASE_KEY: string
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url)
    
    if (url.pathname === '/api/posts') {
      const response = await fetch(`${env.SUPABASE_URL}/rest/v1/posts?select=*`, {
        headers: {
          apikey: env.SUPABASE_KEY,
          Authorization: `Bearer ${env.SUPABASE_KEY}`,
        },
      })
      
      return new Response(response.body, {
        status: response.status,
        headers: {
          'Content-Type': 'application/json',
          'Access-Control-Allow-Origin': '*',
        },
      })
    }
    
    return new Response('Not Found', { status: 404 })
  },
}
```

---

## Google Cloud Functions

```python
# main.py
import functions_framework
import httpx
import os

@functions_framework.http
def get_posts(request):
    """HTTP Cloud Function that proxies Supabase"""
    
    # Enable CORS
    if request.method == 'OPTIONS':
        headers = {
            'Access-Control-Allow-Origin': 'https://yourproduct.com',
            'Access-Control-Allow-Methods': 'GET',
            'Access-Control-Allow-Headers': 'Authorization',
            'Access-Control-Max-Age': '3600',
        }
        return ('', 204, headers)
    
    supabase_url = os.environ.get('SUPABASE_URL')
    supabase_key = os.environ.get('SUPABASE_KEY')
    
    with httpx.Client() as client:
        response = client.get(
            f"{supabase_url}/rest/v1/posts",
            headers={"apikey": supabase_key, "Authorization": f"Bearer {supabase_key}"}
        )
    
    headers = {'Access-Control-Allow-Origin': 'https://yourproduct.com'}
    return (response.text, response.status_code, headers)
```

Deploy to `asia-south1` (Mumbai):

```bash
gcloud functions deploy get-posts \
  --runtime python311 \
  --trigger-http \
  --region asia-south1 \
  --set-env-vars SUPABASE_URL=https://your-project.supabase.co,SUPABASE_KEY=key
```

---

## Azure Functions

```csharp
// PostsFunction.cs
[FunctionName("GetPosts")]
public static async Task<IActionResult> Run(
    [HttpTrigger(AuthorizationLevel.Anonymous, "get", Route = "posts")] HttpRequest req,
    ILogger log)
{
    var supabaseUrl = Environment.GetEnvironmentVariable("SUPABASE_URL");
    var supabaseKey = Environment.GetEnvironmentVariable("SUPABASE_KEY");
    
    using var httpClient = new HttpClient();
    httpClient.DefaultRequestHeaders.Add("apikey", supabaseKey);
    httpClient.DefaultRequestHeaders.Add("Authorization", $"Bearer {supabaseKey}");
    
    var response = await httpClient.GetStringAsync($"{supabaseUrl}/rest/v1/posts?select=*");
    
    return new OkObjectResult(response);
}
```

Deploy to `Central India` region for Indian users.

---

## Cold Start Mitigation

Serverless functions have cold starts (100-2000ms latency on first request). To minimize:

```javascript
// Cloudflare Workers: no cold start (V8 isolates, not Node)
// AWS Lambda: use provisioned concurrency
// Vercel: Edge Runtime (no cold start)

// Vercel Edge Runtime   fastest option
export const runtime = 'edge'

export async function GET() {
  // Runs in Cloudflare's V8 isolate   near-zero cold start
  const data = await fetch('https://api.yourproduct.com/...')
  return Response.json(await data.json())
}
```

---

## Serverless ISP Bypass Summary

| Platform | Default URL | Blocked? | Custom Domain | Best Region for India |
|:---------|:-----------|:---------|:-------------|:---------------------|
| Cloudflare Workers | `*.workers.dev` | Rarely | Yes (free) | Global edge |
| Vercel | `*.vercel.app` | Rarely | Yes (free) | `bom1` (Mumbai) |
| AWS Lambda | `*.lambda-url.aws.com` | Sometimes | Yes (API GW) | `ap-south-1` |
| GCP Functions | `*.cloudfunctions.net` | Sometimes | Yes (Cloud Run) | `asia-south1` |
| Azure Functions | `*.azurewebsites.net` | Rarely | Yes | `Central India` |

Always use a custom domain. Never ship the default `*.workers.dev` or `*.vercel.app` URLs to production.


