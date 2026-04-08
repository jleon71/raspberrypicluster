# Phase 4: Monitoring with Prometheus and Grafana (Rocky Linux)

## Overview

Set up comprehensive monitoring for your Rocky Linux K3s cluster using Prometheus (metrics collection) and Grafana (visualization).

**Estimated Time:** 1-2 hours

**Note:** This guide works identically for Rocky Linux as Prometheus/Grafana are platform-agnostic

---

## What You'll Do

1. Install Prometheus for metrics collection
2. Install Grafana for visualization
3. Configure data sources
4. Create dashboards
5. Set up alerting rules
6. Enable monitoring for applications

---

## Prerequisites

- K3s cluster running (from Phase 3)
- kubectl access configured
- Helm installed on control machine

---

## Step 1: Install Helm

If you don't have Helm:

```bash
# Download and install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
helm version
```

---

## Step 2: Add Prometheus Helm Repository

```bash
# Add Prometheus community charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Verify
helm repo list
```

---

## Step 3: Create Prometheus Values File

Create `helm-values/prometheus-values.yml`:

```yaml
# Prometheus Helm Values - Optimized for Raspberry Pi
prometheus:
  prometheusSpec:
    # Resource limits for Raspberry Pi (limited memory)
    resources:
      limits:
        cpu: 500m
        memory: 512Mi
      requests:
        cpu: 100m
        memory: 256Mi

    # Retention period
    retention: 15d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: local-path
          accessModes: [ "ReadWriteOnce" ]
          resources:
            requests:
              storage: 5Gi

    # Service monitor selectors
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false

    # Scrape interval
    scrapeInterval: 30s
    evaluationInterval: 30s

# Grafana Prometheus datasource
grafana:
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
      - name: Prometheus
        type: prometheus
        url: http://prometheus-operated:9090
        access: proxy
        isDefault: true

# AlertManager (simple setup)
alertmanager:
  enabled: true
  alertmanagerSpec:
    resources:
      limits:
        cpu: 100m
        memory: 128Mi
      requests:
        cpu: 50m
        memory: 64Mi

# Node Exporter (for hardware metrics)
nodeExporter:
  enabled: true

# Prometheus Operator
prometheusOperator:
  resources:
    limits:
      cpu: 100m
      memory: 128Mi
    requests:
      cpu: 50m
      memory: 64Mi
```

---

## Step 4: Install Prometheus Stack

```bash
# Create monitoring namespace
kubectl create namespace monitoring

# Install using Helm
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values helm-values/prometheus-values.yml

# Wait for pods to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=prometheus -n monitoring --timeout=300s

# Check status
kubectl get all -n monitoring
```

---

## Step 5: Install Grafana

The Prometheus stack includes Grafana, but we'll configure it separately for more control.

Create `helm-values/grafana-values.yml`:

```yaml
# Grafana Helm Values - Optimized for Raspberry Pi
image:
  repository: grafana/grafana
  tag: latest

# Replicas
replicas: 1

# Resources for Raspberry Pi
resources:
  limits:
    cpu: 250m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

# Storage
persistence:
  enabled: true
  size: 1Gi
  storageClassName: local-path

# Grafana admin credentials
adminPassword: admin  # CHANGE THIS!

# Data sources
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-operated.monitoring.svc:9090
      access: proxy
      isDefault: true

# Pre-installed dashboards
dashboardProviders:
  dashboardproviders.yaml:
    apiVersion: 1
    providers:
    - name: 'default'
      orgId: 1
      folder: ''
      type: file
      disableDeletion: false
      updateIntervalSeconds: 10
      allowUiUpdates: true
      options:
        path: /var/lib/grafana/dashboards/default

dashboards:
  default:
    k3s:
      url: https://grafana.com/api/dashboards/13981/revisions/1/download
    kubernetes-cluster:
      url: https://grafana.com/api/dashboards/7249/revisions/1/download
    node-exporter:
      url: https://grafana.com/api/dashboards/1860/revisions/23/download
```

Install Grafana:

```bash
helm install grafana grafana/grafana \
  --namespace monitoring \
  --values helm-values/grafana-values.yml

# Check status
kubectl get all -n monitoring | grep grafana
```

---

## Step 6: Access Grafana

### 6.1 Port Forward to Local Machine

```bash
# Forward Grafana port
kubectl port-forward svc/grafana 3000:80 -n monitoring

# Open in browser: http://localhost:3000
# Default credentials: admin / admin (from values file)
```

### 6.2 Create Ingress for Grafana

Create `k3s-manifests/grafana-ingress.yml`:

```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  namespace: monitoring
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  ingressClassName: nginx
  rules:
  - host: grafana.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: grafana
            port:
              number: 80
```

Deploy:

```bash
kubectl apply -f k3s-manifests/grafana-ingress.yml

# Access via: http://grafana.local
```

---

## Step 7: Configure Datasources in Grafana

1. Log in to Grafana (http://localhost:3000)
2. Go to **Configuration** → **Data Sources**
3. Click **Add data source**
4. Select **Prometheus**
5. Set URL to: `http://prometheus-operated:9090`
6. Click **Save & Test**

---

## Step 8: Import Pre-built Dashboards

In Grafana, go to **+ → Import Dashboard** and use these IDs:

- **1860** - Node Exporter (hardware metrics)
- **7249** - Kubernetes Cluster Monitoring
- **13981** - K3s Exporter
- **3662** - Prometheus Stats

Or import from URL: https://grafana.com/grafana/dashboards

---

## Step 9: Create Custom Dashboard

### 9.1 Create Cluster Overview Dashboard

1. In Grafana, click **+ → Create → Dashboard**
2. Click **Add new panel**
3. Use these queries:

**Panel 1: Node Status**
```promql
sum(kube_node_status_condition{condition="Ready", status="true"})
```

**Panel 2: Pod Count**
```promql
sum(kube_pod_info)
```

**Panel 3: Memory Usage**
```promql
sum(node_memory_MemTotal_bytes) - sum(node_memory_MemAvailable_bytes)
```

**Panel 4: CPU Usage**
```promql
sum(rate(node_cpu_seconds_total[5m])) by (instance)
```

**Panel 5: Network IO**
```promql
rate(node_network_receive_bytes_total[5m])
```

---

## Step 10: Set Up Alert Rules

Create `k3s-manifests/prometheus-alerts.yml`:

```yaml
---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: k3s-alerts
  namespace: monitoring
  labels:
    prometheus: kube-prometheus
spec:
  groups:
  - name: k3s.rules
    interval: 30s
    rules:
    # Node Down
    - alert: NodeDown
      expr: kube_node_status_condition{condition="Ready",status="true"} == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Node {{ $labels.node }} is down"
        description: "Node has been unready for 5 minutes"

    # High Memory Usage
    - alert: HighMemoryUsage
      expr: (1 - (sum(node_memory_MemAvailable_bytes) / sum(node_memory_MemTotal_bytes))) > 0.85
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High memory usage on {{ $labels.instance }}"
        description: "Memory usage is above 85%"

    # High CPU Usage
    - alert: HighCPUUsage
      expr: sum(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) < 0.3
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High CPU usage on {{ $labels.instance }}"
        description: "CPU usage is above 70%"

    # Pod Crash Loop
    - alert: PodCrashLooping
      expr: rate(kube_pod_container_status_restarts_total[30m]) > 0
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} in namespace {{ $labels.namespace }} is crash looping"

    # PVC Almost Full
    - alert: PVCAlmostFull
      expr: kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes > 0.8
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "PVC {{ $labels.persistentvolumeclaim }} is 80% full"
```

Deploy alerts:

```bash
kubectl apply -f k3s-manifests/prometheus-alerts.yml

# Check alerts
kubectl get prometheusrules -n monitoring
```

---

## Step 11: Configure Application Monitoring

### 11.1 Monitor Your Applications

Create `k3s-manifests/app-monitoring.yml`:

```yaml
---
# Service Monitor for Prometheus
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: application-monitor
  namespace: applications
spec:
  selector:
    matchLabels:
      monitored: "true"
  endpoints:
  - port: metrics
    interval: 30s

---
# Prometheus Rules for Applications
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: application-alerts
  namespace: applications
spec:
  groups:
  - name: applications
    interval: 30s
    rules:
    - alert: ApplicationDown
      expr: up{job="applications"} == 0
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "Application {{ $labels.instance }} is down"
```

Deploy:

```bash
kubectl apply -f k3s-manifests/app-monitoring.yml
```

---

## Step 12: Enable Metrics for K3s

K3s exposes metrics by default. Enable collection:

Create `k3s-manifests/k3s-servicemonitor.yml`:

```yaml
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: k3s-metrics
  namespace: monitoring
spec:
  selector:
    matchLabels:
      k8s-app: kubelet
  endpoints:
  - port: https-metrics
    interval: 30s
    scheme: https
    tlsConfig:
      insecureSkipVerify: true
```

Deploy:

```bash
kubectl apply -f k3s-manifests/k3s-servicemonitor.yml
```

---

## Step 13: View Prometheus Targets

```bash
# Port forward to Prometheus
kubectl port-forward svc/prometheus-operated 9090:9090 -n monitoring

# Open: http://localhost:9090/targets
# Should show all scraped targets (nodes, pods, services)
```

---

## Step 14: Dashboard Best Practices

### Recommended Dashboards

1. **Infrastructure Dashboard**
   - Node status
   - Memory/CPU per node
   - Disk usage
   - Network stats

2. **Application Dashboard**
   - Pod status
   - Request rates
   - Error rates
   - Response times

3. **Cluster Health Dashboard**
   - Node count
   - Pod count
   - API server health
   - Scheduler health

---

## Monitoring Commands

```bash
# Check Prometheus scrape configs
kubectl port-forward svc/prometheus-operated 9090:9090 -n monitoring
# Open http://localhost:9090/config

# Query Prometheus
curl 'http://localhost:9090/api/v1/query?query=up'

# Check alert manager
kubectl port-forward svc/prometheus-kube-prom-alertmanager 9093:9093 -n monitoring
# Open http://localhost:9093

# View Grafana logs
kubectl logs -f deployment/grafana -n monitoring
```

---

## Backup Grafana Configuration

```bash
# Backup Grafana dashboards
kubectl exec -it deployment/grafana -n monitoring -- \
  grafana-cli admin export-dashboard

# Or backup entire volume
kubectl exec -it deployment/grafana -n monitoring -- \
  tar -czf /tmp/grafana-backup.tar.gz /var/lib/grafana

kubectl cp monitoring/grafana-xxx:/tmp/grafana-backup.tar.gz ./grafana-backup.tar.gz
```

---

## Troubleshooting

### Prometheus not scraping targets
```bash
# Check targets in Prometheus UI
# http://localhost:9090/targets

# Check ServiceMonitor creation
kubectl get servicemonitors -A

# Check Prometheus pod logs
kubectl logs -f prometheus-0 -n monitoring -c prometheus
```

### Grafana not showing metrics
```bash
# Check datasource configuration
# Go to Grafana → Configuration → Data Sources

# Test datasource
curl http://prometheus-operated.monitoring.svc:9090/api/v1/query?query=up

# Check Grafana logs
kubectl logs -f deployment/grafana -n monitoring
```

### High memory usage
```bash
# Reduce retention period
kubectl edit prometheus prometheus -n monitoring

# Change retention: to 7d or 5d

# Scale down replicas
kubectl scale deployment grafana --replicas=0 -n monitoring
```

---

## Performance Optimization

For limited Raspberry Pi resources:

```yaml
# Reduce scrape frequency (in values file)
scrapeInterval: 60s  # From 30s to 60s
evaluationInterval: 60s

# Reduce storage
retention: 7d  # From 15d to 7d

# Disable metrics
metrics.enabled: false
```

---

## Next Steps

1. **05_CLOUDFLARE_AGENT.md** - Configure networking
2. Create custom dashboards for your applications
3. Set up notification channels (Slack, email, etc.)

---

**Status:** Prometheus and Grafana fully configured and monitoring cluster
**Next:** Cloudflare Agent setup
