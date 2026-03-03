# Kubernetes and Microservices

## Overview

In Kubernetes/microservices architectures, the ISP bypass challenge operates at two levels:
1. **Ingress level**   User traffic entering the cluster must go through a CDN
2. **Service-to-service**   Internal services calling blocked external APIs (Supabase, Firebase, etc.)

---

## Architecture: Kubernetes ISP Bypass

```
Internet Traffic Flow (ISP Bypass):
User → Cloudflare → Ingress Controller → K8s Service → Pod

External API Access:
Pod → Internal Egress Proxy → External API (Supabase/Firebase)
```

---

## Solution 1: Cloudflare + Nginx Ingress Controller

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/rewrite-target: /
    # Tell Nginx to trust Cloudflare IPs for real IP
    nginx.ingress.kubernetes.io/use-forwarded-headers: "true"
    nginx.ingress.kubernetes.io/server-snippet: |
      real_ip_header CF-Connecting-IP;
spec:
  tls:
    - hosts:
        - api.yourproduct.com
      secretName: tls-secret
  rules:
    - host: api.yourproduct.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
```

All traffic reaches Kubernetes through Cloudflare. ISPs cannot block your cluster's IP directly.

---

## Solution 2: Supabase Proxy Sidecar

For services that need to call Supabase, deploy a proxy sidecar container in the same pod.

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  replicas: 3
  template:
    spec:
      containers:
        # Main application
        - name: app
          image: your-app:latest
          env:
            - name: SUPABASE_URL
              value: "http://localhost:3000"  # Proxy sidecar
            
        # Proxy sidecar for Supabase
        - name: supabase-proxy
          image: nginx:alpine
          ports:
            - containerPort: 3000
          volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/nginx.conf
              subPath: nginx.conf
      volumes:
        - name: nginx-config
          configMap:
            name: nginx-proxy-config
```

```yaml
# configmap.yaml   Nginx proxy config
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-proxy-config
data:
  nginx.conf: |
    events {}
    http {
      server {
        listen 3000;
        location / {
          proxy_pass https://your-project.supabase.co;
          proxy_set_header Host your-project.supabase.co;
          proxy_ssl_server_name on;
          proxy_http_version 1.1;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection "upgrade";
        }
      }
    }
```

---

## Solution 3: Kubernetes NetworkPolicy + Egress Control

Control which external endpoints pods can reach:

```yaml
# networkpolicy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-egress-policy
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Egress
  egress:
    # Allow DNS
    - ports:
        - protocol: UDP
          port: 53
    # Allow HTTPS to your own proxy only (block direct Supabase)
    - ports:
        - protocol: TCP
          port: 443
      to:
        - ipBlock:
            cidr: 104.16.0.0/12  # Cloudflare IPs only
```

This enforces that pods only reach external services through Cloudflare's IP range   not directly.

---

## Solution 4: Service Mesh (Istio) for External Traffic

Istio lets you define `ServiceEntry` for external services and route them through your proxy:

```yaml
# serviceentry.yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: supabase-via-proxy
spec:
  hosts:
    - api.yourproduct.com  # Your Cloudflare Worker proxy
  location: MESH_EXTERNAL
  ports:
    - number: 443
      name: https
      protocol: HTTPS
  resolution: DNS
```

```yaml
# virtualservice.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: supabase-routing
spec:
  hosts:
    - your-project.supabase.co  # The blocked target
  http:
    - route:
        - destination:
            host: api.yourproduct.com  # Redirect to proxy
          weight: 100
```

All Supabase calls in the cluster are transparently rerouted through your proxy.

---

## Solution 5: Helm Chart for Multi-Region Deployment

```yaml
# values.yaml
replicaCount: 3

image:
  repository: your-app
  tag: latest

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: api.yourproduct.com
      paths:
        - path: /
          pathType: Prefix

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

regions:
  - name: mumbai
    nodeSelector:
      topology.kubernetes.io/region: ap-south-1
    weight: 70
  - name: singapore
    nodeSelector:
      topology.kubernetes.io/region: ap-southeast-1
    weight: 30
```

---

## Database in Kubernetes

```yaml
# External secret for DB credentials (never hardcode)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials
    template:
      data:
        DATABASE_URL: "postgresql://{{ .username }}:{{ .password }}@{{ .host }}:5432/{{ .dbname }}"
  data:
    - secretKey: username
      remoteRef:
        key: production/db
        property: username
```

---

## Health Checks and Probes

```yaml
# deployment.yaml   proper health probes for ISP-resilient deployments
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 2

# The /ready endpoint should check that:
# 1. DB connection is alive
# 2. Proxy to external service is alive
# 3. Cache connection is alive
# If proxy is down → pod reports not ready → removed from load balancer
```


