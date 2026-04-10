# QNAP Virtualization Station - Ansible Control Machine Deployment Guide

## Quick Start: 3-4 Hours to Full Cluster Operability

### Prerequisites Checklist

- [ ] QNAP TS-473a with Virtualization Station available
- [ ] 100GB+ free space on QNAP for VM + storage
- [ ] All Raspberry Pis running Rocky Linux with IPs assigned
- [ ] Network connectivity verified (ping between devices)
- [ ] Admin credentials for QNAP web interface
- [ ] SSH client installed on your computer

---

## Phase 1: Enable Virtualization & Prepare (30 minutes)

### Step 1: Enable Virtualization Station

```
1. Open QNAP Web Interface (https://192.168.1.1:8080)
2. Login with admin credentials
3. Control Panel → Applications → App Center
4. Search: "Virtualization Station"
5. Click "Install"
6. Wait 5-10 minutes for installation
7. Click "Open" to launch Virtualization Station
```

### Step 2: Create Shared Storage for Ansible

```
1. Control Panel → Shared Folders
2. Click "Create" → "Shared Folder"
3. Name: "Ansible"
4. Location: Any drive with free space
5. Permissions: Read/Write enabled
6. Click "Create"
7. Note the path shown (e.g., /share/Ansible)
```

### Step 3: Download Rocky Linux ISO

**Option A: Download in Virtualization Station (Recommended)**

```
1. In Virtualization Station, go to "Resources"
2. Click "ISO" → "Download"
3. Select "Rocky Linux" → latest version
4. Click "Download" (this will take 10-15 minutes)
```

**Option B: Pre-download**

```
1. Go to: https://rockylinux.org/download/
2. Download: Rocky-X.X-x86_64-dvd.iso (7-8GB)
3. Upload to QNAP via Virtualization Station
```

---

## Phase 2: Create and Configure VM (1 hour)

### Step 4: Create Virtual Machine

**In Virtualization Station:**

```
1. Click "Virtual Machines" → "Create"
2. Choose "New Virtual Machine"
3. Select OS: "Linux"
4. Fill in details:
   - Name: Ansible-Control
   - Description: Ansible Orchestration
   - vCPU: 4 cores
   - RAM: 8192 MB (8GB)
   - Virtual Disk: 50 GB
   - Network: Bridged
5. Review and click "Create"
6. Wait 2-3 minutes for VM to be created
```

### Step 5: Install Rocky Linux

**In Virtualization Station:**

```
1. Select "Ansible-Control" VM
2. Click "Start" (Power icon)
3. VNC Console opens automatically
4. Rocky Linux installation begins
5. When installer menu appears:
   - Select language: English
   - Click "Installation Destination"
   - Select the virtual disk
   - Click "Done"
6. Network & Hostname:
   - Ensure network is enabled (automatic DHCP)
   - Set hostname: "ansible-control"
7. Root password:
   - Set a strong password
8. Create user:
   - Username: "ansible"
   - Check "Make this user administrator"
   - Set password
9. Click "Begin Installation"
10. Wait 10-15 minutes for installation
11. Click "Reboot" when complete
12. Wait for VM to fully boot (5 minutes)
```

### Step 6: Find VM IP Address

**In VNC Console:**

```bash
# Login as ansible or root
# Run:
ip addr show

# Look for something like: 192.168.1.xxx
# Note this IP address
```

### Step 7: Configure Static IP (Recommended)

**SSH into VM from your computer:**

```bash
ssh ansible@<vm-ip-found-above>
# Enter password when prompted
```

**On the VM:**

```bash
# Edit network configuration
sudo nano /etc/sysconfig/network-scripts/ifcfg-eth0

# Change to static IP:
TYPE=Ethernet
BOOTPROTO=static
NAME=eth0
DEVICE=eth0
ONBOOT=yes
IPADDR=192.168.1.200
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
DNS1=8.8.8.8
DNS2=1.1.1.1

# Save (Ctrl+O, Enter, Ctrl+X)

# Restart networking
sudo systemctl restart network

# Verify
ip addr show
```

---

## Phase 3: Install Ansible (30 minutes)

### Step 8: Update System and Install Dependencies

```bash
# SSH to VM with static IP
ssh ansible@192.168.1.200

# Update system
sudo dnf update -y

# Install packages
sudo dnf install -y \
  python3-pip \
  python3-devel \
  gcc \
  git \
  openssh-server \
  openssh-clients \
  curl \
  wget \
  nano \
  vim

# Verify Python
python3 --version
```

### Step 9: Install Ansible

```bash
# Install Ansible and dependencies
sudo pip3 install ansible ansible-core

# Verify
ansible --version

# Install collections
ansible-galaxy collection install community.general
```

### Step 10: Configure Sudoers

```bash
# Allow ansible user passwordless sudo (already done during install, verify:)
sudo visudo

# Make sure this line exists and is uncommented:
# %wheel ALL=(ALL) NOPASSWD:ALL

# Save if needed (Ctrl+O, Enter, Ctrl+X)
```

---

## Phase 4: Mount QNAP Storage & Setup SSH (30 minutes)

### Step 11: Mount QNAP Shared Storage

**On VM, create mount directory:**

```bash
sudo mkdir -p /mnt/ansible-data
sudo chmod 755 /mnt/ansible-data
```

**Create credentials file:**

```bash
sudo nano /root/.credentials

# Add (replace with your QNAP credentials):
username=admin
password=your_qnap_password
```

**Update fstab:**

```bash
sudo nano /etc/fstab

# Add this line (use QNAP's IP, typically 192.168.1.1):
//192.168.1.1/Ansible /mnt/ansible-data cifs credentials=/root/.credentials,uid=1000,gid=1000 0 0

# Save and exit
```

**Mount the share:**

```bash
sudo mount -a

# Verify
df -h /mnt/ansible-data

# Should show the mounted share with size
```

### Step 12: Generate SSH Keys

```bash
# Switch to ansible user
sudo su - ansible

# Generate SSH key
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""

# Verify
ls -la ~/.ssh/
cat ~/.ssh/id_ed25519.pub  # Copy this output
```

### Step 13: Distribute SSH Keys to Cluster Nodes

**From VM, for each Raspberry Pi:**

```bash
# Copy public key to each node
ssh-copy-id -i ~/.ssh/id_ed25519.pub rocky@192.168.1.100
ssh-copy-id -i ~/.ssh/id_ed25519.pub rocky@192.168.1.101
# ... repeat for all workers

# When prompted, enter the rocky user's password on each node
```

**Test SSH access:**

```bash
# Should connect without password
ssh -i ~/.ssh/id_ed25519 rocky@192.168.1.100 "echo 'Master connected'"
ssh -i ~/.ssh/id_ed25519 rocky@192.168.1.101 "echo 'Worker-1 connected'"
```

---

## Phase 5: Setup Ansible Project (30 minutes)

### Step 14: Create Ansible Project Structure

```bash
# Create project directory
cd /mnt/ansible-data
mkdir -p rpi-cluster
cd rpi-cluster

# Initialize git
git init
git config user.email "ansible@qnap"
git config user.name "QNAP Ansible"

# Create directory structure
mkdir -p inventory playbooks roles/{common,k3s} group_vars host_vars logs backup

# Create .gitignore
cat > .gitignore << 'EOF'
__pycache__/
*.pyc
.DS_Store
ansible.log
retry_files/
*.vault
.k3s_token
EOF

git add .
git commit -m "Initial Ansible project structure"
```

### Step 15: Create Configuration Files

**Create ansible.cfg:**

```bash
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
```

**Create inventory/hosts.ini:**

```bash
cat > inventory/hosts.ini << 'EOF'
[all:vars]
ansible_user=rocky
ansible_ssh_private_key_file=~/.ssh/id_ed25519
ansible_python_interpreter=/usr/bin/python3
ansible_become=True
ansible_become_method=sudo

[masters]
rpi-master ansible_host=192.168.1.100 node_type=master

[workers]
rpi-worker-1 ansible_host=192.168.1.101 node_type=worker
# rpi-worker-2 ansible_host=192.168.1.102 node_type=worker

[k3s_cluster:children]
masters
workers

[monitoring]
rpi-master
EOF
```

**Create group_vars/all.yml:**

```bash
cat > group_vars/all.yml << 'EOF'
---
system_timezone: "UTC"
ansible_log_path: "/mnt/ansible-data/rpi-cluster/logs"
k3s_version: "latest"
firewall_enabled: true
EOF
```

**Create group_vars/masters.yml:**

```bash
cat > group_vars/masters.yml << 'EOF'
---
k3s_node_type: "master"
k3s_server_disable:
  - "traefik"
  - "servicelb"
EOF
```

**Create group_vars/workers.yml:**

```bash
cat > group_vars/workers.yml << 'EOF'
---
k3s_node_type: "worker"
EOF
```

---

## Phase 6: Create Playbooks (20 minutes)

### Step 16: Create Test Playbook

```bash
cat > playbooks/00-ping.yml << 'EOF'
---
- name: Test Connectivity
  hosts: all
  gather_facts: no
  tasks:
    - name: Ping all hosts
      ping:
    - name: Display status
      debug:
        msg: "{{ inventory_hostname }} is reachable"
EOF
```

### Step 17: Create Common Configuration Playbook

```bash
cat > playbooks/01-common.yml << 'EOF'
---
- name: Common Configuration
  hosts: all
  become: yes

  tasks:
    - name: Update system
      dnf:
        name: "*"
        state: latest

    - name: Install packages
      dnf:
        name:
          - curl
          - wget
          - git
          - htop
          - python3-pip
          - openssh-server
          - openssh-clients
        state: present

    - name: Enable firewall
      systemd:
        name: firewalld
        state: started
        enabled: yes

    - name: Allow SSH
      firewalld:
        service: ssh
        permanent: yes
        state: enabled
        immediate: yes

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
EOF
```

### Step 18: Create K3s Installation Playbook

```bash
cat > playbooks/02-k3s.yml << 'EOF'
---
- name: Install K3s
  hosts: k3s_cluster
  become: yes

  tasks:
    - name: Install K3s on master
      block:
        - name: Download K3s
          shell: |
            curl -sfL https://get.k3s.io | \
            INSTALL_K3S_EXEC="--disable traefik --disable servicelb" sh -
          args:
            creates: /usr/local/bin/k3s

        - name: Wait for API
          shell: /usr/local/bin/k3s kubectl get nodes
          retries: 30
          delay: 5
          changed_when: false

        - name: Get token
          shell: cat /var/lib/rancher/k3s/server/node-token
          register: k3s_token
          changed_when: false

      when: "'masters' in group_names"

    - name: Install K3s on workers
      block:
        - name: Get master token
          set_fact:
            k3s_token: "{{ hostvars[groups['masters'][0]]['k3s_token']['stdout'] }}"

        - name: Install agent
          shell: |
            curl -sfL https://get.k3s.io | \
            K3S_URL="https://{{ groups['masters'][0] }}:6443" \
            K3S_TOKEN="{{ k3s_token }}" sh -
          args:
            creates: /usr/local/bin/k3s

      when: "'workers' in group_names"

    - name: Configure firewall
      firewalld:
        port: "{{ item }}"
        permanent: yes
        state: enabled
        immediate: yes
      loop:
        - "6443/tcp"
        - "10250/tcp"
        - "10256/tcp"
EOF
```

---

## Phase 7: Deploy Cluster (30-45 minutes)

### Step 19: Test Connectivity

```bash
cd /mnt/ansible-data/rpi-cluster

# List hosts
ansible all --list-hosts

# Test connectivity
ansible all -m ping -v

# Should see SUCCESS for all hosts
```

### Step 20: Run Playbooks

```bash
# Check syntax
ansible-playbook playbooks/01-common.yml --syntax-check

# Dry run
ansible-playbook playbooks/01-common.yml -C

# Run it
ansible-playbook playbooks/01-common.yml -v
```

Wait 15-20 minutes for all nodes to update.

```bash
# Run K3s installation
ansible-playbook playbooks/02-k3s.yml -v
```

Wait 10-15 minutes for K3s to install and nodes to join.

### Step 21: Verify Cluster

```bash
# Check K3s status from VM
/usr/local/bin/k3s kubectl get nodes

# Expected output:
# NAME          STATUS   ROLES                  AGE
# rpi-master    Ready    control-plane,master   2m
# rpi-worker-1  Ready    <none>                 1m

# Check pods
/usr/local/bin/k3s kubectl get pods -A
```

---

## Phase 8: Setup Automation (15 minutes)

### Step 22: Create Status Script

```bash
cat > /mnt/ansible-data/rpi-cluster/status.sh << 'EOF'
#!/bin/bash

echo "=== Ansible Control Status ==="
echo "Host: $(hostname)"
echo "IP: $(hostname -I | awk '{print $1}')"
echo ""

echo "=== Cluster Status ==="
ansible all --list-hosts
echo ""

echo "=== K3s Nodes ==="
/usr/local/bin/k3s kubectl get nodes 2>/dev/null || echo "K3s not ready"
echo ""

echo "=== Recent Logs ==="
tail -5 logs/ansible.log 2>/dev/null || echo "No logs yet"
EOF

chmod +x /mnt/ansible-data/rpi-cluster/status.sh
```

### Step 23: Setup Cron Automation

```bash
# Edit crontab
crontab -e

# Add:
# Hourly: Connectivity check
0 * * * * cd /mnt/ansible-data/rpi-cluster && ansible all -m ping >> logs/hourly.log 2>&1

# Daily: System updates (2 AM)
0 2 * * * cd /mnt/ansible-data/rpi-cluster && ansible-playbook playbooks/01-common.yml >> logs/daily-update.log 2>&1

# Daily: Backup (3 AM)
0 3 * * * tar -czf backup/ansible-$(date +\%Y\%m\%d).tar.gz /mnt/ansible-data/rpi-cluster >> logs/backup.log 2>&1

# Save (Ctrl+O, Enter, Ctrl+X)
```

### Step 24: Schedule VM Backups

**In Virtualization Station:**

```
1. Select "Ansible-Control" VM
2. Right-click → Backup
3. Set schedule: Daily at 4:00 AM
4. Keep: Last 7 backups
5. Enable automatic backup
```

---

## Final Verification Checklist

- [ ] VM is running and stable
- [ ] Ansible installed and working
- [ ] SSH keys distributed to all nodes
- [ ] Inventory file updated with correct IPs
- [ ] `ansible all -m ping` shows all hosts
- [ ] Playbooks run without errors
- [ ] K3s cluster is fully operational
- [ ] All nodes show Ready status
- [ ] Storage is mounted and accessible
- [ ] Backups configured

---

## Quick Reference Commands

```bash
# Navigate to project
cd /mnt/ansible-data/rpi-cluster

# Test connectivity
ansible all -m ping

# Check hosts
ansible all --list-hosts

# Run a playbook
ansible-playbook playbooks/00-ping.yml -v

# Check K3s cluster
/usr/local/bin/k3s kubectl get nodes

# View logs
tail -f ansible.log

# Check storage
df -h /mnt/ansible-data

# Run status script
./status.sh

# SSH to a node
ssh -i ~/.ssh/id_ed25519 rocky@192.168.1.100
```

---

## Troubleshooting Quick Links

| Issue | Check |
|-------|-------|
| SSH fails | Verify keys: `ls -la ~/.ssh/` |
| Ping fails | Check network: `ping 192.168.1.100` |
| K3s not ready | Check firewall: `sudo firewall-cmd --list-all` |
| Storage not mounted | Try: `sudo mount -a` |
| Playbook errors | Run with: `ansible-playbook playbooks/01-common.yml -vvv` |

---

## Next Steps

1. ✅ VM created and Ansible installed
2. ✅ Cluster configured and K3s running
3. ⏭️ Add monitoring (Prometheus/Grafana)
4. ⏭️ Configure networking (Cloudflare Agent)
5. ⏭️ Deploy applications
6. ⏭️ Plan scaling strategy

---

**Total Deployment Time**: 3-4 hours
**Status**: Ansible Control Machine on QNAP Virtualization Station is operational
**Next**: See QNAP_SCALING_AUTOMATION.md for advanced operations

Good luck! 🚀
