# Phase 2: Ansible Setup on QNAP TS-473a as Control Machine

## Overview

This guide configures your QNAP TS-473a running Rocky Linux as a **persistent Ansible control machine** for managing and orchestrating your Raspberry Pi K3s cluster. The QNAP becomes the central hub for all infrastructure automation, monitoring, and scaling operations.

**Benefits of QNAP as Control Machine:**
- Persistent infrastructure orchestration (always on)
- Centralized configuration management
- Scheduled automated tasks and health checks
- Storage for logs, backups, and configuration versions
- Easy scaling of cluster (add nodes without manual intervention)
- Integration with QNAP's management interface

**Estimated Time:** 60-90 minutes

---

## Prerequisites

- QNAP TS-473a running Rocky Linux (see ROCKY_LINUX_MIGRATION_GUIDE.md)
- At least one Raspberry Pi running Rocky Linux (master node)
- Additional Raspberry Pis for workers (optional)
- SSH access configured between QNAP and all Raspberry Pis
- Minimum 2GB free space on QNAP for Ansible projects and logs

---

## Architecture

```
QNAP TS-473a (Rocky Linux)
├── Ansible Control Machine (Python, Ansible, roles, playbooks)
├── Persistent Storage
│   ├── /mnt/ansible-data/inventory
│   ├── /mnt/ansible-data/playbooks
│   ├── /mnt/ansible-data/logs
│   └── /mnt/ansible-data/backup
├── Cron/Scheduler for automated runs
└── SSH Keys for cluster authentication

         ↓ SSH (Port 22)

┌─────────────────────────────────────────────┐
│   Raspberry Pi K3s Cluster (Rocky Linux)    │
├─────────────────────────────────────────────┤
│ Master (192.168.1.100)                      │
│ ├─ K3s Control Plane                        │
│ ├─ Kube-APIServer                           │
│ └─ Etcd                                     │
│                                             │
│ Workers (192.168.1.101+)                    │
│ ├─ K3s Agent Nodes                          │
│ └─ Workload Pods                            │
└─────────────────────────────────────────────┘
```

---

## Step 1: Prepare QNAP Storage

SSH into QNAP and create persistent storage directories:

```bash
# SSH to QNAP
ssh -i ~/.ssh/id_ed25519 rocky@<qnap-ip>

# Create persistent storage structure
sudo mkdir -p /mnt/ansible-data/{inventory,playbooks,roles/{common,k3s,monitoring,cloudflare},group_vars,host_vars,logs,backup}

# Set permissions
sudo chown -R rocky:rocky /mnt/ansible-data
sudo chmod -R 755 /mnt/ansible-data

# Verify structure
tree /mnt/ansible-data
```

---

## Step 2: Install Ansible on QNAP TS-473a

### Install Dependencies

```bash
# Update system
sudo dnf update -y

# Install Python development tools
sudo dnf install -y \
  python3-pip \
  python3-devel \
  gcc \
  git \
  openssh-clients \
  openssh-server \
  curl \
  wget \
  nano \
  vim
```

### Install Ansible

```bash
# Install Ansible via pip (recommended for QNAP)
pip3 install --user ansible ansible-core

# Or system-wide (requires sudo)
sudo pip3 install ansible ansible-core

# Verify installation
ansible --version
# Should show: ansible [version] ... python3 [version]

# Add Ansible to PATH if needed
export PATH=$PATH:$HOME/.local/bin
echo 'export PATH=$PATH:$HOME/.local/bin' >> ~/.bashrc
source ~/.bashrc
```

### Install Additional Ansible Collections (Optional)

```bash
# Install community collections for advanced tasks
ansible-galaxy collection install community.general
ansible-galaxy collection install ansible.posix
```

---

## Step 3: Set Up SSH Key Infrastructure

### Generate SSH Key on QNAP (if not already done)

```bash
# Generate Ed25519 key (more secure than RSA)
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""

# Set proper permissions
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub

# Display public key
cat ~/.ssh/id_ed25519.pub
```

### Distribute Public Key to All Cluster Nodes

```bash
# For each Raspberry Pi in cluster
ssh-copy-id -i ~/.ssh/id_ed25519.pub rocky@192.168.1.100
ssh-copy-id -i ~/.ssh/id_ed25519.pub rocky@192.168.1.101
# ... repeat for all worker nodes
```

### Test SSH Access

```bash
# Test connectivity to all nodes
ssh -i ~/.ssh/id_ed25519 rocky@192.168.1.100 "echo 'Connected to master'"
ssh -i ~/.ssh/id_ed25519 rocky@192.168.1.101 "echo 'Connected to worker-1'"
```

---

## Step 4: Create Ansible Project Structure

### Initialize Git Repository (Recommended)

```bash
# Create project directory
cd /mnt/ansible-data
git init rpi-cluster
cd rpi-cluster

# Create project structure
mkdir -p {inventory,playbooks,roles/{common,k3s,monitoring,cloudflare}}
mkdir -p group_vars host_vars

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
EOF

# Initialize git
git config user.email "ansible@qnap-cluster"
git config user.name "QNAP Ansible"
git add .
git commit -m "Initial Ansible project structure"
```

---

## Step 5: Create Inventory File

Create `/mnt/ansible-data/rpi-cluster/inventory/hosts.ini`:

```ini
# Rocky Linux Raspberry Pi Cluster Inventory
# Managed by QNAP TS-473a Ansible Control Machine

[all:vars]
ansible_user=rocky
ansible_ssh_private_key_file=~/.ssh/id_ed25519
ansible_python_interpreter=/usr/bin/python3
ansible_become=True
ansible_become_method=sudo
ansible_become_user=root
ansible_connection=ssh
ansible_port=22
ansible_timeout=30

# Master node (K3s control plane)
[masters]
rpi-master ansible_host=192.168.1.100 node_type=master cluster_role=control_plane

# Worker nodes (K3s agents)
[workers]
rpi-worker-1 ansible_host=192.168.1.101 node_type=worker
# rpi-worker-2 ansible_host=192.168.1.102 node_type=worker
# rpi-worker-3 ansible_host=192.168.1.103 node_type=worker

# K3s cluster grouping
[k3s_cluster:children]
masters
workers

# Monitoring nodes
[monitoring]
rpi-master

# All cluster nodes
[cluster:children]
k3s_cluster
```

### Create Inventory with Variables

Create `/mnt/ansible-data/rpi-cluster/group_vars/all.yml`:

```yaml
---
# Global variables for all nodes

# System
system_timezone: "UTC"  # Change as needed
system_locale: "en_US.UTF-8"

# Ansible control machine
ansible_control_machine_ip: "<qnap-ip>"
ansible_log_path: "/mnt/ansible-data/logs"
ansible_backup_path: "/mnt/ansible-data/backup"

# K3s cluster
k3s_version: "latest"
k3s_release_url: "https://github.com/k3s-io/k3s/releases/download"

# Firewall
firewall_enabled: true
firewall_service: firewalld

# SSH
ssh_key_type: "ed25519"
ssh_port: 22
```

Create `/mnt/ansible-data/rpi-cluster/group_vars/masters.yml`:

```yaml
---
# Variables for master nodes

k3s_node_type: "master"
k3s_server_disable:
  - "traefik"
  - "servicelb"

# K3s API server settings
k3s_api_bind_address: "0.0.0.0"
k3s_api_port: 6443

# Kubelet settings
k3s_kubelet_max_pods: 110
```

Create `/mnt/ansible-data/rpi-cluster/group_vars/workers.yml`:

```yaml
---
# Variables for worker nodes

k3s_node_type: "worker"
```

---

## Step 6: Create Ansible Configuration File

Create `/mnt/ansible-data/rpi-cluster/ansible.cfg`:

```ini
[defaults]
# Inventory settings
inventory = inventory/hosts.ini

# User and authentication
remote_user = rocky
private_key_file = ~/.ssh/id_ed25519
host_key_checking = False
host_key_auto_add = True

# Connection settings
timeout = 30
connection = ssh
gather_subset = !hardware,!facter

# Behavior
retry_files_enabled = True
retry_files_save_path = ./retry_files/
log_path = ./ansible.log
system_warnings = True
deprecation_warnings = True

# Roles and collections
roles_path = ./roles
collections_paths = ~/.ansible/collections

# Callbacks and plugins
callback_whitelist = profile_tasks,timer
stdout_callback = default

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
pipelining = True
control_path = ~/.ssh/ansible-%%h-%%p-%%r
```

---

## Step 7: Create Test Playbook

Create `/mnt/ansible-data/rpi-cluster/playbooks/00-ping.yml`:

```yaml
---
- name: Test Ansible Connectivity
  hosts: all
  gather_facts: no

  tasks:
    - name: Ping all hosts
      ping:
      register: ping_result

    - name: Display connection status
      debug:
        msg: "Host {{ inventory_hostname }} is reachable"
      when: ping_result.ping == 'pong'

    - name: Get system info
      setup:

    - name: Display system information
      debug:
        msg: |
          Hostname: {{ ansible_hostname }}
          OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
          Python: {{ ansible_python_version }}
          Kernel: {{ ansible_kernel }}
```

---

## Step 8: Test Ansible Connectivity

```bash
# Navigate to project directory
cd /mnt/ansible-data/rpi-cluster

# Check Ansible syntax
ansible-playbook playbooks/00-ping.yml --syntax-check

# Run connectivity test
ansible-playbook playbooks/00-ping.yml -v

# Should output successful ping from all hosts
```

If you encounter errors, troubleshoot:

```bash
# Test SSH connectivity directly
ssh -i ~/.ssh/id_ed25519 rocky@192.168.1.100 "echo success"

# Test Ansible ad-hoc command
ansible all -i inventory/hosts.ini -m ping -v

# Check specific host
ansible rpi-master -i inventory/hosts.ini -m setup
```

---

## Step 9: Create Common Configuration Playbook

Create `/mnt/ansible-data/rpi-cluster/playbooks/01-common.yml`:

```yaml
---
- name: Common Configuration for Rocky Linux Cluster
  hosts: all
  become: yes
  gather_facts: yes

  vars:
    log_dir: "/var/log/rpi-cluster"
    config_dir: "/etc/rpi-cluster"

  tasks:
    - name: Update system packages
      dnf:
        name: "*"
        state: latest
      register: dnf_update

    - name: Install essential packages
      dnf:
        name:
          - curl
          - wget
          - git
          - htop
          - net-tools
          - vim
          - nano
          - openssh-server
          - openssh-clients
          - python3-pip
          - python3-devel
          - gcc
          - make
          - openssl-devel
          - conntrack
          - ebtables
          - ethtool
          - iptables
          - kmod
          - util-linux
        state: present

    - name: Set timezone
      timezone:
        name: "{{ system_timezone | default('UTC') }}"

    - name: Enable and start SSH
      systemd:
        name: sshd
        state: started
        enabled: yes

    - name: Create cluster logging directory
      file:
        path: "{{ log_dir }}"
        state: directory
        mode: '0755'
        owner: root
        group: root

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

    - name: Configure sudoers for wheel group
      lineinfile:
        path: /etc/sudoers
        line: '%wheel ALL=(ALL) NOPASSWD:ALL'
        validate: '/usr/sbin/visudo -cf %s'

    - name: Set SELinux to permissive (for K3s)
      selinux:
        state: permissive

    - name: Update kernel parameters for K3s
      sysctl:
        name: "{{ item.key }}"
        value: "{{ item.value }}"
        state: present
        sysctl_set: yes
      loop:
        - { key: "net.ipv4.ip_forward", value: "1" }
        - { key: "net.bridge.bridge-nf-call-iptables", value: "1" }
        - { key: "net.bridge.bridge-nf-call-ip6tables", value: "1" }
        - { key: "net.netfilter.nf_conntrack_max", value: "262144" }

    - name: Disable password authentication for security
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PasswordAuthentication'
        line: 'PasswordAuthentication no'
      notify: Restart SSH

    - name: Create cluster configuration directory
      file:
        path: "{{ config_dir }}"
        state: directory
        mode: '0755'

    - name: Display update summary
      debug:
        msg: |
          System Update Summary:
          - System updated: {{ dnf_update.changed }}
          - Hostname: {{ inventory_hostname }}
          - Timezone: {{ system_timezone | default('UTC') }}

  handlers:
    - name: Restart SSH
      systemd:
        name: sshd
        state: restarted
```

---

## Step 10: Create K3s Installation Playbook

Create `/mnt/ansible-data/rpi-cluster/playbooks/02-k3s.yml`:

```yaml
---
- name: Install and Configure K3s Cluster
  hosts: k3s_cluster
  become: yes
  gather_facts: yes

  vars:
    k3s_version: "latest"
    k3s_install_url: "https://get.k3s.io"
    k3s_config_dir: "/etc/rancher/k3s"
    k3s_data_dir: "/var/lib/rancher/k3s"

  pre_tasks:
    - name: Validate prerequisites
      block:
        - name: Check for required kernel modules
          shell: |
            lsmod | grep -E 'overlay|br_netfilter'
          register: kernel_modules
          changed_when: false

        - name: Verify cgroups
          shell: cat /proc/cmdline | grep -E 'cgroup'
          register: cgroup_check
          changed_when: false

  tasks:
    - name: Install K3s on master node
      block:
        - name: Get K3s installer
          shell: |
            curl -sfL {{ k3s_install_url }} | \
            INSTALL_K3S_EXEC="--disable traefik --disable servicelb --write-kubeconfig-mode 644" \
            sh -
          args:
            creates: /usr/local/bin/k3s
          register: k3s_install
          retries: 3
          delay: 10

        - name: Wait for K3s API to be ready
          shell: /usr/local/bin/k3s kubectl get nodes
          retries: 30
          delay: 5
          changed_when: false

        - name: Enable K3s service
          systemd:
            name: k3s
            state: started
            enabled: yes

        - name: Get K3s node token
          shell: cat /var/lib/rancher/k3s/server/node-token
          register: k3s_token
          changed_when: false

        - name: Get K3s server URL
          set_fact:
            k3s_server_url: "https://{{ ansible_default_ipv4.address }}:6443"

        - name: Store K3s token in control machine
          copy:
            content: "{{ k3s_token.stdout }}"
            dest: /mnt/ansible-data/rpi-cluster/.k3s_token
            mode: '0600'
          delegate_to: localhost
          become: no

        - name: Store K3s server URL
          copy:
            content: "{{ k3s_server_url }}"
            dest: /mnt/ansible-data/rpi-cluster/.k3s_url
            mode: '0600'
          delegate_to: localhost
          become: no

      when: "'masters' in group_names"

    - name: Install K3s on worker nodes
      block:
        - name: Wait for master to be ready
          wait_for:
            host: "{{ hostvars['rpi-master']['ansible_default_ipv4']['address'] }}"
            port: 6443
            timeout: 300

        - name: Get K3s token from master
          set_fact:
            k3s_token: "{{ lookup('file', '/mnt/ansible-data/rpi-cluster/.k3s_token') }}"
            k3s_server_url: "{{ lookup('file', '/mnt/ansible-data/rpi-cluster/.k3s_url') }}"

        - name: Install K3s agent
          shell: |
            curl -sfL {{ k3s_install_url }} | \
            K3S_URL="{{ k3s_server_url }}" \
            K3S_TOKEN="{{ k3s_token }}" \
            sh -
          args:
            creates: /usr/local/bin/k3s
          register: k3s_agent_install
          retries: 3
          delay: 10

        - name: Enable K3s agent service
          systemd:
            name: k3s-agent
            state: started
            enabled: yes

      when: "'workers' in group_names"

    - name: Configure firewall for K3s
      block:
        - name: Allow K3s API server (master only)
          firewalld:
            port: 6443/tcp
            permanent: yes
            state: enabled
            immediate: yes
          when: "'masters' in group_names"

        - name: Allow kubelet API
          firewalld:
            port: 10250/tcp
            permanent: yes
            state: enabled
            immediate: yes

        - name: Allow metrics server
          firewalld:
            port: 10251/tcp
            permanent: yes
            state: enabled
            immediate: yes

        - name: Allow kube-proxy
          firewalld:
            port: 10256/tcp
            permanent: yes
            state: enabled
            immediate: yes

  post_tasks:
    - name: Verify K3s installation
      shell: /usr/local/bin/k3s kubectl get nodes
      register: k3s_nodes
      changed_when: false
      when: "'masters' in group_names"

    - name: Display cluster status
      debug:
        msg: "{{ k3s_nodes.stdout }}"
      when: "'masters' in group_names"
```

---

## Step 11: Create Persistent Scheduler Configuration

Create `/mnt/ansible-data/rpi-cluster/cron/ansible-jobs.cron`:

```bash
#!/bin/bash
# Ansible automation jobs on QNAP control machine

# Log directory
ANSIBLE_LOG_DIR="/mnt/ansible-data/logs"
ANSIBLE_PROJECT_DIR="/mnt/ansible-data/rpi-cluster"

# Create log entry
log_message() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" >> "$ANSIBLE_LOG_DIR/scheduler.log"
}

# Daily health check and updates (2 AM)
0 2 * * * cd $ANSIBLE_PROJECT_DIR && \
    ansible-playbook playbooks/01-common.yml >> $ANSIBLE_LOG_DIR/daily-update.log 2>&1 && \
    log_message "Daily update completed"

# Weekly cluster maintenance (Sunday 3 AM)
0 3 * * 0 cd $ANSIBLE_PROJECT_DIR && \
    ansible-playbook playbooks/02-k3s.yml >> $ANSIBLE_LOG_DIR/weekly-maintenance.log 2>&1 && \
    log_message "Weekly maintenance completed"

# Hourly connectivity check
0 * * * * cd $ANSIBLE_PROJECT_DIR && \
    ansible all -i inventory/hosts.ini -m ping >> $ANSIBLE_LOG_DIR/hourly-ping.log 2>&1

# Daily backup of configurations
30 4 * * * tar -czf $ANSIBLE_BACKUP_PATH/ansible-backup-$(date +\%Y\%m\%d).tar.gz \
    $ANSIBLE_PROJECT_DIR && \
    log_message "Backup created"
```

Install cron jobs:

```bash
# Edit crontab
crontab -e

# Add the jobs above
# Or load from file:
crontab /mnt/ansible-data/rpi-cluster/cron/ansible-jobs.cron
```

---

## Step 12: Run Playbooks

### Test Connectivity

```bash
cd /mnt/ansible-data/rpi-cluster

# Run ping test
ansible-playbook playbooks/00-ping.yml -v

# List all hosts
ansible all -i inventory/hosts.ini --list-hosts
```

### Dry Run (No Changes)

```bash
# Check syntax
ansible-playbook playbooks/01-common.yml --syntax-check

# Run in check mode (dry run)
ansible-playbook playbooks/01-common.yml -C -v
```

### Run Actual Playbooks

```bash
# Run common configuration
ansible-playbook playbooks/01-common.yml -v | tee logs/01-common-$(date +%Y%m%d).log

# Run K3s installation
ansible-playbook playbooks/02-k3s.yml -v | tee logs/02-k3s-$(date +%Y%m%d).log
```

---

## Step 13: Verify Cluster

### From QNAP

```bash
# Check cluster nodes
/usr/local/bin/k3s kubectl get nodes

# Check all pods
/usr/local/bin/k3s kubectl get pods -A

# Monitor cluster status
watch /usr/local/bin/k3s kubectl get pods -A
```

### From QNAP Ansible

```bash
# Get kubeconfig
ansible rpi-master -i inventory/hosts.ini -m fetch \
  -a "src=/etc/rancher/k3s/k3s.yaml dest=/mnt/ansible-data/.kubeconfig/config"

# Export kubeconfig
export KUBECONFIG=/mnt/ansible-data/.kubeconfig/config

# Use kubectl from QNAP
kubectl get nodes
kubectl get pods -A
```

---

## Step 14: Create Backup and Recovery Playbook

Create `/mnt/ansible-data/rpi-cluster/playbooks/03-backup.yml`:

```yaml
---
- name: Backup K3s Configuration and Data
  hosts: masters
  become: yes

  vars:
    backup_dir: "/mnt/ansible-data/backup"
    backup_date: "{{ ansible_date_time.iso8601_basic_short }}"

  tasks:
    - name: Create backup directory
      file:
        path: "{{ backup_dir }}"
        state: directory
        mode: '0755'

    - name: Backup K3s configuration
      archive:
        path:
          - /etc/rancher/k3s
          - /var/lib/rancher/k3s/server
        dest: "{{ backup_dir }}/k3s-config-{{ backup_date }}.tar.gz"
        format: gz

    - name: Backup etcd database
      shell: |
        /usr/local/bin/k3s kubectl get all --all-namespaces \
          -o yaml > {{ backup_dir }}/k3s-resources-{{ backup_date }}.yaml

    - name: Create backup manifest
      copy:
        content: |
          Backup Date: {{ backup_date }}
          K3s Version: {{ k3s_version }}
          Master: {{ inventory_hostname }}
          Nodes: {{ groups['k3s_cluster'] | join(', ') }}
        dest: "{{ backup_dir }}/backup-{{ backup_date }}.manifest"

    - name: List recent backups
      shell: ls -lh {{ backup_dir }}/
      register: backup_list
      changed_when: false

    - name: Display backup summary
      debug:
        msg: "{{ backup_list.stdout }}"
```

---

## Step 15: Document QNAP as Control Machine

Create `/mnt/ansible-data/rpi-cluster/QNAP_CONTROL_MACHINE.md`:

```markdown
# QNAP TS-473a as Ansible Control Machine

## Role
- Central orchestration and automation hub
- Configuration management for all cluster nodes
- Persistent storage for playbooks, logs, and backups
- Scheduled automated tasks and health checks

## Persistent Storage Structure
```
/mnt/ansible-data/
├── rpi-cluster/              # Main Ansible project
│   ├── inventory/            # Host inventory files
│   ├── playbooks/            # Ansible playbooks
│   ├── roles/                # Ansible roles
│   ├── logs/                 # Execution logs
│   ├── backup/               # Configuration backups
│   └── .kubeconfig/          # Kubernetes configuration
├── logs/                     # QNAP-level logs
└── backup/                   # System backups
```

## Key Commands

# Check cluster status
ansible all -i inventory/hosts.ini -m ping

# Run playbooks
ansible-playbook playbooks/01-common.yml -v

# Get K3s status
/usr/local/bin/k3s kubectl get nodes

# View logs
tail -f /mnt/ansible-data/logs/ansible.log

## Maintenance
- Review logs weekly: `/mnt/ansible-data/logs/`
- Check backups monthly: `/mnt/ansible-data/backup/`
- Update Ansible quarterly: `pip install --upgrade ansible`
- Monitor QNAP storage: Keep > 10% free space
```

---

## Troubleshooting

### Ansible Connection Issues

```bash
# Test SSH directly
ssh -i ~/.ssh/id_ed25519 rocky@192.168.1.100 "uptime"

# Test with verbose Ansible
ansible all -i inventory/hosts.ini -m ping -vvv

# Check SSH configuration
cat /etc/ssh/sshd_config | grep -E "^[^#]"
```

### K3s Installation Failures

```bash
# Check K3s logs
sudo journalctl -u k3s -n 50

# Check firewall rules
sudo firewall-cmd --list-all

# Verify cgroups and kernel modules
cat /proc/cmdline
lsmod | grep overlay
```

### Storage Issues

```bash
# Check QNAP storage usage
df -h /mnt/ansible-data

# Find large files
du -sh /mnt/ansible-data/*

# Clean old logs
find /mnt/ansible-data/logs -mtime +30 -delete
```

---

## Next Steps

1. **Run Ansible playbooks** to configure the cluster
2. **Set up monitoring** with Prometheus and Grafana
3. **Configure Cloudflare Agent** for secure networking
4. **Schedule automated tasks** for maintenance and updates
5. **Document your cluster** configuration and procedures

---

**Status:** QNAP TS-473a ready as Ansible control machine
**Cluster Configuration:** Rocky Linux Raspberry Pi K3s cluster managed by QNAP
