# Phase 2: Ansible Setup & Infrastructure Automation

## Overview

Ansible allows you to configure all your Raspberry Pis automatically from your control machine. This is the foundation for scalable infrastructure.

**Estimated Time:** 45 minutes - 1 hour

---

## What You'll Do

1. Install Ansible on control machine
2. Set up inventory of your Raspberry Pis
3. Configure SSH access
4. Test Ansible connectivity
5. Run baseline configuration playbooks

---

## Prerequisites

- At least one Raspberry Pi running Debian (from Phase 1)
- SSH access to your Pi(s)
- Control machine with Python 3.8+

---

## Step 1: Install Ansible

### Linux (Ubuntu/Debian)

```bash
sudo apt update
sudo apt install -y ansible
```

### macOS

```bash
# Using Homebrew
brew install ansible

# Or using pip
pip install ansible
```

### Windows (WSL2)

```bash
# In WSL2 terminal
sudo apt update
sudo apt install -y ansible
```

### Verify Installation

```bash
ansible --version
# Should show: ansible [version] ... python [version]
```

---

## Step 2: Create Ansible Project Structure

Create a directory for your infrastructure code:

```bash
# Create project directory
mkdir -p ~/projects/rpi-cluster
cd ~/projects/rpi-cluster

# Create subdirectories
mkdir -p {inventory,playbooks,roles/common,roles/k3s,roles/monitoring,roles/cloudflare}
mkdir -p group_vars host_vars

# Initialize git (optional but recommended)
git init
echo "__pycache__/" > .gitignore
echo "*.pyc" >> .gitignore
echo "group_vars/*/secrets.yml" >> .gitignore
```

---

## Step 3: Create Inventory File

Create `inventory/hosts.ini`:

```ini
# Raspberry Pi Cluster Inventory

[all:vars]
ansible_user=debian
ansible_ssh_private_key_file=~/.ssh/id_ed25519
ansible_python_interpreter=/usr/bin/python3

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
```

**Adjust:**
- IP addresses to match your Pis
- `ansible_user` if you use a different username
- SSH key path if needed
- Add/remove worker nodes as needed

---

## Step 4: Test Ansible Connectivity

```bash
# Test connectivity to all hosts
ansible all -i inventory/hosts.ini -m ping

# Should output:
# rpi-master | SUCCESS => {
#     "changed": false,
#     "ping": "pong"
# }
```

If you get connection errors:
- Check IP addresses in inventory
- Verify SSH access: `ssh debian@<ip>`
- Ensure SSH key is accessible

---

## Step 5: Create Basic Playbooks

### 5.1 Common Configuration Playbook

Create `playbooks/01-common.yml`:

```yaml
---
- name: Common Configuration for All Raspberry Pis
  hosts: all
  become: yes
  gather_facts: yes

  tasks:
    - name: Update package lists
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Upgrade system packages
      apt:
        upgrade: dist
        autoremove: yes
        autoclean: yes

    - name: Install essential packages
      apt:
        name:
          - curl
          - wget
          - git
          - htop
          - net-tools
          - vim
          - nano
          - openssh-server
          - openssh-client
          - python3-pip
          - python3-dev
          - build-essential
        state: present

    - name: Configure SSH key authentication (disable password)
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#PasswordAuthentication yes'
        line: 'PasswordAuthentication no'
      notify: Restart SSH

    - name: Enable SSH
      service:
        name: ssh
        state: started
        enabled: yes

    - name: Configure sudo without password for debian user
      lineinfile:
        path: /etc/sudoers
        line: 'debian ALL=(ALL) NOPASSWD:ALL'
        validate: '/usr/sbin/visudo -cf %s'

    - name: Set timezone
      timezone:
        name: UTC
      # Change to: America/New_York, Europe/London, etc.

    - name: Configure UFW firewall
      apt:
        name: ufw
        state: present
      when: ansible_os_family == "Debian"

    - name: Enable UFW
      ufw:
        state: enabled
        default: deny
        direction: incoming

    - name: Allow SSH through firewall
      ufw:
        rule: allow
        port: '22'
        proto: tcp

    - name: Create system monitoring directory
      file:
        path: /var/log/rpi-cluster
        state: directory
        mode: '0755'

  handlers:
    - name: Restart SSH
      service:
        name: ssh
        state: restarted
```

### 5.2 K3s Installation Playbook

Create `playbooks/02-k3s.yml`:

```yaml
---
- name: Install and Configure K3s
  hosts: k3s_cluster
  become: yes

  vars:
    k3s_version: "latest"
    k3s_release_url: "https://github.com/k3s-io/k3s/releases/download"

  tasks:
    - name: Verify cgroups are enabled
      shell: cat /proc/cmdline | grep cgroup
      register: cgroup_check
      failed_when: cgroup_check.rc != 0

    - name: Install K3s dependencies
      apt:
        name:
          - curl
          - wget
          - apparmor
          - apparmor-utils
          - conntrack
          - ebtables
          - ethtool
          - iptables
          - kmod
          - libnetfilter-conntrack-3
          - libnetfilter-queue-1
          - util-linux
        state: present

    - name: Check if K3s is already installed
      stat:
        path: /usr/local/bin/k3s
      register: k3s_binary

    - name: Download and install K3s on master
      shell: |
        curl -sfL https://get.k3s.io | \
        INSTALL_K3S_EXEC="--disable traefik --disable servicelb" \
        sh -
      when:
        - not k3s_binary.stat.exists
        - "'masters' in group_names"

    - name: Wait for K3s to be ready on master
      shell: /usr/local/bin/k3s kubectl get nodes
      retries: 30
      delay: 5
      register: k3s_ready
      when: "'masters' in group_names"
      changed_when: false

    - name: Get K3s token from master
      shell: cat /var/lib/rancher/k3s/server/node-token
      register: k3s_token
      changed_when: false
      when: "'masters' in group_names"

    - name: Set K3s token fact
      set_fact:
        k3s_server_token: "{{ hostvars['rpi-master']['k3s_token'].stdout }}"

    - name: Get K3s server URL
      set_fact:
        k3s_server_url: "https://192.168.1.100:6443"
      # Update IP to match your master node

    - name: Install K3s on worker nodes
      shell: |
        curl -sfL https://get.k3s.io | \
        K3S_URL="{{ k3s_server_url }}" \
        K3S_TOKEN="{{ k3s_server_token }}" \
        sh -
      when:
        - not k3s_binary.stat.exists
        - "'workers' in group_names"

    - name: Allow K3s API port through firewall
      ufw:
        rule: allow
        port: '6443'
        proto: tcp
      when: "'masters' in group_names"

    - name: Allow node ports through firewall
      ufw:
        rule: allow
        port: '10250'
        proto: tcp

    - name: Copy kubeconfig to control machine (optional)
      fetch:
        src: /etc/rancher/k3s/k3s.yaml
        dest: ~/.kube/config-rpi
        flat: yes
      when: "'masters' in group_names"

  handlers:
    - name: Restart K3s
      service:
        name: k3s
        state: restarted
      when: "'masters' in group_names"
```

---

## Step 6: Run Playbooks

### First, test with dry-run

```bash
# Check syntax
ansible-playbook -i inventory/hosts.ini playbooks/01-common.yml --syntax-check

# Dry run (no changes made)
ansible-playbook -i inventory/hosts.ini playbooks/01-common.yml -C
```

### If dry-run looks good, run for real

```bash
# Run common configuration
ansible-playbook -i inventory/hosts.ini playbooks/01-common.yml -v

# Then run K3s installation
ansible-playbook -i inventory/hosts.ini playbooks/02-k3s.yml -v
```

**This will take 5-15 minutes on first run**

---

## Step 7: Verify K3s Cluster

Once K3s is installed:

```bash
# SSH into master node
ssh debian@192.168.1.100

# Check cluster nodes
sudo k3s kubectl get nodes

# Should show:
# NAME            STATUS   ROLES                  AGE
# rpi-master      Ready    control-plane,master   1m
# rpi-worker-1    Ready    <none>                 1m

# Check all pods
sudo k3s kubectl get pods -A

# Watch cluster status
watch sudo k3s kubectl get pods -A
```

Or from your control machine (if kubeconfig was copied):

```bash
# Set KUBECONFIG
export KUBECONFIG=~/.kube/config-rpi

# Get cluster info
kubectl cluster-info
kubectl get nodes
```

---

## Step 8: Create Ansible Configuration File

Create `ansible.cfg` in your project root:

```ini
[defaults]
inventory = inventory/hosts.ini
remote_user = debian
private_key_file = ~/.ssh/id_ed25519
host_key_checking = False
retry_files_enabled = False
log_path = ansible.log

[privilege_escalation]
become = True
become_method = sudo
become_user = root
```

Now you can run playbooks without specifying inventory:

```bash
# Shorter syntax
ansible-playbook playbooks/01-common.yml
ansible-playbook playbooks/02-k3s.yml
```

---

## Common Ansible Commands

```bash
# Ping all hosts
ansible all -m ping

# Run command on all hosts
ansible all -m shell -a 'uptime'

# Get facts about hosts
ansible all -m setup

# Run playbook
ansible-playbook playbooks/01-common.yml

# Run with verbose output
ansible-playbook playbooks/01-common.yml -v

# Run on specific hosts
ansible-playbook playbooks/01-common.yml --limit rpi-master

# List all hosts
ansible all --list-hosts
```

---

## Scaling to More Nodes

To add a new Raspberry Pi:

1. Install Debian (Phase 1)
2. Add to inventory: `inventory/hosts.ini`
3. Run playbooks:
   ```bash
   ansible-playbook playbooks/01-common.yml --limit new-node
   ansible-playbook playbooks/02-k3s.yml --limit new-node
   ```

That's it! The new node automatically joins the cluster.

---

## Troubleshooting

### Connection refused
```bash
# Verify SSH works
ssh debian@<ip> "echo success"

# Check key permissions
ls -la ~/.ssh/id_ed25519
# Should be: -rw------- (600)
```

### K3s installation fails
```bash
# Check manually
ssh debian@<ip>
sudo bash -c 'curl -sfL https://get.k3s.io | sh -'

# View logs
sudo journalctl -u k3s -n 50
```

### Node not joining cluster
```bash
# Check K3s service
sudo systemctl status k3s-agent

# View logs
sudo journalctl -u k3s-agent -n 50
```

---

## Next Steps

Once Ansible and K3s are working:

1. **03_K3S_DEPLOYMENT.md** - Advanced K3s configuration
2. **04_PROMETHEUS_GRAFANA.md** - Set up monitoring
3. **05_CLOUDFLARE_AGENT.md** - Configure networking

---

**Status:** Ansible and K3s cluster ready for applications
**Next:** Monitoring setup with Prometheus and Grafana
