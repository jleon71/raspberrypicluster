# QNAP Virtualization + Ansible - Quick Start Guide

**Total Setup Time**: 3-4 hours | **Experience Level**: Intermediate

---

## 🎯 What You'll Build

```
QNAP TS-473a
    ↓
    ├─ Virtualization Station (Enable)
    │  ├─ Rocky Linux VM (Create)
    │  └─ Ansible installed (Install)
    │
    ├─ Shared Storage (Mount in VM)
    │  └─ Ansible Project & Playbooks
    │
    └─ Raspberry Pi Cluster (Manage via Ansible)
       ├─ K3s Master (192.168.1.100)
       ├─ K3s Worker-1 (192.168.1.101)
       ├─ K3s Worker-2 (192.168.1.102)
       └─ ... (Unlimited scalability)
```

---

## 📋 Before You Start

**Hardware Checklist:**
- [ ] QNAP TS-473a with 100GB+ free space
- [ ] 2+ Raspberry Pis with Rocky Linux
- [ ] Network connectivity between all devices
- [ ] QNAP admin credentials
- [ ] SSH client on your computer

**Software Checklist:**
- [ ] Virtualization Station available on QNAP
- [ ] Rocky Linux ISO downloaded (7-8GB)
- [ ] SSH keys accessible on your computer

---

## ⚡ 8 Quick Phases (90 Minutes Each)

### Phase 1: Enable Virtualization (30 min)

```
QNAP Web → App Center → Search "Virtualization Station"
Click Install → Wait → Click Open
```

**Create shared storage:**
```
Control Panel → Shared Folders → Create
Name: "Ansible"
Keep note of path: /share/Ansible
```

**Have Rocky Linux ISO ready:**
- Download from: https://rockylinux.org/download/
- Or let Virtualization Station download it

---

### Phase 2: Create & Install VM (60 min)

**Create VM:**
```
Virtualization Station → Virtual Machines → Create
Name: Ansible-Control
vCPU: 4 cores
RAM: 8GB
Disk: 50GB
Network: Bridged
```

**Install Rocky Linux:**
- Start VM
- Boot from ISO
- Follow installation (15 minutes)
- Set hostname: "ansible-control"
- Create user: "ansible"
- Reboot when done

**Get VM IP:**
```bash
# In VM console
ip addr show
# Note: 192.168.1.xxx
```

---

### Phase 3: Basic VM Setup (30 min)

**SSH into VM:**
```bash
ssh ansible@<vm-ip>
```

**Update system:**
```bash
sudo dnf update -y
sudo dnf install -y python3-pip python3-devel gcc git openssh-server openssh-clients
```

**Set static IP (optional but recommended):**
```bash
sudo nano /etc/sysconfig/network-scripts/ifcfg-eth0
# Change to: BOOTPROTO=static, IPADDR=192.168.1.200
# Save and restart: sudo systemctl restart network
```

---

### Phase 4: Install Ansible (20 min)

**Install Ansible:**
```bash
sudo pip3 install ansible ansible-core
ansible --version  # Verify
ansible-galaxy collection install community.general
```

**Generate SSH keys:**
```bash
sudo su - ansible
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
cat ~/.ssh/id_ed25519.pub  # Copy this output
```

**Distribute to cluster nodes:**
```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub rocky@192.168.1.100
ssh-copy-id -i ~/.ssh/id_ed25519.pub rocky@192.168.1.101
```

---

### Phase 5: Mount Storage & Setup Project (30 min)

**Create mount directory:**
```bash
sudo mkdir -p /mnt/ansible-data
```

**Mount QNAP share (CIFS):**
```bash
sudo nano /etc/fstab
# Add: //192.168.1.1/Ansible /mnt/ansible-data cifs username=admin,password=QNAP_PASS 0 0
sudo mount -a
df -h  # Verify
```

**Create Ansible project:**
```bash
cd /mnt/ansible-data
mkdir rpi-cluster && cd rpi-cluster
mkdir -p {inventory,playbooks,group_vars,logs,backup}
```

**Create ansible.cfg:**
```bash
cat > ansible.cfg << 'EOF'
[defaults]
inventory = inventory/hosts.ini
remote_user = rocky
private_key_file = ~/.ssh/id_ed25519
host_key_checking = False
timeout = 30
log_path = ./ansible.log

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False
EOF
```

---

### Phase 6: Create Inventory & Playbooks (30 min)

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
rpi-master ansible_host=192.168.1.100

[workers]
rpi-worker-1 ansible_host=192.168.1.101

[k3s_cluster:children]
masters
workers
EOF
```

**Create playbooks/00-ping.yml:**
```bash
cat > playbooks/00-ping.yml << 'EOF'
---
- name: Test Connectivity
  hosts: all
  gather_facts: no
  tasks:
    - name: Ping
      ping:
EOF
```

**Create playbooks/01-common.yml:** (See full documentation for complete content)

**Create playbooks/02-k3s.yml:** (See full documentation for complete content)

---

### Phase 7: Deploy Cluster (45 min)

**Test connectivity:**
```bash
ansible all --list-hosts
ansible all -m ping  # All should show SUCCESS
```

**Run playbooks:**
```bash
# System configuration (15 min)
ansible-playbook playbooks/01-common.yml -v

# K3s installation (20 min)
ansible-playbook playbooks/02-k3s.yml -v
```

**Verify:**
```bash
/usr/local/bin/k3s kubectl get nodes
# Should show: rpi-master Ready and rpi-worker-1 Ready
```

---

### Phase 8: Setup Automation (15 min)

**Create status script:**
```bash
cat > status.sh << 'EOF'
#!/bin/bash
echo "=== Status ==="
ansible all --list-hosts
/usr/local/bin/k3s kubectl get nodes 2>/dev/null || echo "K3s not ready"
echo "Done"
EOF
chmod +x status.sh
```

**Setup cron jobs:**
```bash
crontab -e
# Add:
0 * * * * cd /mnt/ansible-data/rpi-cluster && ansible all -m ping >> logs/hourly.log 2>&1
0 2 * * * cd /mnt/ansible-data/rpi-cluster && ansible-playbook playbooks/01-common.yml >> logs/daily.log 2>&1
0 3 * * * tar -czf backup/ansible-$(date +\%Y\%m\%d).tar.gz /mnt/ansible-data/rpi-cluster
```

**Enable VM backup in Virtualization Station:**
```
Select Ansible-Control VM → Right-click → Backup
Set: Daily at 4:00 AM, Keep 7 backups
```

---

## 📊 Essential Commands

```bash
# Navigate to project
cd /mnt/ansible-data/rpi-cluster

# Test connectivity
ansible all -m ping

# Run a playbook
ansible-playbook playbooks/01-common.yml -v

# Check K3s cluster
/usr/local/bin/k3s kubectl get nodes

# View logs
tail -f logs/ansible.log

# Check storage
df -h /mnt/ansible-data

# SSH to a node
ssh -i ~/.ssh/id_ed25519 rocky@192.168.1.100

# Get status
./status.sh

# View cron jobs
crontab -l
```

---

## ✅ Success Checklist

- [ ] Virtualization Station enabled
- [ ] VM created and running (Ansible-Control)
- [ ] Rocky Linux installed on VM
- [ ] Ansible installed: `ansible --version`
- [ ] Storage mounted: `df -h /mnt/ansible-data`
- [ ] SSH keys generated: `ls ~/.ssh/`
- [ ] SSH keys distributed to all nodes
- [ ] `ansible all -m ping` shows all SUCCESS
- [ ] Playbooks created in /mnt/ansible-data/rpi-cluster
- [ ] K3s master running
- [ ] K3s workers joined
- [ ] All nodes show Ready: `kubectl get nodes`
- [ ] Status script works: `./status.sh`
- [ ] Cron jobs configured: `crontab -l`
- [ ] VM backup scheduled in Virtualization Station

---

## 🐛 Quick Troubleshooting

| Problem | Solution |
|---------|----------|
| SSH fails | Check keys: `ls -la ~/.ssh/` |
| Ping fails | Verify IPs: `ping 192.168.1.100` |
| Playbook errors | Run with verbose: `ansible-playbook playbooks/01-common.yml -vvv` |
| K3s not ready | Check firewall: `sudo firewall-cmd --list-all` |
| Storage not mounted | Try: `sudo mount -a` |
| Inventory issues | Test: `ansible-inventory --list` |

---

## 📁 File Locations

**On VM (in QNAP):**
```
/mnt/ansible-data/rpi-cluster/
├── inventory/hosts.ini         ← Edit for your nodes
├── playbooks/                  ← Your Ansible playbooks
├── logs/                       ← Check for issues
├── backup/                     ← Automated backups
└── ansible.cfg                 ← Ansible configuration
```

**On QNAP:**
```
/share/Ansible/                 ← Shared storage
└── rpi-cluster/               ← All Ansible data
```

**On VM (system):**
```
~/.ssh/                         ← SSH keys
/var/lib/rancher/k3s/          ← K3s data
/etc/rancher/k3s/              ← K3s config
```

---

## 🔧 VM Resources (Can be adjusted later)

```
vCPU:  4 cores      (shared from QNAP)
RAM:   8 GB         (shared from QNAP)
Disk:  50 GB        (virtual disk)
IP:    192.168.1.200 (static, optional)
```

To adjust:
1. Stop VM in Virtualization Station
2. Right-click → Edit
3. Change vCPU or RAM
4. Start VM

---

## 📖 Full Documentation

- **QNAP_VIRTUALIZATION_SUMMARY.md** - Complete overview
- **02_ANSIBLE_SETUP_QNAP_VIRTUALIZATION.md** - Detailed technical guide
- **QNAP_VIRTUALIZATION_DEPLOYMENT.md** - Step-by-step deployment
- **QNAP_SCALING_AUTOMATION.md** - Adding nodes and advanced ops

---

## ⏭️ Next Steps After Deployment

1. **Monitor** the cluster for 1 week
2. **Review** logs: `ls -la logs/`
3. **Test** backup restore (monthly)
4. **Add more workers** when ready (see QNAP_SCALING_AUTOMATION.md)
5. **Deploy applications** to K3s
6. **Setup monitoring** (Prometheus/Grafana)
7. **Configure networking** (Cloudflare Agent)

---

## 💡 Tips for Success

- ✅ Read full documentation before starting
- ✅ Take notes of all IPs and passwords
- ✅ Test SSH before running Ansible
- ✅ Use `--syntax-check` on new playbooks
- ✅ Use `-C` (check mode) before running for real
- ✅ Review logs after each playbook run
- ✅ Keep backups of QNAP and VM
- ✅ Document any changes you make

---

## 🎯 Key Differences From Native QNAP Approach

| Aspect | Virtualization | Native QNAP |
|--------|---|---|
| QNAP OS Impact | None (isolated VM) | Direct (modified) |
| Easy Recovery | Yes (VM backup) | Complex |
| Resource Isolation | Yes | Shared with QNAP |
| Backup/Restore | Minutes | Hours |
| Professional | ✅ Yes | Not standard |
| Security | Better isolation | Less isolated |
| Scalability | Easy (adjust VM) | Limited |

---

## 🚀 You're Ready!

This setup provides:
- **Enterprise-grade** infrastructure
- **Automatic** daily backups
- **Scalable** to hundreds of nodes
- **Professional** virtualization standard
- **24/7** automation capability

---

## 📞 Need Help?

1. **Check logs**: `tail -f logs/ansible.log`
2. **Run verbose**: `ansible-playbook playbooks/01-common.yml -vvv`
3. **Test directly**: `ssh -v rocky@192.168.1.100`
4. **Read full guide**: See 02_ANSIBLE_SETUP_QNAP_VIRTUALIZATION.md

---

**Ready to begin? Follow the 8 phases above, or see QNAP_VIRTUALIZATION_DEPLOYMENT.md for more details.**

**Good luck! 🚀**

---

**Created**: April 9, 2026
**Status**: Complete and ready for immediate deployment
**Estimated Time to Operational Cluster**: 3-4 hours
