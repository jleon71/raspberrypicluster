# Phase 5: Cloudflare Agent Configuration

## Overview

Set up Cloudflare Agent to provide DNS, DDoS protection, and edge connectivity for your Raspberry Pi cluster.

**Estimated Time:** 45 minutes - 1 hour

---

## What You'll Do

1. Create Cloudflare account and domain
2. Install Cloudflare Agent on cluster
3. Configure DNS
4. Set up SSL/TLS
5. Enable DDoS protection
6. Monitor traffic

---

## Prerequisites

- K3s cluster running
- Cloudflare account (free at https://www.cloudflare.com/plans/free/)
- A domain name (can be free or paid)

---

## Step 1: Cloudflare Account Setup

### 1.1 Create Account

1. Go to https://www.cloudflare.com
2. Sign up for free account
3. Verify email

### 1.2 Add Domain

1. In Cloudflare dashboard, click **Add domain**
2. Enter your domain (e.g., `rpi-cluster.com`)
3. Select free plan
4. Cloudflare will scan your DNS records
5. Update nameservers at your domain registrar

**Points to Cloudflare:**
```
nameserver1.cloudflare.com
nameserver2.cloudflare.com
```

Wait 24-48 hours for DNS propagation.

---

## Step 2: Install Cloudflare Agent on K3s

The Cloudflare Agent (Argo Tunnel) is deployed as a Kubernetes deployment.

### 2.1 Create Namespace

```bash
kubectl create namespace cloudflare
```

### 2.2 Create Cloudflare Tunnel

In Cloudflare Dashboard:

1. Go to **Access** → **Tunnels**
2. Click **Create a tunnel**
3. Select **Kubernetes**
4. Follow the guided setup
5. Note your tunnel token

### 2.3 Create Tunnel Secret

```bash
# Create secret with tunnel credentials
kubectl create secret generic cloudflare-tunnel \
  --from-literal=token=<YOUR_TUNNEL_TOKEN> \
  -n cloudflare
```

---

## Step 3: Deploy Cloudflare Agent

Create `k3s-manifests/cloudflare-agent.yml`:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflare-agent
  namespace: cloudflare
  labels:
    app: cloudflare-agent
spec:
  selector:
    matchLabels:
      app: cloudflare-agent
  replicas: 1
  template:
    metadata:
      labels:
        app: cloudflare-agent
    spec:
      serviceAccountName: cloudflare-sa
      containers:
      - name: cloudflared
        image: cloudflare/cloudflared:latest
        args:
          - tunnel
          - --token
          - $(TUNNEL_TOKEN)
          - run
        env:
        - name: TUNNEL_TOKEN
          valueFrom:
            secretKeyRef:
              name: cloudflare-tunnel
              key: token
        resources:
          limits:
            cpu: 100m
            memory: 128Mi
          requests:
            cpu: 50m
            memory: 64Mi
        livenessProbe:
          httpGet:
            path: /healthcheck
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 30
        securityContext:
          runAsNonRoot: true
          readOnlyRootFilesystem: true
          allowPrivilegeEscalation: false
          capabilities:
            drop:
              - ALL

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cloudflare-sa
  namespace: cloudflare

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cloudflare-agent
rules:
- apiGroups: [""]
  resources: ["nodes", "pods", "services"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cloudflare-agent
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cloudflare-agent
subjects:
- kind: ServiceAccount
  name: cloudflare-sa
  namespace: cloudflare

---
apiVersion: v1
kind: Service
metadata:
  name: cloudflare-agent
  namespace: cloudflare
spec:
  selector:
    app: cloudflare-agent
  ports:
  - port: 8000
    targetPort: 8000
  type: ClusterIP
```

Deploy:

```bash
kubectl apply -f k3s-manifests/cloudflare-agent.yml

# Check status
kubectl get all -n cloudflare

# View logs
kubectl logs -f deployment/cloudflare-agent -n cloudflare
```

---

## Step 4: Configure DNS Records

In Cloudflare Dashboard, add DNS records for your services:

### 4.1 CNAME Records

```
Type   | Name          | Content           | TTL    | Proxy
-------|---------------|-------------------|--------|-------
CNAME  | grafana       | example.com       | Auto   | Proxied
CNAME  | api           | example.com       | Auto   | Proxied
CNAME  | www           | example.com       | Auto   | Proxied
CNAME  | @             | example.com       | Auto   | Proxied
```

**Or A Records (if you have static IP):**

```
Type | Name    | IPv4        | TTL  | Proxy
-----|---------|-------------|------|--------
A    | @       | 203.0.113.1 | Auto | Proxied
A    | grafana | 203.0.113.1 | Auto | Proxied
A    | api     | 203.0.113.1 | Auto | Proxied
```

---

## Step 5: Configure Tunnel Ingress Routes

Create `k3s-manifests/cloudflare-routes.yml`:

```yaml
---
apiVersion: monitoring.coreos.com/v1
kind: ConfigMap
metadata:
  name: cloudflare-config
  namespace: cloudflare
data:
  config.yml: |
    tunnel: $(TUNNEL_ID)
    credentials-file: /etc/cloudflare/creds.json

    ingress:
      # Grafana
      - hostname: grafana.example.com
        service: http://grafana.monitoring.svc.cluster.local:80

      # API services
      - hostname: api.example.com
        service: http://hello-world.applications.svc.cluster.local:80

      # Default ingress
      - hostname: "*.example.com"
        service: http://nginx-ingress-ingress-nginx-controller.ingress-nginx.svc.cluster.local:80

      # Catch-all for health checks
      - service: http_status:404
```

---

## Step 6: Enable SSL/TLS

In Cloudflare Dashboard:

1. Go to **SSL/TLS** → **Overview**
2. Select **Flexible** (your cluster is not directly exposed)
   - This encrypts traffic from users to Cloudflare only
   - Or use **Full** if you can generate certificates
3. Enable **Always Use HTTPS**
4. Enable **Automatic HTTPS Rewrites**

### Alternative: Generate Self-Signed Certificates

For your internal cluster:

```bash
# On master node
ssh debian@192.168.1.100

# Generate certificate
sudo openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout /etc/rancher/k3s/server.key \
  -out /etc/rancher/k3s/server.crt \
  -subj "/CN=rpi-cluster.local"

# Create secret
kubectl create secret tls tls-secret \
  --cert=/etc/rancher/k3s/server.crt \
  --key=/etc/rancher/k3s/server.key \
  -n default

# Update ingress to use TLS
kubectl annotate ingress grafana \
  cert-manager.io/cluster-issuer=letsencrypt-prod \
  -n monitoring
```

---

## Step 7: Configure DDoS and Security

In Cloudflare Dashboard:

### 7.1 DDoS Protection

1. Go to **Security** → **DDoS**
2. Select **Medium** (recommended for free tier)
3. Monitor attacks in dashboard

### 7.2 WAF (Web Application Firewall)

1. Go to **Security** → **WAF**
2. Enable **Managed Rules**
3. Create custom rules:

```
(cf.threat_score > 50)
```

### 7.3 Rate Limiting

1. Go to **Security** → **Rate limiting**
2. Create rule:
   ```
   Rate limit: 10 requests per 10 seconds
   Action: Block
   URI: /api/*
   ```

### 7.4 Access Control

1. Go to **Access** → **App Launcher**
2. Create application:
   - App name: Grafana
   - Session duration: 24 hours
   - Require Cloudflare login

---

## Step 8: Monitor Traffic

In Cloudflare Dashboard:

1. Go to **Analytics** → **Overview**
   - View requests, bandwidth, threats
2. Go to **Analytics** → **Web Traffic**
   - View top URLs, status codes
3. Go to **Security** → **Events**
   - View blocked attacks, threats

---

## Step 9: Set Up Notifications

In Cloudflare Dashboard:

1. Go to **Notifications**
2. Create notification for:
   - High DDoS activity
   - SSL certificate expiring
   - Firewall blocks

---

## Step 10: Test Cloudflare Integration

```bash
# Test DNS resolution
nslookup grafana.example.com

# Should resolve to Cloudflare IP

# Check Cloudflare headers
curl -I https://grafana.example.com

# Should include:
# Server: cloudflare
# cf-ray: xxxxx
```

---

## Cloudflare Agent Configuration Details

### Full Configuration with Multiple Services

Create `cloudflare-config.yaml`:

```yaml
---
# Advanced Cloudflare Agent Config
tunnel: my-k3s-tunnel
credentials-file: /etc/cloudflare-creds.json
logLevel: info

# Ingress rules (order matters - first match wins)
ingress:
  # Admin services (protected)
  - hostname: admin.example.com
    path: "^/.*$"
    service: http://grafana.monitoring.svc.cluster.local:80
    originRequest:
      connectTimeout: 30s
      tlsTimeout: 30s
      tcpKeepAlive: 30s
      noHappyEyeballs: false

  # API services
  - hostname: api.example.com
    service: http://api.applications.svc.cluster.local:80

  # Web services with websocket support
  - hostname: app.example.com
    service: http://hello-world.applications.svc.cluster.local:80
    originRequest:
      disableChunkedEncoding: false

  # Subdomain wildcard
  - hostname: "*.app.example.com"
    service: http://nginx-ingress-ingress-nginx-controller.ingress-nginx.svc.cluster.local:80

  # Main domain
  - hostname: example.com
    service: http://nginx-ingress-ingress-nginx-controller.ingress-nginx.svc.cluster.local:80

  # 404 fallback
  - service: http_status:404

# Warp client configuration (optional)
warp-routing:
  enabled: false

# DNS query logging (optional)
dns: {}
```

---

## Troubleshooting

### Tunnel not connecting

```bash
# Check Cloudflare Agent logs
kubectl logs -f deployment/cloudflare-agent -n cloudflare

# Verify tunnel token
kubectl get secret cloudflare-tunnel -n cloudflare -o yaml

# Restart agent
kubectl rollout restart deployment/cloudflare-agent -n cloudflare
```

### DNS not resolving

```bash
# Check DNS propagation
nslookup example.com
dig example.com

# Check Cloudflare nameservers
ns example.com

# Wait 24-48 hours for full propagation
```

### Services unreachable through Cloudflare

```bash
# Test direct access (if exposed)
curl http://192.168.1.100/api

# Check ingress configuration
kubectl get ingress -A

# Verify tunnel routes
kubectl exec -it pod/cloudflare-agent-xxx -n cloudflare -- \
  cloudflared tunnel info
```

### SSL certificate issues

```bash
# Check certificate
curl -v https://example.com

# Verify Cloudflare SSL mode
# Dashboard → SSL/TLS → Overview

# If needed, generate new certificate
certbot certonly --manual -d example.com
```

---

## Performance Optimization

### Reduce Cloudflare Agent Load

```yaml
# In deployment
resources:
  limits:
    cpu: 50m  # From 100m
    memory: 64Mi  # From 128Mi
  requests:
    cpu: 25m
    memory: 32Mi
```

### Cache Configuration

In Cloudflare Dashboard:

1. **Caching** → **Cache Control**
   - Set Browser Cache TTL: 30 minutes
   - Set Cache Level: Aggressive

2. **Caching** → **Rules**
   ```
   If request URL contains: /api/
   Cache Level: Cache Everything
   Browser Cache TTL: 1 hour
   ```

---

## Security Best Practices

✅ **Enable:**
- Always Use HTTPS
- Automatic HTTPS Rewrites
- Brotli Compression
- Early Hints
- TLS 1.3
- Minimum TLS Version 1.2

✅ **Disable:**
- Universal SSL (if using Full)
- Auto Minify for custom apps

✅ **Configure:**
- Email obfuscation
- Server-side excludes
- Hotlink protection

---

## Cloudflare Free Tier Benefits

- ✅ Free DDoS protection
- ✅ Free DNS hosting
- ✅ Free SSL/TLS
- ✅ Free WAF (basic rules)
- ✅ Free caching
- ✅ Free analytics
- ✅ Free rate limiting (3 rules)
- ⚠️ Limited: Advanced WAF rules (upgrade to Pro)

---

## Monitoring Cloudflare

```bash
# Check Cloudflare Agent status
kubectl get pods -n cloudflare

# View tunnel traffic
# Dashboard → Traffic Analytics

# Check threats blocked
# Dashboard → Security → Events

# Monitor cache hit ratio
# Dashboard → Analytics → Caching
```

---

## Next Steps

1. **06_SCALING_GUIDE.md** - Add more Raspberry Pis
2. Deploy your applications to the cluster
3. Set up continuous deployment (optional)
4. Monitor and optimize performance

---

**Status:** Cloudflare Agent configured and cluster publicly accessible
**Next:** Scaling infrastructure for more nodes
