# QNAP Ansible Control Machine - Scaling & Automation

## Overview

This guide covers how to scale your cluster, automate routine tasks, and maintain the QNAP control machine as your infrastructure grows.

---

## Part 1: Scaling the Cluster

### Adding a New Raspberry Pi Node

#### Step 1: Prepare the New Node

```bash
# On the new Raspberry Pi:
# 1. Install Rocky Linux (follow START_HERE_ROCKY_LINUX.md or ROCKY_LINUX_MIGRATION_GUIDE.md)
# 2. SSH into the new node
ssh rocky@<new-node-ip>

# 3. Create .ssh directory
mkdir -p ~/.ssh
chmod 700 ~/.ssh

# 4. Wait for public key from QNAP
```

#### Step 2: Add SSH Key to New Node

From QNAP:

```bash
# Copy SSH key to new node
ssh-copy-id -i ~/.ssh/id_ed25519.pub rocky@<new-node-ip>

# Verify connection
ssh -i ~/.ssh/id_ed25519 rocky@<new-node-ip> "echo 'Success'"
```

#### Step 3: Update Inventory

Edit `/mnt/ansible-data/rpi-cluster/inventory/hosts.ini`:

```ini
# Add new worker to [workers] section
[workers]
rpi-worker-1 ansible_host=192.168.1.101 node_type=worker node_index=1
rpi-worker-2 ansible_host=192.168.1.102 node_type=worker node_index=2  # NEW NODE
rpi-worker-3 ansible_host=192.168.1.103 node_type=worker node_index=3
```

#### Step 4: Run Configuration on New Node

```bash
cd /mnt/ansible-data/rpi-cluster

# Test connectivity to new node only
ansible rpi-worker-2 -m ping

# Run common configuration on new node
ansible-playbook playbooks/01-common.yml --limit rpi-worker-2 -v

# Install K3s on new node
ansible-playbook playbooks/02-k3s.yml --limit rpi-worker-2 -v
```

#### Step 5: Verify New Node Joined Cluster

```bash
# Check if node appears in K3s cluster
/usr/local/bin/k3s kubectl get nodes

# Should see the new node in Ready state
# NAME          STATUS   ROLES   AGE
# rpi-worker-2  Ready    <none>  1m

# Check pod distribution
/usr/local/bin/k3s kubectl get pods -A -o wide
```

### Adding Multiple Nodes at Once

```bash
# Create a batch addition playbook
cat > playbooks/04-add-nodes.yml << 'EOF'
---
- name: Add multiple nodes to cluster
  hosts: new_workers
  become: yes
  gather_facts: yes

  tasks:
    - name: Run common configuration
      include_tasks: ../roles/common/tasks/main.yml

    - name: Install K3s agent
      include_tasks: ../roles/k3s/tasks/agent.yml

    - name: Wait for node to join
      shell: /usr/local/bin/k3s kubectl get nodes | grep {{ inventory_hostname }}
      retries: 30
      delay: 5
      delegate_to: "{{ groups['masters'][0] }}"
      changed_when: false

    - name: Verify node is ready
      shell: /usr/local/bin/k3s kubectl get nodes {{ inventory_hostname }} -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'
      register: node_status
      until: node_status.stdout == 'True'
      retries: 10
      delay: 10
      delegate_to: "{{ groups['masters'][0] }}"
      changed_when: false
EOF

# Update inventory to use new_workers group
# Run with: ansible-playbook playbooks/04-add-nodes.yml -v
```

---

## Part 2: Automated Maintenance Tasks

### Daily Health Checks

Create `/mnt/ansible-data/rpi-cluster/playbooks/05-health-check.yml`:

```yaml
---
- name: Daily Cluster Health Check
  hosts: k3s_cluster
  become: yes
  gather_facts: yes

  vars:
    health_log: "/var/log/rpi-cluster/health-check.log"

  tasks:
    - name: Check system resources
      block:
        - name: Get memory usage
          shell: free | grep Mem | awk '{print int($3/$2 * 100)}'
          register: memory_usage
          changed_when: false

        - name: Get disk usage
          shell: df /var | tail -1 | awk '{print int($3/$2 * 100)}'
          register: disk_usage
          changed_when: false

        - name: Get CPU load
          shell: uptime | awk -F'load average:' '{print $2}'
          register: cpu_load
          changed_when: false

        - name: Log system metrics
          copy:
            content: |
              Timestamp: {{ ansible_date_time.iso8601 }}
              Hostname: {{ inventory_hostname }}
              Memory Usage: {{ memory_usage.stdout }}%
              Disk Usage: {{ disk_usage.stdout }}%
              Load Average: {{ cpu_load.stdout }}
            dest: "{{ health_log }}"
            mode: '0644'

    - name: Check service health
      block:
        - name: Check SSH service
          systemd:
            name: sshd
          register: ssh_status

        - name: Check firewall (if enabled)
          systemd:
            name: firewalld
          register: firewall_status
          ignore_errors: yes

        - name: Alert on service failures
          debug:
            msg: "WARNING: {{ item }} is not running!"
          when: item.status != "enabled"
          loop:
            - "{{ ssh_status }}"
            - "{{ firewall_status }}"

    - name: Check K3s on master
      block:
        - name: K3s status
          systemd:
            name: k3s
          register: k3s_status

        - name: Get K3s node status
          shell: /usr/local/bin/k3s kubectl get nodes -o json
          register: k3s_nodes
          changed_when: false

        - name: Check for NotReady nodes
          debug:
            msg: "WARNING: Node {{ item.metadata.name }} is not ready!"
          when: item.status.conditions[3].status != 'True'
          loop: "{{ (k3s_nodes.stdout | from_json).items }}"

      when: "'masters' in group_names"

    - name: Collect metrics summary
      debug:
        msg: |
          Health Check Summary for {{ inventory_hostname }}
          ================================================
          Memory: {{ memory_usage.stdout }}% used
          Disk: {{ disk_usage.stdout }}% used
          Load: {{ cpu_load.stdout }}
          SSH: {{ ssh_status.status }}
          Firewall: {{ firewall_status.status | default('N/A') }}
```

### Weekly System Updates

Create `/mnt/ansible-data/rpi-cluster/playbooks/06-weekly-update.yml`:

```yaml
---
- name: Weekly System Update and Maintenance
  hosts: k3s_cluster
  become: yes
  gather_facts: yes

  vars:
    backup_dir: "/mnt/ansible-data/backup"
    update_log: "/var/log/rpi-cluster/updates.log"

  tasks:
    - name: Update system packages
      block:
        - name: Update package lists
          dnf:
            name: "*"
            state: latest
          register: update_result

        - name: Log update results
          copy:
            content: |
              Update Date: {{ ansible_date_time.iso8601 }}
              Hostname: {{ inventory_hostname }}
              Packages Updated: {{ update_result.results | length | default(0) }}
              Changes Made: {{ update_result.changed }}
            dest: "{{ update_log }}"
            mode: '0644'

    - name: Cleanup disk space
      block:
        - name: Remove package cache
          shell: dnf clean all
          changed_when: false

        - name: Remove old logs
          shell: find /var/log -type f -mtime +30 -delete
          changed_when: false

    - name: Backup configuration on master
      block:
        - name: Backup K3s config
          archive:
            path:
              - /etc/rancher/k3s
              - /var/lib/rancher/k3s/server
            dest: "{{ backup_dir }}/k3s-backup-{{ ansible_date_time.date }}.tar.gz"
            format: gz

        - name: Backup etcd
          shell: |
            /usr/local/bin/k3s kubectl get all --all-namespaces -o yaml > \
            {{ backup_dir }}/k3s-resources-{{ ansible_date_time.date }}.yaml

      when: "'masters' in group_names"

    - name: Verify services after update
      block:
        - name: Verify SSH
          systemd:
            name: sshd
            state: started
            enabled: yes

        - name: Verify firewall
          systemd:
            name: firewalld
            state: started
            enabled: yes

        - name: Verify K3s (master)
          systemd:
            name: k3s
            state: started
            enabled: yes
          when: "'masters' in group_names"

        - name: Verify K3s (workers)
          systemd:
            name: k3s-agent
            state: started
            enabled: yes
          when: "'workers' in group_names"

  post_tasks:
    - name: Display update summary
      debug:
        msg: |
          Weekly Update Complete
          =====================
          Host: {{ inventory_hostname }}
          Packages Updated: {{ update_result.changed }}
          Backup Location: {{ backup_dir }}
          Log: {{ update_log }}
```

---

## Part 3: Cron-Based Automation

### Complete Cron Configuration

```bash
# Create comprehensive cron schedule
cat > /mnt/ansible-data/rpi-cluster/cron/complete-schedule.cron << 'EOF'
# Complete Ansible Automation Schedule
# QNAP TS-473a Control Machine

SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
ANSIBLE_PROJECT_DIR="/mnt/ansible-data/rpi-cluster"
ANSIBLE_LOG_DIR="/mnt/ansible-data/logs"
BACKUP_DIR="/mnt/ansible-data/backup"

# ===== HOURLY TASKS =====

# Hourly connectivity check (every hour at :00)
0 * * * * cd $ANSIBLE_PROJECT_DIR && ansible all -m ping >> $ANSIBLE_LOG_DIR/hourly-ping.log 2>&1

# ===== DAILY TASKS =====

# Daily health check (2:00 AM)
0 2 * * * cd $ANSIBLE_PROJECT_DIR && ansible-playbook playbooks/05-health-check.yml >> $ANSIBLE_LOG_DIR/daily-health.log 2>&1

# Daily updates (2:30 AM)
30 2 * * * cd $ANSIBLE_PROJECT_DIR && ansible-playbook playbooks/01-common.yml >> $ANSIBLE_LOG_DIR/daily-update.log 2>&1

# Daily backup of Ansible project (3:00 AM)
0 3 * * * tar -czf $BACKUP_DIR/ansible-backup-$(date +\%Y\%m\%d).tar.gz $ANSIBLE_PROJECT_DIR >> $ANSIBLE_LOG_DIR/backup.log 2>&1

# Daily log rotation (4:00 AM)
0 4 * * * find $ANSIBLE_LOG_DIR -name "*.log" -mtime +30 -delete >> $ANSIBLE_LOG_DIR/cleanup.log 2>&1

# ===== WEEKLY TASKS =====

# Weekly full system update and maintenance (Sunday 3:00 AM)
0 3 * * 0 cd $ANSIBLE_PROJECT_DIR && ansible-playbook playbooks/06-weekly-update.yml >> $ANSIBLE_LOG_DIR/weekly-update.log 2>&1

# Weekly backup verification (Sunday 4:00 AM)
0 4 * * 0 ls -lh $BACKUP_DIR/ >> $ANSIBLE_LOG_DIR/backup-list.log 2>&1

# ===== MONTHLY TASKS =====

# Monthly cleanup of old backups (1st of month at 1:00 AM)
0 1 1 * * find $BACKUP_DIR -name "*.tar.gz" -mtime +90 -delete >> $ANSIBLE_LOG_DIR/cleanup.log 2>&1

# Monthly full system check (1st of month at 2:00 AM)
0 2 1 * * cd $ANSIBLE_PROJECT_DIR && ansible-playbook playbooks/05-health-check.yml >> $ANSIBLE_LOG_DIR/monthly-check.log 2>&1

# ===== MONITORING =====

# Generate daily status report (5:00 AM)
0 5 * * * cd $ANSIBLE_PROJECT_DIR && ./status.sh >> $ANSIBLE_LOG_DIR/daily-status.log 2>&1
EOF

# Install the cron schedule
crontab /mnt/ansible-data/rpi-cluster/cron/complete-schedule.cron

# Verify
crontab -l
```

### Creating a Cron Management Script

```bash
cat > /mnt/ansible-data/rpi-cluster/cron/manage-cron.sh << 'EOF'
#!/bin/bash

CRON_FILE="/mnt/ansible-data/rpi-cluster/cron/complete-schedule.cron"
LOG_DIR="/mnt/ansible-data/logs"

case "$1" in
    install)
        echo "Installing cron jobs..."
        crontab "$CRON_FILE"
        echo "✓ Cron jobs installed"
        crontab -l
        ;;
    remove)
        echo "Removing cron jobs..."
        crontab -r
        echo "✓ Cron jobs removed"
        ;;
    list)
        echo "Current cron jobs:"
        crontab -l
        ;;
    test)
        echo "Testing Ansible connectivity..."
        cd /mnt/ansible-data/rpi-cluster
        ansible all -m ping
        ;;
    logs)
        echo "Recent cron execution logs:"
        tail -20 "$LOG_DIR"/*.log
        ;;
    status)
        echo "=== Cron Status ==="
        echo "Cron Service: $(systemctl is-active crond)"
        echo ""
        echo "=== Installed Jobs ==="
        crontab -l 2>/dev/null || echo "No cron jobs installed"
        echo ""
        echo "=== Recent Logs ==="
        ls -lh "$LOG_DIR"/*.log 2>/dev/null | tail -5
        ;;
    *)
        echo "Usage: $0 {install|remove|list|test|logs|status}"
        exit 1
        ;;
esac
EOF

chmod +x /mnt/ansible-data/rpi-cluster/cron/manage-cron.sh

# Test it
/mnt/ansible-data/rpi-cluster/cron/manage-cron.sh status
```

---

## Part 4: Monitoring and Alerting

### Create Monitoring Playbook

Create `/mnt/ansible-data/rpi-cluster/playbooks/07-monitoring.yml`:

```yaml
---
- name: Setup Monitoring and Alerting
  hosts: monitoring
  become: yes
  gather_facts: yes

  tasks:
    - name: Install monitoring tools
      dnf:
        name:
          - htop
          - iotop
          - nethogs
          - sysstat
        state: present

    - name: Create monitoring directory
      file:
        path: /opt/cluster-monitoring
        state: directory
        mode: '0755'

    - name: Create monitoring script
      copy:
        dest: /opt/cluster-monitoring/check-cluster.sh
        mode: '0755'
        content: |
          #!/bin/bash

          echo "=== K3s Cluster Status Check ==="
          echo "Time: $(date)"
          echo ""

          echo "Nodes:"
          /usr/local/bin/k3s kubectl get nodes
          echo ""

          echo "Pod Status:"
          /usr/local/bin/k3s kubectl get pods -A --field-selector=status.phase!=Running
          echo ""

          echo "Resource Usage:"
          /usr/local/bin/k3s kubectl top nodes 2>/dev/null || echo "Metrics server not available"
          echo ""

          echo "System Disk Usage:"
          df -h / /var
          echo ""

          echo "Check complete"

    - name: Create alert script
      copy:
        dest: /opt/cluster-monitoring/check-alerts.sh
        mode: '0755'
        content: |
          #!/bin/bash

          ALERT_LOG="/var/log/rpi-cluster/alerts.log"

          # Check for NotReady nodes
          NOT_READY=$(/usr/local/bin/k3s kubectl get nodes --field-selector=status.phase!=Running | wc -l)
          if [ $NOT_READY -gt 0 ]; then
              echo "[$(date)] ALERT: $NOT_READY nodes not ready" >> $ALERT_LOG
          fi

          # Check for failed pods
          FAILED=$(/usr/local/bin/k3s kubectl get pods -A --field-selector=status.phase=Failed | wc -l)
          if [ $FAILED -gt 0 ]; then
              echo "[$(date)] ALERT: $FAILED failed pods" >> $ALERT_LOG
          fi

          # Check disk space
          DISK=$(df / | tail -1 | awk '{print int($5)}')
          if [ $DISK -gt 80 ]; then
              echo "[$(date)] ALERT: Disk usage at ${DISK}%" >> $ALERT_LOG
          fi

          # Check memory
          MEM=$(free | grep Mem | awk '{print int($3/$2*100)}')
          if [ $MEM -gt 80 ]; then
              echo "[$(date)] ALERT: Memory usage at ${MEM}%" >> $ALERT_LOG
          fi
```

---

## Part 5: Scaling Best Practices

### Pre-Scaling Checklist

```bash
cat > /mnt/ansible-data/rpi-cluster/docs/scaling-checklist.md << 'EOF'
# Cluster Scaling Checklist

## Before Adding Nodes

- [ ] Verify current cluster health
- [ ] Ensure QNAP has sufficient storage (> 10% free)
- [ ] Update Ansible inventory with new node details
- [ ] Prepare new Raspberry Pi with Rocky Linux
- [ ] Test SSH connectivity to new node
- [ ] Verify network connectivity between all nodes
- [ ] Check K3s API server is healthy

## Node Addition Process

- [ ] SSH key distribution to new node
- [ ] Update inventory file
- [ ] Run common configuration playbook
- [ ] Run K3s installation playbook
- [ ] Verify node joins cluster
- [ ] Monitor node for 15 minutes
- [ ] Run health check playbook

## After Adding Nodes

- [ ] Verify all nodes show Ready status
- [ ] Check pod distribution across nodes
- [ ] Verify storage is healthy
- [ ] Update documentation
- [ ] Test cluster failover
- [ ] Schedule backup

## Performance Monitoring

- [ ] Monitor CPU usage on new node
- [ ] Monitor memory usage
- [ ] Monitor network bandwidth
- [ ] Check K3s pod scheduling
- [ ] Verify no node bottlenecks
EOF
```

### Cluster Growth Planning

```markdown
# Recommended Growth Path

## Phase 1: Foundation (3 nodes)
- 1x Master (control plane)
- 2x Workers (workload)
- Storage: 2TB
- Network: Gigabit

## Phase 2: Scale-Out (6 nodes)
- 1x Master (HA ready)
- 5x Workers
- Storage: 4TB
- Network: Gigabit + backup

## Phase 3: High Availability (9+ nodes)
- 3x Masters (HA cluster)
- 6+ Workers
- Storage: 8TB+
- Network: Redundant Gigabit

## Per-Node Requirements
- Raspberry Pi 4/5 (4GB+ RAM recommended)
- 64GB+ microSD or USB SSD
- Stable network connectivity
- Power backup capability
```

---

## Part 6: Backup and Disaster Recovery

### Automated Backup Strategy

```bash
cat > /mnt/ansible-data/rpi-cluster/playbooks/08-backup-strategy.yml << 'EOF'
---
- name: Comprehensive Backup Strategy
  hosts: masters
  become: yes

  vars:
    backup_dir: "/mnt/ansible-data/backup"
    retention_days: 30

  tasks:
    - name: Create backup structure
      file:
        path: "{{ backup_dir }}/{{ item }}"
        state: directory
      loop:
        - "k3s"
        - "etcd"
        - "applications"
        - "configs"

    - name: Backup K3s configuration
      archive:
        path:
          - /etc/rancher/k3s
          - /var/lib/rancher/k3s/server/certs
        dest: "{{ backup_dir }}/k3s/k3s-config-{{ ansible_date_time.date }}.tar.gz"
        format: gz

    - name: Backup etcd database
      shell: |
        /usr/local/bin/k3s kubectl get all --all-namespaces -o yaml > \
        {{ backup_dir }}/etcd/k3s-state-{{ ansible_date_time.date }}.yaml

    - name: Backup application manifests
      shell: |
        /usr/local/bin/k3s kubectl get all -A -o yaml > \
        {{ backup_dir }}/applications/all-resources-{{ ansible_date_time.date }}.yaml

    - name: Create backup manifest
      copy:
        content: |
          Backup Date: {{ ansible_date_time.iso8601 }}
          K3s Version: $(cat /usr/local/bin/k3s version)
          Nodes: {{ groups['k3s_cluster'] | join(', ') }}
          Backup Location: {{ backup_dir }}
        dest: "{{ backup_dir }}/BACKUP_{{ ansible_date_time.date }}.manifest"

    - name: Cleanup old backups
      shell: find {{ backup_dir }}/* -type f -mtime +{{ retention_days }} -delete
      changed_when: false
EOF
```

---

## Summary: Complete Automation Workflow

```
QNAP TS-473a Control Machine
│
├── Cron Scheduler
│   ├── Hourly: Connectivity checks
│   ├── Daily: Health checks, updates, backups
│   └── Weekly: Full maintenance, backups
│
├── Ansible Playbooks
│   ├── 00-ping.yml: Connectivity test
│   ├── 01-common.yml: System configuration
│   ├── 02-k3s.yml: Kubernetes setup
│   ├── 05-health-check.yml: Monitoring
│   ├── 06-weekly-update.yml: Maintenance
│   ├── 07-monitoring.yml: Observability
│   └── 08-backup-strategy.yml: Disaster recovery
│
├── Storage
│   ├── /mnt/ansible-data/inventory/: Host configs
│   ├── /mnt/ansible-data/playbooks/: Automation
│   ├── /mnt/ansible-data/logs/: Execution logs
│   └── /mnt/ansible-data/backup/: Backups
│
└── K3s Cluster
    ├── Master (control plane)
    ├── Workers (scalable)
    └── Applications (your workloads)
```

---

**Status**: Scalable, automated, resilient infrastructure
**Next**: Deploy applications to your cluster
