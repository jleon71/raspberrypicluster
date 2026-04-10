# Phase 3: K3s Cluster Advanced Configuration (Rocky Linux)

## Overview

K3s (Kubernetes) is now installed on your Rocky Linux cluster. This guide covers advanced configuration, networking, storage, and application deployment.

**Estimated Time:** 1-2 hours

**Note:** Firewall rules use firewalld (Rocky Linux default)

---

## What You'll Do

1. Configure cluster networking
2. Set up persistent storage
3. Configure ingress controller
4. Create namespaces
5. Deploy test applications
6. Enable metrics server (for HPA and monitoring)

---

## Prerequisites

- K3s cluster running (from Phase 2)
- `kubectl` access configured
- At least one worker node

---

## Step 1: Verify Cluster Health

From your control machine:

```bash
# Set kubeconfig
export KUBECONFIG=~/.kube/config-rpi

# Check nodes
kubectl get nodes -o wide

# Check system pods
kubectl get pods -A

# Check cluster info
kubectl cluster-info
kubectl version --short
```

---

## Step 2: Configure K3s CNI (Container Network Interface)

K3s comes with Flannel by default, which is good for Raspberry Pi. Verify:

```bash
# Check CNI plugin
kubectl get daemonset -n kube-system -l k8s-app=flannel

# Check network policies
kubectl get networkpolicies -A
```

To see network configuration:

```bash
# Check Flannel config
kubectl get cm kube-flannel-cfg -n kube-system -o yaml
```

---

## Step 3: Set Up Persistent Storage

### 3.1 Storage via Local Path Provisioner

K3s includes local-path-provisioner by default. Verify:

```bash
# Check if storage class exists
kubectl get storageclass

# Should show:
# NAME                   PROVISIONER             RECLAIMPOLICY
# local-path (default)   rancher.io/local-path   Delete
```

### 3.2 Create Persistent Volume

Create `k3s-manifests/storage-test.yml`:

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 1Gi

---
apiVersion: v1
kind: Pod
metadata:
  name: test-storage
  namespace: default
spec:
  containers:
  - name: busybox
    image: busybox:latest
    command: ["sleep", "3600"]
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: test-pvc
```

Deploy and test:

```bash
# Create manifest directory
mkdir -p k3s-manifests

# Deploy test storage
kubectl apply -f k3s-manifests/storage-test.yml

# Check PVC
kubectl get pvc

# Check if pod can mount
kubectl logs test-storage

# Clean up
kubectl delete -f k3s-manifests/storage-test.yml
```

---

## Step 4: Configure Ingress Controller

K3s includes Traefik by default, but we disabled it for resource savings. Let's use a lighter alternative: Nginx Ingress.

### 4.1 Install Nginx Ingress Controller

```bash
# Add Nginx ingress helm repo
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install lightweight version for ARM
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.image.tag=1.8.1 \
  --set controller.resources.limits.cpu=100m \
  --set controller.resources.limits.memory=90Mi \
  --set controller.metrics.enabled=false
```

### 4.2 Create a Test Ingress

Create `k3s-manifests/test-ingress.yml`:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-app
  template:
    metadata:
      labels:
        app: test-app
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 50m
            memory: 32Mi

---
apiVersion: v1
kind: Service
metadata:
  name: test-app
  namespace: default
spec:
  selector:
    app: test-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-app
  namespace: default
spec:
  ingressClassName: nginx
  rules:
  - host: test-app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: test-app
            port:
              number: 80
```

Deploy and test:

```bash
# Deploy
kubectl apply -f k3s-manifests/test-ingress.yml

# Check deployment
kubectl get all

# Get ingress info
kubectl get ingress

# Test from master node
ssh rocky@192.168.1.100
curl http://test-app.local
# or
curl http://localhost:80
```

---

## Step 5: Create Namespaces

Organize applications into namespaces:

```bash
# Create namespaces
kubectl create namespace monitoring
kubectl create namespace applications
kubectl create namespace cloudflare

# List namespaces
kubectl get namespaces
```

---

## Step 6: Configure Resource Quotas

For better cluster stability, set resource quotas per namespace.

Create `k3s-manifests/resource-quotas.yml`:

```yaml
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: app-quota
  namespace: applications
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
    pods: "20"

---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: monitoring-quota
  namespace: monitoring
spec:
  hard:
    requests.cpu: "500m"
    requests.memory: 512Mi
    limits.cpu: "1"
    limits.memory: 1Gi
    pods: "10"

---
apiVersion: v1
kind: LimitRange
metadata:
  name: app-limits
  namespace: applications
spec:
  limits:
  - max:
      cpu: "500m"
      memory: "512Mi"
    min:
      cpu: "50m"
      memory: "32Mi"
    type: Container
```

Deploy:

```bash
kubectl apply -f k3s-manifests/resource-quotas.yml

# Check quotas
kubectl describe quota -n applications
kubectl describe quota -n monitoring
```

---

## Step 7: Enable Metrics Server

Metrics server is needed for monitoring and horizontal pod autoscaling.

K3s includes it by default. Verify:

```bash
# Check if metrics-server is running
kubectl get deployment metrics-server -n kube-system

# Check metrics (may take 30 seconds)
kubectl top nodes
kubectl top pods
```

---

## Step 8: Deploy a Sample Application

Deploy a simple application to verify everything works.

Create `k3s-manifests/sample-app.yml`:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  namespace: applications
  labels:
    app: hello-world
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello
        image: gcr.io/google-samples/hello-app:2.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 50m
            memory: 32Mi
          limits:
            cpu: 100m
            memory: 64Mi
        env:
        - name: PORT
          value: "8080"

---
apiVersion: v1
kind: Service
metadata:
  name: hello-world
  namespace: applications
spec:
  selector:
    app: hello-world
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-world
  namespace: applications
spec:
  ingressClassName: nginx
  rules:
  - host: hello.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-world
            port:
              number: 80
```

Deploy and test:

```bash
# Deploy
kubectl apply -f k3s-manifests/sample-app.yml

# Check deployment
kubectl get all -n applications

# Get logs
kubectl logs -n applications -l app=hello-world -f

# Test from control machine
curl http://hello.local
# or from cluster
ssh rocky@192.168.1.100
curl http://hello.local
```

---

## Step 9: Configure RBAC (Role-Based Access Control)

Create limited permissions for applications:

Create `k3s-manifests/rbac.yml`:

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-reader
  namespace: applications

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-reader
  namespace: applications
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-reader
  namespace: applications
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-reader
subjects:
- kind: ServiceAccount
  name: app-reader
  namespace: applications
```

Deploy:

```bash
kubectl apply -f k3s-manifests/rbac.yml

# Verify
kubectl get sa -n applications
kubectl get roles -n applications
kubectl get rolebindings -n applications
```

---

## Step 10: Network Policies

Restrict traffic between namespaces (optional but recommended):

Create `k3s-manifests/network-policies.yml`:

```yaml
---
# Deny all ingress traffic by default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: applications
spec:
  podSelector: {}
  policyTypes:
  - Ingress

---
# Allow traffic from ingress controller
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress
  namespace: applications
spec:
  podSelector:
    matchLabels:
      app: hello-world
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
```

Deploy:

```bash
kubectl apply -f k3s-manifests/network-policies.yml
```

---

## Cluster Backup

Create a backup script for your cluster data:

Create `scripts/backup-cluster.sh`:

```bash
#!/bin/bash

BACKUP_DIR=$HOME/backups/k3s-cluster
DATE=$(date +%Y-%m-%d_%H-%M-%S)

# Create backup directory
mkdir -p $BACKUP_DIR

# Backup kubeconfig
cp ~/.kube/config-rpi $BACKUP_DIR/kubeconfig-$DATE.yaml

# Backup all resources
kubectl get all --all-namespaces -o yaml > $BACKUP_DIR/all-resources-$DATE.yaml

# Backup etcd from master
ssh rocky@192.168.1.100 "sudo k3s kubectl get all --all-namespaces -o yaml" > $BACKUP_DIR/master-backup-$DATE.yaml

# Create tarball
tar -czf $BACKUP_DIR/k3s-backup-$DATE.tar.gz -C $BACKUP_DIR kubeconfig-$DATE.yaml all-resources-$DATE.yaml master-backup-$DATE.yaml

echo "Backup created: $BACKUP_DIR/k3s-backup-$DATE.tar.gz"
```

Make it executable and run:

```bash
chmod +x scripts/backup-cluster.sh
./scripts/backup-cluster.sh

# Schedule with cron for daily backups
crontab -e
# Add: 0 2 * * * ~/projects/rpi-cluster/scripts/backup-cluster.sh
```

---

## Useful kubectl Commands

```bash
# View cluster status
kubectl cluster-info
kubectl get nodes -o wide
kubectl top nodes

# Manage applications
kubectl get deployments -A
kubectl get pods -A
kubectl get services -A
kubectl get ingress -A

# View logs
kubectl logs -f deployment/hello-world -n applications

# Execute commands in pods
kubectl exec -it hello-world-xxxx -n applications -- /bin/sh

# Port forward to local machine
kubectl port-forward svc/hello-world 8080:80 -n applications

# Describe resources
kubectl describe node rpi-master
kubectl describe pod hello-world-xxxx -n applications

# Scale deployments
kubectl scale deployment hello-world --replicas=3 -n applications

# Update deployments
kubectl set image deployment/hello-world hello=nginx:1.19 -n applications
```

---

## Monitoring K3s

Real-time cluster monitoring:

```bash
# Watch nodes
watch kubectl get nodes

# Watch pods
watch kubectl get pods -A

# Watch resources
watch kubectl top nodes

# Check K3s service logs on master
ssh rocky@192.168.1.100
sudo journalctl -u k3s -f

# Check K3s agent logs on workers
sudo journalctl -u k3s-agent -f
```

---

## Next Steps

1. **04_PROMETHEUS_GRAFANA.md** - Set up comprehensive monitoring
2. **05_CLOUDFLARE_AGENT.md** - Configure networking and DNS
3. Deploy your own applications to the cluster

---

**Status:** K3s cluster fully configured and ready for applications
**Next:** Monitoring with Prometheus and Grafana
