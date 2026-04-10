# Phase 2: Ansible Control Machine via QNAP Virtualization Station

## Overview

This guide configures a **Virtual Rocky Linux Server** on QNAP TS-473a using **Virtualization Station**, then installs **Ansible** as your persistent orchestration platform for managing the Raspberry Pi K3s cluster.

**Why This Approach:**
- QNAP's native services remain unaffected
- Dedicated VM resources for Ansible
- Easy backup and recovery
- Can scale VM resources independently
- Better isolation and security
- Easy to rebuild or clone

**Estimated Total Time:** 2-3 hours

---

## Architecture

```
┌──────────────────────────────────────────────────┐
│         QNAP TS-473a NAS (Native OS)             │
│  ┌────────────────────────────────────────────┐  │
│  │  Virtualization Station                    │  │
│  │  ┌────────────────────────────────────────┐│  │
│  │  │ Rocky Linux Virtual Machine             ││  │
│  │  │ ├─ Ansible Control Machine              ││  │
│  │  │ ├─ Python 3 + Ansible Framework         ││  │
│  │  │ ├─ Playbooks & Inventory               ││  │
│  │  │ ├─ SSH Keys for Cluster Auth           ││  │
│  │  │ └─ Persistent Storage Mount            ││  │
│  │  └────────────────────────────────────────┘│  │
│  └────────────────────────────────────────────┘  │
│                                                  │
│  ┌────────────────────────────────────────────┐  │
│  │ Shared Storage (/share/homes/Ansible)      │  │
│  │ ├─ Playbooks                              │  │
│  │ ├─ Inventory                              │  │
│  │ ├─ Logs                                   │  │
│  │ └─ Backups                                │  │
│  └────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────┘
                        │
        SSH (Port 22, Key-Based Auth)
                        │
       ┌────────────┬───┴────┬─────────────┐
       │            │        │             │
       ↓            ↓        ↓             ↓
    ┌──────┐   ┌──────┐ ┌──────┐    ┌──────┐
    │Master│   │Wkr 1 │ │Wkr 2 │... │Wkr N │
    │192.1 │   │192.1 │ │192.1 │    │192.1 │
    │ .100 │   │ .101 │ │ .102 │    │ .10X │
    └──────┘   └──────┘ └──────┘    └──────┘

    Raspberry Pi K3s Cluster (Rocky Linux)
```

---

## Prerequisites

### QNAP TS-473a Requirements
- ✅ Minimum 100GB free space (VM + storage)
- ✅ Virtualization Station enabled
- ✅ Network configured (bridge mode recommended)
- ✅ Admin access to QNAP

### Cluster Requirements
- ✅ 1+ Raspberry Pis running Rocky Linux
- ✅ Network connectivity to all nodes
- ✅ SSH access available

### Local Requirements
- ✅ QNAP admin credentials
- ✅ Access to QNAP web interface
- ✅ SSH client on your computer (for initial setup)

---

## Part 1: Prepare QNAP Virtualization Station

### Step 1.1: Enable Virtualization Station

**Via QNAP Web Interface:**

1. Log in to QNAP (https://<qnap-ip>:8080 or :443)
2. Navigate: **Control Panel** → **Applications** → **App Center**
3. Search for "**Virtualization Station**"
4. Click **Install**
5. Wait for installation to complete (5-10 minutes)
6. Once installed, go back to **App Center** → **Virtualization Station** → **Open**

### Step 1.2: Verify Virtualization Resources

In Virtualization Station, check:
- ✅ Available CPU cores
- ✅ Available RAM
- ✅ Available disk space
- ✅ Network connectivity

**Recommended VM Resources:**
- vCPU: 4 cores (from total available)
- RAM: 8GB (or 6GB minimum)
- Disk: 50-60GB
- Network: Bridge mode

### Step 1.3: Prepare Storage for VM

Create shared storage for Ansible data:

1. Go to **Control Panel** → **Shared Folders**
2. Click **Create** → **Shared Folder**
3. Name: `Ansible` (or similar)
4. Location: Any available drive with free space
5. Permissions: Read/Write for your user
6. Note the path: `/share/Ansible` (or `/share/homes/Ansible`)

---

## Part 2: Create and Configure Rocky Linux VM

### Step 2.1: Download Rocky Linux ISO

**Two Options:**

**Option A: Download via QNAP (Recommended)**
1. In Virtualization Station, go to **Virtual Machines** → **Resources**
2. Look for **ISO** or **Media** section
3. Click **Upload** or **Download**
4. Download Rocky Linux 8 or 9 ISO
   - URL: https://rockylinux.org/download/
   - Choose: **Rocky-X.X-x86_64-dvd.iso** (about 7-8GB)

**Option B: Pre-download on your computer**
1. Download Rocky Linux ISO locally
2. Upload via QNAP web interface using Virtualization Station

### Step 2.2: Create Virtual Machine

**Via Virtualization Station Web Interface:**

1. Click **Virtual Machines** → **Create**
2. Choose **"New Virtual Machine"**
3. Select **Operating System**: **Linux**
4. Fill in details:
   - **Name**: `Ansible-Control` (or `ansible-master`)
   - **Description**: `Ansible Orchestration Platform`
   - **vCPU**: 4 cores
   - **RAM**: 8GB (8192 MB)
   - **Virtual Disk**: 50GB
   - **Network**: **Bridged** (important for cluster access)

5. Review settings and click **Create**
6. Wait for VM to be created (2-3 minutes)

### Step 2.3: Install Rocky Linux on VM

**In Virtualization Station:**

1. Select your new VM (`Ansible-Control`)
2. Click **Start** (or power icon)
3. A VNC console will open
4. Select the Rocky Linux ISO you downloaded
5. Complete Rocky Linux installation:

   **During Installation:**
   - Hostname: `ansible-control`
   - Network: Enable (automatic DHCP or static)
   - Installation Destination: Select virtual disk
   - Root Password: Set strong password
   - Create User: `ansible` (password or SSH key)

   **Installation Steps:**
   ```
   1. Select language
   2. Select installation destination
   3. Configure network
   4. Set hostname
   5. Set root password
   6. Create user "ansible"
   7. Start Installation
   8. Wait for completion (~10-15 minutes)
   9. Reboot
   ```

6. After reboot, you'll see Rocky Linux login prompt

### Step 2.4: Configure Static IP (Recommended)

**Get Current IP:**
```bash
# In VM console (login as root or ansible)
ip addr show

# Note the IP address (e.g., 192.168.1.xxx)
```

**Set Static IP (on VM):**

Edit `/etc/sysconfig/network-scripts/ifcfg-eth0`:

```bash
# Login to VM as root
sudo -i

# Edit network config
nano /etc/sysconfig/network-scripts/ifcfg-eth0
```

Change to:
```
TYPE=Ethernet
BOOTPROTO=static
NAME=eth0
DEVICE=eth0
ONBOOT=yes
IPADDR=192.168.1.200
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
DNS1=8.8.8.8
```

Restart networking:
```bash
systemctl restart network
# or
nmcli connection reload
nmcli connection up eth0
```

---

## Part 3: Configure Virtual Machine

### Step 3.1: Initial Setup (From VM Console)

**Login to VM:**
```bash
# Via VNC Console or SSH
ssh ansible@192.168.1.200  # or your chosen IP
```

### Step 3.2: Update System

```bash
# Update package manager
sudo dnf update -y

# Install essential packages
sudo dnf install -y \
  git \
  curl \
  wget \
  nano \
  vim \
  openssh-server \
  openssh-clients \
  python3 \
  python3-pip \
  python3-devel \
  gcc \
  make
```

### Step 3.3: Configure Networking

```bash
# Verify network access
ping 8.8.8.8

# Test SSH access from outside VM
# From your computer:
ssh ansible@192.168.1.200  # Should work now
```

### Step 3.4: Mount QNAP Shared Storage

**Create Mount Directory:**
```bash
sudo mkdir -p /mnt/ansible-data
sudo chmod 755 /mnt/ansible-data
```

**Mount QNAP Share:**

Option A: Static Mount (Recommended)

```bash
# First, get QNAP IP address from QNAP console
# QNAP default IP: 192.168.1.1 (or check your network)

# Create credentials file
sudo nano /root/.credentials

# Add:
username=admin
password=your-qnap-password
```

Edit `/etc/fstab`:
```bash
sudo nano /etc/fstab

# Add line:
//192.168.1.1/Ansible /mnt/ansible-data cifs credentials=/root/.credentials,uid=1000,gid=1000 0 0
```

Mount it:
```bash
sudo mount -a
df -h  # Verify mount
```

Option B: Manual NFS Mount

If QNAP has NFS enabled:
```bash
# Install NFS client
sudo dnf install -y nfs-utils

# Mount NFS
sudo mount -t nfs 192.168.1.1:/share/Ansible /mnt/ansible-data
```

### Step 3.5: Set Hostname

```bash
# Check current hostname
hostname

# Set new hostname
sudo hostnamectl set-hostname ansible-control

# Verify
hostname
```

---

## Part 4: Install Ansible

### Step 4.1: Install Python and Pip

```bash
# Already installed in Step 3.2, verify:
python3 --version
pip3 --version
```

### Step 4.2: Install Ansible via Pip

```bash
# Install Ansible globally (recommended for control machine)
sudo pip3 install ansible ansible-core

# Verify installation
ansible --version

# Should show:
# ansible [version]
# python version = X.X.X
```

### Step 4.3: Install Additional Collections

```bash
# Install community and POSIX collections
ansible-galaxy collection install community.general
ansible-galaxy collection install ansible.posix

# List installed collections
ansible-galaxy collection list
```

### Step 4.4: Create Ansible User Configuration

```bash
# Create Ansible user (if not already created during OS install)
sudo useradd -m -s /bin/bash ansible
sudo usermod -aG wheel ansible

# Set password
sudo passwd ansible

# Or add to sudoers for passwordless sudo
echo "ansible ALL=(ALL) NOPASSWD:ALL" | sudo tee -a /etc/sudoers
```

---

## Part 5: Set Up SSH Infrastructure

### Step 5.1: Generate SSH Keys on Ansible VM

```bash
# Login as ansible user
sudo su - ansible

# Generate Ed25519 key
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""

# Verify
ls -la ~/.ssh/
cat ~/.ssh/id_ed25519.pub
```

### Step 5.2: Distribute Public Key to Cluster Nodes

```bash
# Copy the public key
cat ~/.ssh/id_ed25519.pub

# For each Raspberry Pi, run:
ssh-copy-id -i ~/.ssh/id_ed25519.pub rocky@192.168.1.100
ssh-copy-id -i ~/.ssh/id_ed25519.pub rocky@192.168.1.101
# ... repeat for all worker nodes
```

Or manually on each node:
```bash
# SSH to each node
ssh rocky@192.168.1.100

# Create .ssh directory
mkdir -p ~/.ssh
chmod 700 ~/.ssh

# Paste public key
cat >> ~/.ssh/authorized_keys << 'EOF'
[PASTE KEY FROM ABOVE]
EOF

chmod 600 ~/.ssh/authorized_keys
exit
```

### Step 5.3: Test SSH Access

```bash
# From Ansible VM, test each node
ssh -i ~/.ssh/id_ed25519 rocky@192.168.1.100 "echo 'Master connected'"
ssh -i ~/.ssh/id_ed25519 rocky@192.168.1.101 "echo 'Worker-1 connected'"
```

---

## Part 6: Create Ansible Project Structure

### Step 6.1: Initialize Project

```bash
# Create project directory on mounted storage
cd /mnt/ansible-data
mkdir -p rpi-cluster
cd rpi-cluster

# Initialize git
git init
git config user.email "ansible@qnap"
git config user.name "QNAP Ansible VM"
```

### Step 6.2: Create Directory Structure

```bash
# Create subdirectories
mkdir -p {inventory,playbooks,roles/{common,k3s,monitoring,cloudflare}}
mkdir -p group_vars host_vars logs backup

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
git add .
git commit -m "Initial Ansible project structure"
```

### Step 6.3: Create ansible.cfg

```bash
cat > ansible.cfg << 'EOF'
[defaults]
inventory = inventory/hosts.ini
remote_user = rocky
private_key_file = ~/.ssh/id_ed25519
host_key_checking = False
timeout = 30
connection = ssh
gather_facts = True
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
```

---

## Part 7: Create Inventory and Playbooks

### Step 7.1: Create Inventory File

Create `inventory/hosts.ini`:

```bash
cat > inventory/hosts.ini << 'EOF'
# Rocky Linux Raspberry Pi Cluster Inventory
# Managed by Ansible Control Machine (QNAP VM)

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

[masters]
rpi-master ansible_host=192.168.1.100 node_type=master cluster_role=control_plane

[workers]
rpi-worker-1 ansible_host=192.168.1.101 node_type=worker
# rpi-worker-2 ansible_host=192.168.1.102 node_type=worker

[k3s_cluster:children]
masters
workers

[monitoring]
rpi-master

[cluster:children]
k3s_cluster
EOF
```

### Step 7.2: Create Group Variables

Create `group_vars/all.yml`:

```bash
cat > group_vars/all.yml << 'EOF'
---
system_timezone: "UTC"
system_locale: "en_US.UTF-8"
ansible_control_machine: "ansible-control"
ansible_log_path: "/mnt/ansible-data/rpi-cluster/logs"
k3s_version: "latest"
firewall_enabled: true
ssh_key_type: "ed25519"
ssh_port: 22
EOF
```

Create `group_vars/masters.yml`:

```bash
cat > group_vars/masters.yml << 'EOF'
---
k3s_node_type: "master"
k3s_server_disable:
  - "traefik"
  - "servicelb"
k3s_api_bind_address: "0.0.0.0"
k3s_api_port: 6443
k3s_kubelet_max_pods: 110
EOF
```

Create `group_vars/workers.yml`:

```bash
cat > group_vars/workers.yml << 'EOF'
---
k3s_node_type: "worker"
EOF
```

### Step 7.3: Create Test Playbook

Create `playbooks/00-ping.yml`:

```bash
cat > playbooks/00-ping.yml << 'EOF'
---
- name: Test Ansible Connectivity
  hosts: all
  gather_facts: no

  tasks:
    - name: Ping all hosts
      ping:

    - name: Display connection status
      debug:
        msg: "Host {{ inventory_hostname }} is reachable"

    - name: Get system info
      setup:

    - name: Display system information
      debug:
        msg: |
          Hostname: {{ ansible_hostname }}
          OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
          Python: {{ ansible_python_version }}
EOF
```

---

## Part 8: Test Ansible Connectivity

```bash
# Verify syntax
ansible-playbook playbooks/00-ping.yml --syntax-check

# List all hosts
ansible all --list-hosts

# Test connectivity
ansible all -m ping -v

# Should see:
# rpi-master | SUCCESS => {
#     "ping": "pong"
# }
# rpi-worker-1 | SUCCESS => {
#     "ping": "pong"
# }
```

---

## Part 9: Create Configuration Playbooks

### Step 9.1: Common Configuration Playbook

Create `playbooks/01-common.yml`:

```yaml
---
- name: Common Configuration for Rocky Linux
  hosts: all
  become: yes
  gather_facts: yes

  tasks:
    - name: Update system packages
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

    - name: Enable SSH service
      systemd:
        name: sshd
        state: started
        enabled: yes

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

    - name: Set SELinux to permissive
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

    - name: Create cluster log directory
      file:
        path: /var/log/rpi-cluster
        state: directory
        mode: '0755'
```

### Step 9.2: K3s Installation Playbook

Create `playbooks/02-k3s.yml` (see previous documentation for full content)

**Quick version:**

```yaml
---
- name: Install and Configure K3s Cluster
  hosts: k3s_cluster
  become: yes
  gather_facts: yes

  tasks:
    - name: Install K3s on master
      block:
        - name: Download K3s installer
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

      when: "'masters' in group_names"

    - name: Install K3s on workers
      block:
        - name: Get master token
          set_fact:
            k3s_token: "{{ hostvars[groups['masters'][0]]['k3s_token']['stdout'] }}"
            k3s_server_url: "https://{{ groups['masters'][0] }}:6443"

        - name: Install K3s agent
          shell: |
            curl -sfL https://get.k3s.io | \
            K3S_URL="{{ k3s_server_url }}" \
            K3S_TOKEN="{{ k3s_token }}" \
            sh -
          args:
            creates: /usr/local/bin/k3s

      when: "'workers' in group_names"

    - name: Configure firewall rules
      block:
        - name: Allow K3s API port
          firewalld:
            port: 6443/tcp
            permanent: yes
            state: enabled
            immediate: yes
          when: "'masters' in group_names"

        - name: Allow kubelet port
          firewalld:
            port: 10250/tcp
            permanent: yes
            state: enabled
            immediate: yes
```

---

## Part 10: Deploy and Verify

### Step 10.1: Run Playbooks

```bash
cd /mnt/ansible-data/rpi-cluster

# Test connectivity first
ansible-playbook playbooks/00-ping.yml -v

# Dry run
ansible-playbook playbooks/01-common.yml -C -v

# Run common configuration
ansible-playbook playbooks/01-common.yml -v

# Run K3s installation
ansible-playbook playbooks/02-k3s.yml -v
```

### Step 10.2: Verify Cluster

```bash
# Get K3s cluster status
/usr/local/bin/k3s kubectl get nodes

# Should output:
# NAME          STATUS   ROLES                  AGE
# rpi-master    Ready    control-plane,master   2m
# rpi-worker-1  Ready    <none>                 1m
```

---

## Part 11: Set Up Persistent Automation

### Step 11.1: Create Systemd Service (Auto-start)

Create `/etc/systemd/system/ansible-scheduler.service`:

```bash
sudo nano /etc/systemd/system/ansible-scheduler.service
```

Add:
```ini
[Unit]
Description=Ansible Cluster Scheduler
After=network.target
Wants=ansible-scheduler.timer

[Service]
Type=simple
User=ansible
WorkingDirectory=/mnt/ansible-data/rpi-cluster
ExecStart=/usr/local/bin/ansible-playbook playbooks/01-common.yml
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Enable it:
```bash
sudo systemctl daemon-reload
sudo systemctl enable ansible-scheduler.service
```

### Step 11.2: Create Cron Schedule

```bash
# Edit crontab
crontab -e

# Add automation jobs:
# Hourly connectivity check
0 * * * * cd /mnt/ansible-data/rpi-cluster && ansible all -m ping >> logs/hourly-ping.log 2>&1

# Daily health check (2 AM)
0 2 * * * cd /mnt/ansible-data/rpi-cluster && ansible-playbook playbooks/01-common.yml >> logs/daily-update.log 2>&1

# Daily backup (3 AM)
0 3 * * * tar -czf backup/ansible-$(date +\%Y\%m\%d).tar.gz /mnt/ansible-data/rpi-cluster

# Weekly full system update (Sunday 3 AM)
0 3 * * 0 cd /mnt/ansible-data/rpi-cluster && ansible-playbook playbooks/02-k3s.yml >> logs/weekly-update.log 2>&1
```

### Step 11.3: Create Status Monitoring Script

```bash
cat > /mnt/ansible-data/rpi-cluster/status.sh << 'EOF'
#!/bin/bash

echo "=== Ansible Control Machine Status ==="
echo "Hostname: $(hostname)"
echo "IP: $(hostname -I)"
echo ""

echo "=== Ansible Info ==="
ansible --version | head -1
echo ""

echo "=== K3s Cluster Status ==="
/usr/local/bin/k3s kubectl get nodes 2>/dev/null || echo "K3s not accessible"
echo ""

echo "=== Storage Usage ==="
df -h /mnt/ansible-data
echo ""

echo "=== Cluster Nodes ==="
ansible all --list-hosts
echo ""

echo "=== Recent Logs ==="
tail -5 logs/ansible.log 2>/dev/null || echo "No logs yet"
EOF

chmod +x /mnt/ansible-data/rpi-cluster/status.sh
```

---

## Part 12: Backup QNAP VM

### Step 12.1: Create VM Backup

In Virtualization Station:

1. Select `Ansible-Control` VM
2. Right-click → **Backup**
3. Choose destination: QNAP storage
4. Name: `Ansible-Control-Backup-YYYYMMDD`
5. Start backup (5-20 minutes depending on disk usage)

### Step 12.2: Schedule Regular Backups

In Virtualization Station:

1. Select VM → **Backup** → **Schedule**
2. Set: **Daily** at **4:00 AM**
3. Keep: Last 7 backups
4. Enable

---

## Summary Table

| Component | Details |
|-----------|---------|
| **VM Name** | Ansible-Control |
| **OS** | Rocky Linux 8/9 |
| **vCPU** | 4 cores |
| **RAM** | 8GB |
| **Disk** | 50GB |
| **Network** | Bridged (192.168.1.200) |
| **Storage** | /mnt/ansible-data |
| **Control Machine User** | ansible |
| **SSH Key Type** | Ed25519 |
| **Ansible User** | ansible |
| **Playbooks Location** | /mnt/ansible-data/rpi-cluster |

---

## Next Steps

1. ✅ Create and configure VM
2. ✅ Install Ansible
3. ✅ Test cluster connectivity
4. ✅ Run playbooks to configure cluster
5. ⏭️ Set up monitoring (Prometheus/Grafana)
6. ⏭️ Configure Cloudflare agent
7. ⏭️ Deploy applications

---

**Status**: Ansible Control Machine on QNAP Virtualization Station is ready
**Cluster Architecture**: Rocky Linux Raspberry Pi K3s cluster managed by QNAP VM
