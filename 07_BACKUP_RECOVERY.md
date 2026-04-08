# Phase 7: Backup and Disaster Recovery

## Overview

Protect your cluster data with comprehensive backup and recovery strategies.

**Estimated Time:** 45 minutes setup + ongoing maintenance

---

## What You'll Do

1. Backup etcd (Kubernetes state)
2. Backup application data and PVCs
3. Backup configuration and secrets
4. Test recovery procedures
5. Schedule automated backups
6. Create runbooks for disaster recovery

---

## Prerequisites

- K3s cluster running
- Sufficient storage for backups
- External storage (USB drive, NAS, or cloud)

---

## Step 1: Understand What to Backup

### Critical Data to Backup

| Data | Location | Impact | Backup Frequency |
|------|----------|--------|------------------|
| **etcd** | `/var/lib/rancher/k3s/server/db/` | Loss = entire cluster state | Daily |
| **Secrets** | Kubernetes secrets | Loss = broken authentication | Daily |
| **PVCs** | PersistentVolumeClaims | Loss = application data | Hourly/Daily |
| **Configurations** | ConfigMaps, manifests | Loss = can't redeploy apps | Daily |
| **Terraform/Ansible** | Your IaC files | Loss = can't rebuild | Daily |

---

## Step 2: Backup etcd (Cluster State)

### 2.1 Manual etcd Backup

On the master node:

```bash
# SSH to master
ssh debian@192.168.1.100

# Create backup directory
mkdir -p ~/k3s-backups

# Backup etcd database
sudo k3s etcd-snapshot save my-backup-$(date +%Y-%m-%d-%H-%M-%S)

# List snapshots
sudo k3s etcd-snapshot list

# Check backup size
sudo ls -lh /var/lib/rancher/k3s/server/db/snapshots/
```

### 2.2 Automated etcd Backup

K3s can automatically snapshot etcd. Verify/configure in systemd service:

```bash
# Check K3s service configuration
sudo systemctl cat k3s | grep snapshot

# Or edit service
sudo systemctl edit k3s

# Add/verify these options:
[Service]
ExecStart=... \
  --etcd-snapshot-retention=5 \
  --etcd-snapshot-schedule-cron="0 */6 * * *"
```

This creates snapshots every 6 hours, keeping last 5.

### 2.3 Copy Backups Externally

Create script: `scripts/backup-etcd.sh`

```bash
#!/bin/bash

BACKUP_DIR="$HOME/backups/etcd"
REMOTE_USER="user"
REMOTE_HOST="backup-server.com"
REMOTE_PATH="/mnt/backups/k3s"

# Create local backup directory
mkdir -p $BACKUP_DIR

# Copy from master node
echo "Fetching etcd snapshots from master..."
scp -r debian@192.168.1.100:/var/lib/rancher/k3s/server/db/snapshots/* \
    $BACKUP_DIR/

# Upload to remote server
echo "Uploading to remote backup server..."
scp -r $BACKUP_DIR/* \
    ${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_PATH}/

# Keep local backups for 30 days
find $BACKUP_DIR -type f -mtime +30 -delete

echo "Backup completed at $(date)"
```

Make executable and schedule:

```bash
chmod +x scripts/backup-etcd.sh

# Add to crontab for daily backups at 2 AM
crontab -e
# Add: 0 2 * * * ~/projects/rpi-cluster/scripts/backup-etcd.sh
```

---

## Step 3: Backup Application Data (PVCs)

### 3.1 List All PVCs

```bash
# Find all persistent volumes
kubectl get pvc -A

# Get detailed info
kubectl describe pvc prometheus-db-prometheus-0 -n monitoring
```

### 3.2 Backup PVC Contents

Create script: `scripts/backup-pvcs.sh`

```bash
#!/bin/bash

BACKUP_DIR="$HOME/backups/pvcs"
NAMESPACE="${1:-monitoring}"

mkdir -p $BACKUP_DIR

# Get all PVCs in namespace
kubectl get pvc -n $NAMESPACE -o name | while read pvc; do
    PVC_NAME=$(echo $pvc | cut -d/ -f2)

    echo "Backing up PVC: $PVC_NAME"

    # Find pod using this PVC
    POD=$(kubectl get pods -n $NAMESPACE \
        -o jsonpath="{.items[?(@.spec.volumes[*].persistentVolumeClaim.claimName=='$PVC_NAME')].metadata.name}" | head -1)

    if [ -z "$POD" ]; then
        echo "  Warning: No pod found using $PVC_NAME"
        continue
    fi

    # Backup PVC contents
    kubectl exec -n $NAMESPACE $POD -- \
        tar -czf - /var/lib/grafana > $BACKUP_DIR/${PVC_NAME}-$(date +%Y-%m-%d).tar.gz

    echo "  Backed up to: $BACKUP_DIR/${PVC_NAME}-$(date +%Y-%m-%d).tar.gz"
done

# Keep only last 7 days of backups
find $BACKUP_DIR -type f -mtime +7 -delete

echo "PVC backup completed at $(date)"
```

---

## Step 4: Backup Kubernetes Resources

### 4.1 Export All Resources

Create script: `scripts/backup-k8s-resources.sh`

```bash
#!/bin/bash

BACKUP_DIR="$HOME/backups/k8s-manifests"
DATE=$(date +%Y-%m-%d)

mkdir -p $BACKUP_DIR/$DATE

echo "Exporting Kubernetes resources..."

# Backup all resources from all namespaces
kubectl get all --all-namespaces -o yaml > $BACKUP_DIR/$DATE/all-resources.yaml

# Backup specific resource types
for resource in configmap secret ingress persistentvolumeclaim storageclass networkpolicy; do
    echo "  Backing up $resource..."
    kubectl get $resource --all-namespaces -o yaml > $BACKUP_DIR/$DATE/${resource}s.yaml
done

# Backup RBAC
kubectl get clusterroles,clusterrolebindings,roles,rolebindings -A -o yaml > $BACKUP_DIR/$DATE/rbac.yaml

# Backup custom resources
kubectl get all,configmap,secret,ingress --all-namespaces -o yaml > $BACKUP_DIR/$DATE/complete.yaml

# Create tarball
tar -czf $BACKUP_DIR/k8s-backup-$DATE.tar.gz -C $BACKUP_DIR $DATE

# Cleanup
rm -rf $BACKUP_DIR/$DATE

# Keep 30 days of backups
find $BACKUP_DIR -name "*.tar.gz" -mtime +30 -delete

echo "Kubernetes resource backup completed"
ls -lh $BACKUP_DIR/*.tar.gz | tail -5
```

### 4.2 Backup with Velero (Advanced)

For production clusters, use Velero:

```bash
# Install Velero
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm install velero vmware-tanzu/velero \
  --namespace velero \
  --create-namespace \
  --set configuration.backupStorageLocation.provider=aws \
  --set configuration.backupStorageLocation.bucket=my-k3s-backups

# Create backup schedule
velero schedule create daily-backup --schedule="0 2 * * *"
```

---

## Step 5: Backup Configuration Files

Backup your Ansible, Terraform, and K3s configurations:

Create script: `scripts/backup-config.sh`

```bash
#!/bin/bash

BACKUP_DIR="$HOME/backups/config"
DATE=$(date +%Y-%m-%d)

mkdir -p $BACKUP_DIR

# Backup Ansible inventory and playbooks
tar -czf $BACKUP_DIR/ansible-$DATE.tar.gz \
    ~/projects/rpi-cluster/

# Backup kubeconfig
tar -czf $BACKUP_DIR/kubeconfig-$DATE.tar.gz \
    ~/.kube/

# Backup SSH keys (with encryption!)
openssl enc -aes-256-cbc -salt -in ~/.ssh/id_ed25519 -out $BACKUP_DIR/ssh-key-$DATE.enc

# Backup Grafana dashboards (if using persistent storage)
kubectl exec -n monitoring deployment/grafana -- \
    grafana-cli admin export-dashboard > $BACKUP_DIR/grafana-dashboards-$DATE.json

# Keep only 30 days
find $BACKUP_DIR -mtime +30 -delete

echo "Config backup completed: $BACKUP_DIR"
```

---

## Step 6: Centralized Backup Strategy

### 6.1 Create Master Backup Script

Create `scripts/backup-all.sh`:

```bash
#!/bin/bash

set -e

BACKUP_ROOT="$HOME/backups/k3s-cluster"
DATE=$(date +%Y-%m-%d-%H-%M-%S)
BACKUP_DIR="$BACKUP_ROOT/$DATE"

mkdir -p $BACKUP_DIR

echo "Starting comprehensive K3s cluster backup..."
echo "Backup directory: $BACKUP_DIR"

# 1. Backup etcd
echo "1. Backing up etcd..."
scp -r debian@192.168.1.100:/var/lib/rancher/k3s/server/db/snapshots/* \
    $BACKUP_DIR/etcd/ 2>/dev/null || mkdir -p $BACKUP_DIR/etcd

# 2. Backup Kubernetes resources
echo "2. Backing up Kubernetes resources..."
kubectl get all --all-namespaces -o yaml > $BACKUP_DIR/k8s-all-resources.yaml

# 3. Backup secrets (encrypted!)
echo "3. Backing up secrets..."
kubectl get secrets --all-namespaces -o yaml | \
    openssl enc -aes-256-cbc -salt -out $BACKUP_DIR/k8s-secrets.yaml.enc

# 4. Backup ConfigMaps
echo "4. Backing up ConfigMaps..."
kubectl get configmaps --all-namespaces -o yaml > $BACKUP_DIR/k8s-configmaps.yaml

# 5. Backup PVC data
echo "5. Backing up PVC data..."
mkdir -p $BACKUP_DIR/pvc-data
# Backup Prometheus
kubectl exec -n monitoring prometheus-0 -- \
    tar -czf - -C /prometheus . | gzip > $BACKUP_DIR/pvc-data/prometheus-$(date +%Y-%m-%d).tar.gz 2>/dev/null || true
# Backup Grafana
kubectl exec -n monitoring deployment/grafana -- \
    tar -czf - /var/lib/grafana | gzip > $BACKUP_DIR/pvc-data/grafana-$(date +%Y-%m-%d).tar.gz 2>/dev/null || true

# 6. Backup kubeconfig
echo "6. Backing up kubeconfig..."
cp ~/.kube/config-rpi $BACKUP_DIR/kubeconfig.yaml

# 7. Backup IaC
echo "7. Backing up Infrastructure as Code..."
tar -czf $BACKUP_DIR/iac.tar.gz \
    ~/projects/rpi-cluster/playbooks/ \
    ~/projects/rpi-cluster/inventory/ \
    ~/projects/rpi-cluster/helm-values/ \
    ~/projects/rpi-cluster/k3s-manifests/

# 8. Create backup index
echo "8. Creating backup index..."
cat > $BACKUP_DIR/BACKUP_INDEX.md << EOF
# K3s Cluster Backup - $DATE

## Contents

- **etcd/**: Kubernetes state database snapshots
- **k8s-all-resources.yaml**: All Kubernetes objects
- **k8s-secrets.yaml.enc**: Encrypted secrets (openssl enc -aes-256-cbc -d)
- **k8s-configmaps.yaml**: All ConfigMaps
- **pvc-data/**: Persistent volume backups
- **kubeconfig.yaml**: Cluster configuration
- **iac.tar.gz**: Ansible playbooks and manifests

## Restore Procedures

### Restore etcd
\`\`\`bash
scp etcd/* debian@192.168.1.100:/var/lib/rancher/k3s/server/db/snapshots/
ssh debian@192.168.1.100
sudo systemctl stop k3s
sudo k3s etcd-snapshot restore <snapshot-name>
sudo systemctl start k3s
\`\`\`

### Restore Kubernetes resources
\`\`\`bash
kubectl apply -f k8s-all-resources.yaml
\`\`\`

### Restore secrets
\`\`\`bash
openssl enc -aes-256-cbc -d -in k8s-secrets.yaml.enc | kubectl apply -f -
\`\`\`

## Backup Verification

- Backup size: $(du -sh $BACKUP_DIR | cut -f1)
- Files: $(find $BACKUP_DIR -type f | wc -l)
- Cluster version: $(kubectl version --short 2>/dev/null | head -1)
- Nodes: $(kubectl get nodes --no-headers | wc -l)
- Timestamp: $(date -R)
EOF

# 9. Compress entire backup
echo "9. Compressing backup..."
tar -czf $BACKUP_ROOT/k3s-backup-$DATE.tar.gz -C $BACKUP_ROOT $DATE
rm -rf $BACKUP_DIR

# 10. Upload to remote (optional)
echo "10. Uploading to remote backup..."
if [ ! -z "$REMOTE_BACKUP_HOST" ]; then
    scp $BACKUP_ROOT/k3s-backup-$DATE.tar.gz \
        $REMOTE_BACKUP_USER@$REMOTE_BACKUP_HOST:$REMOTE_BACKUP_PATH/
    echo "Uploaded to $REMOTE_BACKUP_HOST"
fi

# 11. Cleanup old backups
echo "11. Cleaning up old backups (keeping 30 days)..."
find $BACKUP_ROOT -name "*.tar.gz" -mtime +30 -delete

# Final summary
echo ""
echo "=== BACKUP COMPLETE ==="
echo "Backup file: $BACKUP_ROOT/k3s-backup-$DATE.tar.gz"
echo "Size: $(du -sh $BACKUP_ROOT/k3s-backup-$DATE.tar.gz | cut -f1)"
echo "Timestamp: $(date -R)"
echo "=== END BACKUP ==="
```

Make executable:

```bash
chmod +x scripts/backup-all.sh

# Test run
./scripts/backup-all.sh

# Schedule for daily backups at 2 AM
crontab -e
# Add: 0 2 * * * ~/projects/rpi-cluster/scripts/backup-all.sh
```

---

## Step 7: Test Restore Procedures

### 7.1 Test etcd Restore (Important!)

**⚠️ WARNING:** Only do this on a test cluster or with proper planning.

```bash
# SSH to master
ssh debian@192.168.1.100

# List available snapshots
sudo k3s etcd-snapshot list

# Test restore to temporary location
sudo k3s etcd-snapshot restore \
  --etcd-snapshot-dir=/var/lib/rancher/k3s/server/db/snapshots/ \
  --data-dir=/tmp/k3s-restored \
  my-backup-2024-01-20-12-00-00

# Verify restored data
sudo ls -la /tmp/k3s-restored/member/snap
```

### 7.2 Full Cluster Restore Test

In a test environment:

```bash
# 1. Fresh cluster setup (use Ansible)
ansible-playbook playbooks/01-common.yml --limit test-node
ansible-playbook playbooks/02-k3s.yml --limit test-node

# 2. Restore etcd
scp backup/etcd/snapshot debian@test-node:/tmp/
ssh debian@test-node
sudo k3s etcd-snapshot restore --etcd-snapshot-dir=/tmp/ snapshot

# 3. Restore resources
kubectl apply -f backup/k8s-all-resources.yaml

# 4. Restore secrets
openssl enc -aes-256-cbc -d -in backup/k8s-secrets.yaml.enc | kubectl apply -f -

# 5. Verify
kubectl get all -A
```

---

## Step 8: Disaster Recovery Runbook

Create `docs/DISASTER_RECOVERY_RUNBOOK.md`:

```markdown
# Disaster Recovery Runbook

## Scenarios

### 1. Single Node Failure

**Time to Recover:** 15-30 minutes

1. Remove failed node:
   \`\`\`bash
   kubectl delete node rpi-worker-1
   \`\`\`

2. Replace hardware or fix node

3. Use Ansible to restore:
   \`\`\`bash
   ansible-playbook playbooks/01-common.yml --limit rpi-worker-1
   ansible-playbook playbooks/02-k3s.yml --limit rpi-worker-1
   \`\`\`

4. Verify:
   \`\`\`bash
   kubectl get nodes
   kubectl get pods
   \`\`\`

### 2. Master Node Failure (Single Master)

**Time to Recover:** 1-2 hours

1. Back up etcd from failed node (if possible)

2. Use backup to restore:
   \`\`\`bash
   scp backup/etcd/snapshot debian@192.168.1.100:/tmp/
   ssh debian@192.168.1.100
   sudo systemctl stop k3s
   sudo k3s etcd-snapshot restore --etcd-snapshot-dir=/tmp/ snapshot
   sudo systemctl start k3s
   \`\`\`

3. Verify cluster:
   \`\`\`bash
   kubectl get nodes
   kubectl get all -A
   \`\`\`

### 3. Complete Cluster Failure

**Time to Recover:** 3-4 hours

1. Identify issue (power failure, network, etc.)

2. Hardware recovery:
   - Verify all nodes power up
   - Check network connectivity
   - Check disk integrity

3. Restore from backup:
   \`\`\`bash
   # Restore etcd on master
   scp backup/etcd/snapshot debian@192.168.1.100:/tmp/
   sudo k3s etcd-snapshot restore ...

   # Restore resources
   kubectl apply -f backup/k8s-all-resources.yaml

   # Restore secrets
   openssl enc -aes-256-cbc -d -in backup/k8s-secrets.yaml.enc | kubectl apply -f -
   \`\`\`

4. Verify and validate:
   \`\`\`bash
   kubectl get all -A
   kubectl logs -f deployment/...
   \`\`\`

5. Post-incident:
   - Review backup logs
   - Update recovery procedures
   - Test backups again
   - Document findings

### 4. Data Loss (Accidental Deletion)

**Time to Recover:** 30 minutes

1. Don't panic! Kubernetes has many layers of protection.

2. Check if object is in etcd:
   \`\`\`bash
   kubectl get <resource> --all-namespaces
   \`\`\`

3. If deleted, restore from backup:
   \`\`\`bash
   tar -xzf backup/k8s-backup-2024-01-20.tar.gz
   kubectl apply -f <resource>.yaml
   \`\`\`

4. For PVC data, restore from snapshot:
   \`\`\`bash
   # Copy backup to pod
   kubectl cp backup/pvc-data/grafana.tar.gz pod:/tmp/
   kubectl exec pod -- cd /var/lib/grafana && tar -xzf /tmp/grafana.tar.gz
   \`\`\`

## Recovery Time Objectives (RTO)

| Scenario | RTO | RPO |
|----------|-----|-----|
| Single node failure | 30 min | 0 min |
| Master failure | 2 hours | 6 hours (etcd snapshots) |
| Complete cluster | 4 hours | 24 hours |
| Data loss | 1 hour | 1-6 hours (backup frequency) |

## Backup Verification Checklist

- [ ] Run backup script
- [ ] Verify backup file created
- [ ] Check backup size (should match expectations)
- [ ] Verify encryption (if applicable)
- [ ] Test restore on non-production system
- [ ] Document any issues
- [ ] Update runbook if needed

## Contacts

- **Primary:** [Your contact info]
- **Backup:** [Backup contact info]
- **Escalation:** [Escalation contact]
```

---

## Step 9: Monitor Backup Status

Create monitoring alert for backup failures:

```yaml
---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: backup-alerts
spec:
  groups:
  - name: backups
    interval: 1h
    rules:
    - alert: BackupNotRun
      expr: |
        (time() - max(backup_timestamp)) > 86400
      labels:
        severity: warning
      annotations:
        summary: "Backup not run in 24 hours"
        description: "Last backup: {{ $value }} seconds ago"

    - alert: BackupFailed
      expr: backup_success != 1
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Backup failed"
```

---

## Best Practices

✅ **Do:**
- [ ] Test restores regularly (monthly)
- [ ] Keep backups in multiple locations
- [ ] Encrypt sensitive backups
- [ ] Document all procedures
- [ ] Monitor backup success
- [ ] Update backups after major changes
- [ ] Verify backup integrity
- [ ] Keep audit logs of backups

❌ **Don't:**
- [ ] Skip testing restores
- [ ] Keep only one backup copy
- [ ] Store backups on same storage as primary
- [ ] Assume backups work without verification
- [ ] Leave unencrypted secrets in backups
- [ ] Forget recovery procedures
- [ ] Manually backup without automation
- [ ] Keep backups indefinitely (disk space)

---

## Backup Storage Options

### Local Storage
```bash
# USB drive
sudo mount /dev/sda1 /mnt/backup
cp -r ~/backups/* /mnt/backup/
```

### Network Storage (NAS)
```bash
# Mount NAS via NFS or SMB
sudo mount -t nfs nas.local:/backups /mnt/nas-backup
```

### Cloud Storage (AWS S3)
```bash
# Install aws-cli
pip install awscli

# Upload to S3
aws s3 sync ~/backups/ s3://my-k3s-backups/
```

### Local Encrypted Drive
```bash
# Create encrypted backup drive
sudo cryptsetup luksFormat /dev/sdb1
sudo cryptsetup luksOpen /dev/sdb1 backup
sudo mkfs.ext4 /dev/mapper/backup
sudo mount /dev/mapper/backup /mnt/backup
```

---

## Automation with Systemd Timer

Create `/etc/systemd/system/k3s-backup.service`:

```ini
[Unit]
Description=K3s Cluster Backup
After=network.target

[Service]
Type=oneshot
User=$USER
ExecStart=%h/projects/rpi-cluster/scripts/backup-all.sh
StandardOutput=journal
StandardError=journal
```

Create `/etc/systemd/system/k3s-backup.timer`:

```ini
[Unit]
Description=Daily K3s Backup Timer
Requires=k3s-backup.service

[Timer]
OnCalendar=daily
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now k3s-backup.timer
sudo systemctl status k3s-backup.timer
```

---

## Verification Commands

```bash
# Check backup status
ls -lh ~/backups/

# Verify backup integrity
tar -tzf ~/backups/k3s-backup-*.tar.gz | head -20

# Test backup extraction
tar -xzf ~/backups/k3s-backup-*.tar.gz -C /tmp/test

# Monitor backup schedules
systemctl list-timers k3s-backup.timer
journalctl -u k3s-backup -f
```

---

## Next Steps

1. Implement full backup strategy
2. Test restore procedures
3. Schedule automated backups
4. Document recovery procedures
5. Train team on runbooks

---

**Status:** Backup and recovery procedures implemented
**Next:** Deploy applications and monitor production

