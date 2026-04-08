# Quick Reference Card

## Essential Commands

### Cluster Status
```bash
# Node status
kubectl get nodes
kubectl top nodes
kubectl describe node <name>

# Pod status
kubectl get pods -A
kubectl top pods -A
kubectl logs -f pod/<name> -n <namespace>

# Services
kubectl get svc -A
kubectl port-forward svc/<name> 8080:80 -n <namespace>
```

### Ansible
```bash
# Test connectivity
ansible all -m ping

# Run playbook
ansible-playbook playbooks/01-common.yml
ansible-playbook playbooks/02-k3s.yml

# Run on specific host
ansible-playbook playbooks/02-k3s.yml --limit rpi-worker-1

# Check syntax
ansible-playbook playbooks/01-common.yml --syntax-check

# Dry run
ansible-playbook playbooks/01-common.yml -C
```

### K3s Management
```bash
# On master node
ssh debian@192.168.1.100

# Check K3s status
sudo systemctl status k3s

# View K3s logs
sudo journalctl -u k3s -f

# etcd snapshot
sudo k3s etcd-snapshot save <name>
sudo k3s etcd-snapshot list

# Check etcd
sudo k3s kubectl get pods -n kube-system -l component=etcd

# View kubeconfig
cat /etc/rancher/k3s/k3s.yaml
```

### Monitoring
```bash
# Access Grafana
kubectl port-forward svc/grafana 3000:80 -n monitoring
# http://localhost:3000 (admin/admin)

# Access Prometheus
kubectl port-forward svc/prometheus-operated 9090:9090 -n monitoring
# http://localhost:9090

# Check targets
curl http://prometheus-operated.monitoring.svc.cluster.local:9090/api/v1/targets
```

### Backup & Recovery
```bash
# Run backup
~/projects/rpi-cluster/scripts/backup-all.sh

# Check backup
ls -lh ~/backups/

# Restore etcd
scp backup/etcd/snapshot debian@192.168.1.100:/tmp/
sudo k3s etcd-snapshot restore ...

# Restore resources
kubectl apply -f backup/k8s-all-resources.yaml

# Restore secrets
openssl enc -aes-256-cbc -d -in backup/k8s-secrets.yaml.enc | kubectl apply -f -
```

---

## Network Configuration

### IP Ranges
```
Master: 192.168.1.100
Workers: 192.168.1.110+
Services: 192.168.1.200-250
Virtual IP (HA): 192.168.1.50
```

### Ports
```
K3s API: 6443/TCP
Kubelet: 10250/TCP
Metrics: 10255/TCP
Grafana: 3000/TCP
Prometheus: 9090/TCP
HTTP Ingress: 80/TCP
HTTPS Ingress: 443/TCP
```

### DNS
```
Cloudflare Dashboard: https://dash.cloudflare.com
API: https://api.cloudflare.com
Tunnels: https://one.dash.cloudflare.com
```

---

## File Locations

### Key Directories
```
Ansible: ~/projects/rpi-cluster/
Ansible inventory: ~/projects/rpi-cluster/inventory/hosts.ini
K3s config: ~/.kube/config-rpi
Backups: ~/backups/
K3s server: /var/lib/rancher/k3s/
Kubeconfig: /etc/rancher/k3s/k3s.yaml
```

### On Raspberry Pi
```
K3s server: /usr/local/bin/k3s
K3s config: /etc/rancher/k3s/
K3s data: /var/lib/rancher/k3s/
K3s logs: journalctl -u k3s
SSH config: ~/.ssh/
Ansible: /home/debian/.ansible/
```

---

## Troubleshooting Quick Fixes

### Node Not Ready
```bash
# On the node
ssh debian@<ip>
sudo systemctl restart k3s
sudo systemctl status k3s
sudo journalctl -u k3s -n 50

# Check cgroups
cat /proc/cmdline | grep cgroup
```

### Pod Not Starting
```bash
# Check logs
kubectl logs <pod> -n <namespace>
kubectl describe pod <pod> -n <namespace>

# Check resources
kubectl top pods -n <namespace>
kubectl get resourcequota -n <namespace>
```

### Ingress Not Working
```bash
# Check ingress controller
kubectl get pods -n ingress-nginx
kubectl logs -f pod/<name> -n ingress-nginx

# Check ingress
kubectl describe ingress <name> -n <namespace>

# Test from pod
kubectl exec -it <pod> -n <namespace> -- curl http://service:80
```

### Prometheus No Data
```bash
# Check ServiceMonitor
kubectl get servicemonitors -A
kubectl describe sm <name> -n <namespace>

# Check targets in Prometheus UI
http://localhost:9090/targets

# Check Prometheus config
kubectl exec -it prometheus-0 -n monitoring -- cat /etc/prometheus/prometheus.yml
```

### Cloudflare Not Working
```bash
# Check tunnel status
kubectl logs -f deployment/cloudflare-agent -n cloudflare

# Check DNS
nslookup example.com
dig example.com

# Verify tunnel
kubectl exec -it pod/<agent> -n cloudflare -- cloudflared tunnel info
```

---

## Common Tasks

### Add New Node
```bash
# 1. Flash Debian
# Use Balena Etcher with new SD card

# 2. Initial setup
ssh debian@<new-ip>
sudo hostnamectl set-hostname rpi-worker-N

# 3. Update inventory
# Edit inventory/hosts.ini

# 4. Run Ansible
ansible-playbook playbooks/01-common.yml --limit rpi-worker-N
ansible-playbook playbooks/02-k3s.yml --limit rpi-worker-N

# 5. Verify
kubectl get nodes
```

### Remove Node
```bash
# 1. Drain node
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# 2. Delete node
kubectl delete node <node-name>

# 3. On the node
ssh debian@<ip>
sudo /usr/local/bin/k3s-agent-uninstall.sh

# 4. Update inventory
# Remove from inventory/hosts.ini
```

### Update K3s
```bash
# K3s auto-updates via system-upgrade-controller
# Check update status
kubectl get nodes

# Or manually update
ssh debian@192.168.1.100
sudo systemctl stop k3s
# Use installer
curl -sfL https://get.k3s.io | sh -
sudo systemctl start k3s
```

### Deploy Application
```bash
# Create manifest
cat > app.yml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:latest
        ports:
        - containerPort: 8080
EOF

# Deploy
kubectl apply -f app.yml

# Check status
kubectl get deployment myapp
kubectl logs -f deployment/myapp
```

### Create Namespace
```bash
# Create namespace
kubectl create namespace myapp

# Set as default
kubectl config set-context --current --namespace=myapp

# Deploy to namespace
kubectl apply -f app.yml -n myapp

# Check
kubectl get all -n myapp
```

### Increase Storage
```bash
# Current storage
kubectl get pvc -A

# Update PVC (only increase, not decrease)
kubectl patch pvc <name> -n <namespace> \
  -p '{"spec":{"resources":{"requests":{"storage":"10Gi"}}}}'

# Verify
kubectl get pvc -n <namespace>
```

---

## Monitoring Checklist

### Daily
- [ ] Check cluster status: `kubectl get nodes`
- [ ] Review Grafana dashboard
- [ ] Check alerts in Prometheus

### Weekly
- [ ] Review logs: `journalctl -u k3s -n 1000`
- [ ] Check disk usage: `df -h`
- [ ] Verify backups: `ls -lh ~/backups/`

### Monthly
- [ ] Test backup restore
- [ ] Review resource usage
- [ ] Update all software
- [ ] Review security policies

### Quarterly
- [ ] Review architecture
- [ ] Plan for growth
- [ ] Test DR procedures
- [ ] Update documentation

---

## Resource Limits (Per Node)

### Raspberry Pi 4 (4GB)
```
Allocatable CPU: ~2-3 cores
Allocatable Memory: ~2.5 GB
Recommended overhead: 500MB
Available for workloads: ~2 GB
```

### Raspberry Pi 4 (8GB)
```
Allocatable CPU: ~3 cores
Allocatable Memory: ~6.5 GB
Recommended overhead: 1 GB
Available for workloads: ~5.5 GB
```

---

## Performance Tuning

### Reduce Memory Usage
```bash
# Reduce Prometheus retention
kubectl edit prometheus prometheus -n monitoring
# Change: retention: 7d

# Reduce scrape frequency
# Change: scrapeInterval: 60s

# Disable metrics
--disable metrics
```

### Improve Performance
```bash
# Use NVMe SSD (if available)
# Update K3s config: --data-dir=/mnt/nvme

# Increase connection limits
sudo sysctl -w net.ipv4.ip_local_port_range="1024 65535"

# Tune kernel
echo "vm.max_map_count=262144" | sudo tee /etc/sysctl.d/99-k3s.conf
sudo sysctl -p
```

---

## Environment Variables

### Ansible
```bash
export ANSIBLE_HOST_KEY_CHECKING=False
export ANSIBLE_CONFIG=./ansible.cfg
export ANSIBLE_INVENTORY=./inventory/hosts.ini
```

### Kubernetes
```bash
export KUBECONFIG=~/.kube/config-rpi
export KUBECONTEXT=rpi-master
```

### Cloudflare
```bash
export CF_API_TOKEN="your-token"
export CF_ZONE_ID="your-zone-id"
export CF_ACCOUNT_ID="your-account-id"
```

---

## Emergency Procedures

### Complete Cluster Failure
```bash
# 1. Check power and network
ssh debian@192.168.1.100 "echo OK"

# 2. Restart master
ssh debian@192.168.1.100
sudo systemctl restart k3s

# 3. Restart workers (if needed)
for ip in 192.168.1.110 192.168.1.111 192.168.1.112; do
  ssh debian@$ip "sudo systemctl restart k3s-agent"
done

# 4. Wait and verify
sleep 30
kubectl get nodes

# 5. If still down, restore from backup
# See: 07_BACKUP_RECOVERY.md
```

### Data Recovery
```bash
# Check backup exists
ls -lh ~/backups/k3s-backup-*.tar.gz

# Extract backup
tar -xzf ~/backups/k3s-backup-*.tar.gz -C /tmp/

# Restore resources
kubectl apply -f /tmp/backup/k8s-all-resources.yaml

# Restore secrets
openssl enc -aes-256-cbc -d -in /tmp/backup/k8s-secrets.yaml.enc | kubectl apply -f -
```

---

## Useful One-Liners

```bash
# Scale deployment
kubectl scale deployment myapp --replicas=3

# Port forward
kubectl port-forward svc/myapp 8080:80

# Execute command in pod
kubectl exec -it pod/myapp -- /bin/bash

# Copy file from pod
kubectl cp pod/myapp:/data/file.txt ./file.txt

# Delete all pods in namespace
kubectl delete pods --all -n myapp

# Watch changes
watch kubectl get pods -n myapp

# Get pod IP addresses
kubectl get pods -o wide

# Export all resources
kubectl get all -A -o yaml > backup.yaml

# Find pods using PVC
kubectl get pods --all-namespaces -o json | jq '.items[?(@.spec.volumes[*].persistentVolumeClaim)]'
```

---

## Cost Tracking

### Hardware Investment
```
3x Raspberry Pi 4 (8GB): $225
3x 64GB microSD: $60
3x Power supplies: $30
Networking cables: $15
Total: ~$330
```

### Software Cost
```
All open-source: $0
Per month: $0
Per year: $0
5 years: $0
```

### Total 5-Year Cost
```
Hardware: $330 (one-time)
Software: $0 (forever)
Electricity: ~$50 (estimated)
Total: ~$380
```

---

## Bookmarks

### Documentation
- K3s: https://docs.k3s.io
- Kubernetes: https://kubernetes.io
- Ansible: https://docs.ansible.com
- Prometheus: https://prometheus.io
- Grafana: https://grafana.com
- Cloudflare: https://developers.cloudflare.com

### Dashboards
- Grafana: http://localhost:3000
- Prometheus: http://localhost:9090
- Cloudflare: https://dash.cloudflare.com
- Kubernetes Dashboard: `kubectl proxy` then http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/

---

## Last Updated
2026-04-06 | Version 1.0

Keep this card handy while setting up and managing your cluster!
