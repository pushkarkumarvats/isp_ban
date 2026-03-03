# Spring Boot + React / Vue

## Overview

Spring Boot is a Java framework that generates self-contained JARs with embedded Tomcat. It runs entirely server-side. The topology for ISP bypass is:

```
React/Vue (browser) → CDN edge → api.yourproduct.com → Nginx/LB → Spring Boot → DB (private)
```

Spring Boot's database connections (JDBC/JPA) are not affected by browser-side DNS poisoning. The concern is your Spring Boot server's public IP being reachable   which is solved by CDN proxying.

---

## Solution 1: Nginx + Cloudflare in Front of Spring Boot

Put your Spring Boot API behind Nginx, which is behind Cloudflare as CDN.

```nginx
# /etc/nginx/sites-available/springboot
upstream springboot {
    server 127.0.0.1:8080;
    keepalive 32;
}

server {
    listen 443 ssl http2;
    server_name api.yourproduct.com;

    ssl_certificate /etc/letsencrypt/live/api.yourproduct.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.yourproduct.com/privkey.pem;

    # Trust Cloudflare's IPs for real IP restoration
    set_real_ip_from 103.21.244.0/22;
    set_real_ip_from 103.22.200.0/22;
    set_real_ip_from 104.16.0.0/13;
    set_real_ip_from 104.24.0.0/14;
    set_real_ip_from 108.162.192.0/18;
    real_ip_header CF-Connecting-IP;

    location / {
        proxy_pass http://springboot;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

---

## Solution 2: Spring Boot as Proxy for Blocked Services

If React/Vue frontend needs to call Supabase or another blocked service, add a controller in Spring Boot that proxies those calls server-side.

```java
// SupabaseProxyController.java
package com.yourcompany.proxy;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;

import jakarta.servlet.http.HttpServletRequest;
import java.util.Collections;
import java.util.Enumeration;

@RestController
@RequestMapping("/api/supabase")
public class SupabaseProxyController {

    private final WebClient webClient;

    @Value("${supabase.url}")
    private String supabaseUrl;

    @Value("${supabase.anon-key}")
    private String supabaseAnonKey;

    public SupabaseProxyController(WebClient.Builder builder) {
        this.webClient = builder.build();
    }

    @RequestMapping(value = "/{path}", method = {
        RequestMethod.GET, RequestMethod.POST, RequestMethod.PUT,
        RequestMethod.PATCH, RequestMethod.DELETE
    })
    public Mono<ResponseEntity<byte[]>> proxy(
        @PathVariable String path,
        @RequestParam(required = false) String query,
        HttpServletRequest request,
        @RequestBody(required = false) byte[] body
    ) {
        String targetUrl = supabaseUrl + "/" + path;
        if (request.getQueryString() != null) {
            targetUrl += "?" + request.getQueryString();
        }

        HttpMethod method = HttpMethod.valueOf(request.getMethod());
        
        // Build headers
        HttpHeaders headers = new HttpHeaders();
        Enumeration<String> headerNames = request.getHeaderNames();
        while (headerNames.hasMoreElements()) {
            String name = headerNames.nextElement();
            if (!name.equalsIgnoreCase("host")) {
                headers.put(name, Collections.list(request.getHeaders(name)));
            }
        }
        headers.set("apikey", supabaseAnonKey);
        headers.set("host", supabaseUrl.replace("https://", ""));

        return webClient.method(method)
            .uri(targetUrl)
            .headers(h -> h.addAll(headers))
            .bodyValue(body != null ? body : new byte[0])
            .retrieve()
            .toEntity(byte[].class);
    }
}
```

### application.properties

```properties
supabase.url=https://your-project.supabase.co
supabase.anon-key=your-anon-key
corsAllowedOrigins=https://yourproduct.com,http://localhost:3000
```

### CORS Config

```java
// WebConfig.java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Value("${corsAllowedOrigins}")
    private String[] allowedOrigins;

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
            .allowedOrigins(allowedOrigins)
            .allowedMethods("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS")
            .allowedHeaders("*")
            .allowCredentials(true);
    }
}
```

---

## Solution 3: React/Vue with Spring Boot API (No External Services)

Best practice: React/Vue makes calls only to your Spring Boot API. Spring Boot uses JPA/Hibernate for all database operations. No external blocked services contacted from the browser.

```javascript
// React API client
const API_BASE = process.env.REACT_APP_API_URL  // https://api.yourproduct.com

const api = {
  async get(path) {
    return fetch(`${API_BASE}${path}`, {
      credentials: 'include',
      headers: { 'Content-Type': 'application/json' }
    }).then(r => r.json())
  }
}

// Usage
const users = await api.get('/api/users')
```

```java
// UserService.java   all DB through JPA, not affected by ISP
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;

    public List<User> getAllUsers() {
        return userRepository.findAll();
    }
}
```

---

## Spring Boot + PostgreSQL (Supabase DB Direct)

Spring Boot can connect directly to Supabase's PostgreSQL:

```properties
# application.properties
spring.datasource.url=jdbc:postgresql://db.your-project.supabase.co:5432/postgres
spring.datasource.username=postgres
spring.datasource.password=${SUPABASE_DB_PASSWORD}
spring.datasource.driver-class-name=org.postgresql.Driver

# Use connection pooler (port 6543) if 5432 is blocked
# spring.datasource.url=jdbc:postgresql://db.your-project.supabase.co:6543/postgres?pgbouncer=true

spring.jpa.hibernate.ddl-auto=none
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
```

---

## Deployment on AWS (Multi-Region)

```yaml
# AWS ECS Task Definition
{
  "family": "spring-api",
  "networkMode": "awsvpc",
  "containerDefinitions": [{
    "name": "spring-boot",
    "image": "your-ecr-repo:latest",
    "portMappings": [{ "containerPort": 8080 }],
    "environment": [
      { "name": "SUPABASE_URL", "value": "https://your-project.supabase.co" },
      { "name": "SPRING_PROFILES_ACTIVE", "value": "prod" }
    ],
    "secrets": [
      { "name": "SUPABASE_DB_PASSWORD", "valueFrom": "arn:aws:secretsmanager:ap-south-1:..." }
    ]
  }],
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512"
}
```

Deploy to `ap-south-1` (Mumbai) for lowest latency to Indian users.


