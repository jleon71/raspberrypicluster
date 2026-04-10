# QNAP TS-473a Ansible Control Machine - Deployment Guide

## Quick Start (15 minutes)

### Prerequisites Checklist

- [ ] QNAP TS-473a running Rocky Linux
- [ ] All Raspberry Pis running Rocky Linux with IP addresses assigned
- [ ] SSH keys generated on QNAP
- [ ] Network connectivity between QNAP and all Raspberry Pis
- [ ] Minimum 2GB free space on QNAP
- [ ] Admin access to QNAP

---

## Phase 1: QNAP Preparation (10 minutes)

### 1.1 SSH into QNAP

```bash
ssh -i ~/.ssh/id_ed25519 rocky@<qnap-ip>
# Or without key if password is set:
ssh rocky@<qnap-ip>
```

### 1.2 Create Storage Structure

```bash
# Create directory structure
sudo mkdir -p /mnt/ansible-data/{inventory,playbooks,roles/{common,k3s,monitoring,cloudflare},group_vars,host_vars,logs,backup,.kubeconfig}

# Set permissions
sudo chown -R rocky:rocky /mnt/ansible-data
sudo chmod -R 755 /mnt/ansible-data

# Verify
ls -la /mnt/ansible-data
```

### 1.3 Install Ansible

```bash
# Update system
sudo dnf update -y

# Install dependencies
sudo dnf install -y python3-pip python3-devel gcc git openssh-clients curl wget

# Install Ansible (user-level)
pip3 install --user ansible ansible-core

# Verify
ansible --version

# Add to PATH if needed
export PATH=$PATH:$HOME/.local/bin
echo 'export PATH=$PATH:$HOME/.local/bin' >> ~/.bashrc
source ~/.bashrc
```

### 1.4 Generate SSH Keys

```bash
# Generate Ed25519 key
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""

# Set permissions
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub

# Display for copying to Raspberry Pis
cat ~/.ssh/id_ed25519.pub
```

---

## Phase 2: Cluster Node Preparation (5 minutes per node)

For **each** Raspberry Pi in your cluster:

### 2.1 Distribute SSH Public Key

**Option A: Using ssh-copy-id (recommended)**

```bash
# From QNAP terminal
ssh-copy-id -i ~/.ssh/id_ed25519.pub rocky@192.168.1.100
ssh-copy-id -i ~/.ssh/id_ed25519.pub rocky@192.168.1.101
# Repeat for all worker IPs
```

**Option B: Manual copy**

```bash
# SSH to each node
ssh rocky@192.168.1.100

# Create .ssh directory if needed
mkdir -p ~/.ssh
chmod 700 ~/.ssh

# Paste public key from QNAP
cat >> ~/.ssh/authorized_keys << 'EOF'
[PASTE PUBLIC KEY HERE]
EOF

# Set permissions
chmod 600 ~/.ssh/authorized_keys

# Exit
exit
```

### 2.2 Verify SSH Access

```bash
# From QNAP, test each node
ssh -i ~/.ssh/id_ed25519 rocky@192.168.1.100 "echo 'Connected to master'"
ssh -i ~/.ssh/id_ed25519 rocky@192.168.1.101 "echo 'Connected to worker-1'"
```

---

## Phase 3: Ansible Project Setup (5 minutes)

### 3.1 Create Project Structure

```bash
cd /mnt/ansible-data

# Initialize git repository
git init rpi-cluster
cd rpi-cluster

# Create directory structure
mkdir -p {inventory,playbooks,roles/{common,k3s,monitoring,cloudflare},group_vars,host_vars,logs,backup,cron}

# Create .gitignore
cat > .gitignore << 'EOF'
__pycache__/
*.pyc
*.pyo
.DS_Store
ansible.log
retry_files/
group_vars/*/secrets.yml
host_vars/*/secrets.yml
.vault_pass
.k3s_token
.k3s_url
EOF

# Initialize git
git config user.email "ansible@qnap"
git config user.name "QNAP Ansible"
git add .
git commit -m "Initial Ansible project structure"
```

### 3.2 Copy Configuration Files

Copy the provided files from this guide to your QNAP:

```bash
# Copy inventory file
cp qnap-ansible-inventory-hosts.ini /mnt/ansible-data/rpi-cluster/inventory/hosts.ini

# Verify
cat /mnt/ansible-data/rpi-cluster/inventory/hosts.ini
```

### 3.3 Create Ansible Config

```bash
cd /mnt/ansible-data/rpi-cluster

# Create ansible.cfg
cat > ansible.cfg << 'EOF'
[defaults]
inventory = inventory/hosts.ini
remote_user = rocky
private_key_file = ~/.ssh/id_ed25519
host_key_checking = False
timeout = 30
connection = ssh
retry_files_enabled = True
log_path = ./ansible.log
roles_path = ./roles

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
pipelining = True
EOF

# Verify
cat ansible.cfg
```

---

## Phase 4: Connectivity Testing (5 minutes)

### 4.1 Test Ping

```bash
cd /mnt/ansible-data/rpi-cluster

# Ansible ping test
ansible all -m ping

# Expected output:
# rpi-master | SUCCESS => {
#     "ping": "pong"
# }
```

### 4.2 Verify Inventory

```bash
# List all hosts
ansible all --list-hosts

# Get facts from hosts
ansible all -m setup | head -50
```

### 4.3 Create Ping Playbook

```bash
# Create playbook
cat > playbooks/00-ping.yml << 'EOF'
---
- name: Test Connectivity
  hosts: all
  gather_facts: no
  tasks:
    - name: Ping
      ping:
    - name: Show hostname
      debug:
        msg: "{{ ansible_hostname }}"
EOF

# Run playbook
ansible-playbook playbooks/00-ping.yml -v
```

---

## Phase 5: Run Configuration Playbooks (30-45 minutes)

### 5.1 Create Common Configuration Playbook

```bash
cat > playbooks/01-common.yml << 'EOF'
---
- name: Common Configuration
  hosts: all
  become: yes
  gather_facts: yes

  tasks:
    - name: Update system
      dnf:
        name: "*"
        state: latest

    - name: Install essential packages
      dnf:
        name:
          - curl
          - wget
          - git
          - htop
          - net-tools
          - vim
          - python3-pip
          - openssh-server
          - openssh-clients
        state: present

    - name: Enable firewalld
      systemd:
        name: firewalld
        state: started
        enabled: yes

    - name: Allow SSH through firewall
      firewalld:
        service: ssh
        permanent: yes
        state: enabled
        immediate: yes

    - name: Configure sudoers
      lineinfile:
        path: /etc/sudoers
        line: '%wheel ALL=(ALL) NOPASSWD:ALL'
        validate: '/usr/sbin/visudo -cf %s'

    - name: Update kernel parameters
      sysctl:
        name: "{{ item.key }}"
        value: "{{ item.value }}"
        state: present
        sysctl_set: yes
      loop:
        - { key: "net.ipv4.ip_forward", value: "1" }
        - { key: "net.bridge.bridge-nf-call-iptables", value: "1" }
        - { key: "net.bridge.bridge-nf-call-ip6tables", value: "1" }

    - name: Create cluster log directory
      file:
        path: /var/log/rpi-cluster
        state: directory
        mode: '0755'
EOF

# Verify syntax
ansible-playbook playbooks/01-common.yml --syntax-check

# Dry run
ansible-playbook playbooks/01-common.yml -C

# Run it
ansible-playbook playbooks/01-common.yml -v
```

### 5.2 Create K3s Installation Playbook

```bash
cat > playbooks/02-k3s.yml << 'EOF'
---
- name: Install K3s Cluster
  hosts: k3s_cluster
  become: yes
  gather_facts: yes

  tasks:
    # Master installation
    - name: Install K3s on master
      block:
        - name: Get K3s installer
          shell: |
            curl -sfL https://get.k3s.io | \
            INSTALL_K3S_EXEC="--disable traefik --disable servicelb" \
            sh -
          args:
            creates: /usr/local/bin/k3s

        - name: Wait for K3s API
          shell: /usr/local/bin/k3s kubectl get nodes
          retries: 30
          delay: 5
          changed_when: false

        - name: Get K3s token
          shell: cat /var/lib/rancher/k3s/server/node-token
          register: k3s_token
          changed_when: false

        - name: Get K3s URL
          set_fact:
            k3s_server_url: "https://{{ ansible_default_ipv4.address }}:6443"

        - name: Save token and URL for workers
          copy:
            content: "{{ k3s_token.stdout }}"
            dest: /tmp/k3s_token
            mode: '0600'

      when: "'masters' in group_names"

    # Worker installation
    - name: Install K3s on workers
      block:
        - name: Get master token
          set_fact:
            k3s_token: "{{ hostvars[groups['masters'][0]]['k3s_token']['stdout'] }}"
            k3s_server_url: "{{ hostvars[groups['masters'][0]]['k3s_server_url'] }}"

        - name: Install K3s agent
          shell: |
            curl -sfL https://get.k3s.io | \
            K3S_URL="{{ k3s_server_url }}" \
            K3S_TOKEN="{{ k3s_token }}" \
            sh -
          args:
            creates: /usr/local/bin/k3s

      when: "'workers' in group_names"

    # Firewall rules
    - name: Configure firewall
      block:
        - name: Allow K3s API (master)
          firewalld:
            port: 6443/tcp
            permanent: yes
            state: enabled
            immediate: yes
          when: "'masters' in group_names"

        - name: Allow kubelet
          firewalld:
            port: 10250/tcp
            permanent: yes
            state: enabled
            immediate: yes
      when: true

  post_tasks:
    - name: Verify installation
      shell: /usr/local/bin/k3s kubectl get nodes
      register: k3s_nodes
      changed_when: false
      when: "'masters' in group_names"

    - name: Display cluster status
      debug:
        msg: "{{ k3s_nodes.stdout }}"
      when: "'masters' in group_names"
EOF

# Run it
ansible-playbook playbooks/02-k3s.yml -v
```

---

## Phase 6: Verify Cluster Installation (5 minutes)

### 6.1 Check Nodes

```bash
# From QNAP, check cluster nodes
/usr/local/bin/k3s kubectl get nodes

# Expected output:
# NAME          STATUS   ROLES                  AGE
# rpi-master    Ready    control-plane,master   2m
# rpi-worker-1  Ready    <none>                 1m
```

### 6.2 Check Pods

```bash
# Get all pods
/usr/local/bin/k3s kubectl get pods -A

# Watch pods
watch /usr/local/bin/k3s kubectl get pods -A
```

### 6.3 Get Kubeconfig

```bash
# Copy from master to QNAP
ansible rpi-master -m fetch -a "src=/etc/rancher/k3s/k3s.yaml dest=/mnt/ansible-data/.kubeconfig/config"

# Update permissions
chmod 600 /mnt/ansible-data/.kubeconfig/config

# Use kubectl from QNAP
export KUBECONFIG=/mnt/ansible-data/.kubeconfig/config
kubectl get nodes
kubectl get pods -A
```

---

## Phase 7: Set Up Automation (10 minutes)

### 7.1 Create Cron Schedule

```bash
# Create cron directory
mkdir -p /mnt/ansible-data/rpi-cluster/cron

# Create cron jobs file
cat > /mnt/ansible-data/rpi-cluster/cron/ansible-jobs.cron << 'EOF'
# Ansible automation jobs
SHELL=/bin/bash
ANSIBLE_LOG_DIR="/mnt/ansible-data/logs"
ANSIBLE_PROJECT_DIR="/mnt/ansible-data/rpi-cluster"

# Daily health check at 2 AM
0 2 * * * cd $ANSIBLE_PROJECT_DIR && ansible-playbook playbooks/01-common.yml >> $ANSIBLE_LOG_DIR/daily-update.log 2>&1

# Hourly connectivity check
0 * * * * cd $ANSIBLE_PROJECT_DIR && ansible all -m ping >> $ANSIBLE_LOG_DIR/hourly-ping.log 2>&1

# Daily backup at 4:30 AM
30 4 * * * tar -czf /mnt/ansible-data/backup/ansible-$(date +\%Y\%m\%d).tar.gz $ANSIBLE_PROJECT_DIR >> $ANSIBLE_LOG_DIR/backup.log 2>&1
EOF

# Install cron jobs
crontab /mnt/ansible-data/rpi-cluster/cron/ansible-jobs.cron

# Verify
crontab -l
```

### 7.2 Create Log Management

```bash
# Create log rotation configuration
sudo tee /etc/logrotate.d/ansible > /dev/null << 'EOF'
/mnt/ansible-data/logs/*.log {
    daily
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 rocky rocky
    sharedscripts
    postrotate
        /bin/systemctl reload rsyslog > /dev/null 2>&1 || true
    endscript
}
EOF

# Test log rotation
sudo logrotate -f /etc/logrotate.d/ansible
```

---

## Phase 8: Documentation and Verification (5 minutes)

### 8.1 Create Project README

```bash
cat > /mnt/ansible-data/rpi-cluster/README.md << 'EOF'
# Raspberry Pi K3s Cluster - QNAP Control Machine

## Quick Commands

# Check cluster status
ansible all -m ping

# Get nodes
ansible-playbook playbooks/00-ping.yml

# Update all systems
ansible-playbook playbooks/01-common.yml

# Install/maintain K3s
ansible-playbook playbooks/02-k3s.yml

# Check K3s status
/usr/local/bin/k3s kubectl get nodes
/usr/local/bin/k3s kubectl get pods -A

## Directory Structure
- inventory/: Host inventory
- playbooks/: Ansible playbooks
- roles/: Ansible roles
- logs/: Execution logs
- backup/: Configuration backups

## Logs Location
- /mnt/ansible-data/logs/ansible.log
- /mnt/ansible-data/logs/daily-update.log
- /mnt/ansible-data/logs/hourly-ping.log

## Useful Links
- Ansible Docs: https://docs.ansible.com
- K3s Docs: https://docs.k3s.io
EOF
```

### 8.2 Create Status Report

```bash
# Create status script
cat > /mnt/ansible-data/rpi-cluster/status.sh << 'EOF'
#!/bin/bash

echo "=== QNAP Ansible Control Machine Status ==="
echo ""
echo "Hostname: $(hostname)"
echo "IP Address: $(hostname -I)"
echo ""

echo "=== Ansible Status ==="
ansible --version | head -1
echo ""

echo "=== K3s Cluster Status ==="
/usr/local/bin/k3s kubectl get nodes 2>/dev/null || echo "K3s not accessible"
echo ""

echo "=== Storage Usage ==="
df -h /mnt/ansible-data | tail -1
echo ""

echo "=== Recent Logs ==="
tail -5 /mnt/ansible-data/logs/ansible.log 2>/dev/null || echo "No logs yet"
echo ""

echo "=== Cluster Nodes ==="
ansible all --list-hosts
EOF

chmod +x /mnt/ansible-data/rpi-cluster/status.sh

# Run status
./status.sh
```

---

## Troubleshooting

### Issue: SSH Connection Refused

```bash
# Check SSH service on nodes
ansible all -m systemd -a "name=sshd state=started"

# Verify SSH key
ansible all -m setup | grep ssh

# Test SSH directly
ssh -v -i ~/.ssh/id_ed25519 rocky@192.168.1.100
```

### Issue: K3s Installation Fails

```bash
# Check K3s logs on master
ansible rpi-master -m shell -a "sudo journalctl -u k3s -n 50"

# Verify firewall
ansible rpi-master -m shell -a "sudo firewall-cmd --list-all"

# Check cgroups
ansible all -m shell -a "cat /proc/cmdline | grep cgroup"
```

### Issue: Ansible Inventory Issues

```bash
# Verify inventory syntax
ansible-inventory -i inventory/hosts.ini --list

# Check specific host
ansible-inventory -i inventory/hosts.ini --host rpi-master

# Test connectivity
ansible all -i inventory/hosts.ini -m ping -vvv
```

---

## Next Steps

1. ✅ Ansible installed and configured
2. ✅ K3s cluster deployed
3. ⏭️ Install monitoring (Prometheus/Grafana)
4. ⏭️ Set up Cloudflare agent
5. ⏭️ Configure persistent storage
6. ⏭️ Deploy applications

---

## Support and Resources

- **Ansible Docs**: https://docs.ansible.com/ansible/latest/
- **K3s Docs**: https://docs.k3s.io/
- **Rocky Linux**: https://rockylinux.org/documentation/
- **QNAP Documentation**: https://www.qnap.com/en-us/support/

---

**Deployment Date**: $(date)
**Control Machine**: QNAP TS-473a
**OS**: Rocky Linux
**Status**: Operational
