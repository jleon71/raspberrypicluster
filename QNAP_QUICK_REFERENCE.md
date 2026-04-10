# QNAP TS-473a Ansible Control Machine - Quick Reference

**Last Updated**: April 9, 2026
**Status**: Ready for Deployment
**Estimated Setup Time**: 2-3 hours

---

## 📍 Start Here

```
1. Read QNAP_SUMMARY.md (15 min) - Understand what was created
2. Follow QNAP_DEPLOYMENT_GUIDE.md (90 min) - Deploy the cluster
3. Use QNAP_SCALING_AUTOMATION.md (future) - Scale when ready
```

---

## 🎯 Pre-Deployment Checklist

- [ ] QNAP TS-473a running Rocky Linux
- [ ] Raspberry Pis running Rocky Linux with SSH enabled
- [ ] IP addresses assigned (master: .100, workers: .101+)
- [ ] Network connectivity verified between all devices
- [ ] Admin access to QNAP confirmed
- [ ] 2GB+ free space on QNAP verified

---

## ⚡ 90-Minute Quick Deployment

### Phase 1-3: Setup (50 minutes)
```bash
# SSH to QNAP
ssh rocky@<qnap-ip>

# Create storage
sudo mkdir -p /mnt/ansible-data/{inventory,playbooks,roles/{common,k3s,monitoring,cloudflare},group_vars,host_vars,logs,backup}
sudo chown -R rocky:rocky /mnt/ansible-data
chmod -R 755 /mnt/ansible-data

# Install Ansible
sudo dnf update -y
sudo dnf install -y python3-pip python3-devel gcc git
pip3 install --user ansible

# Generate SSH keys
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
chmod 600 ~/.ssh/id_ed25519

# Distribute to all nodes
ssh-copy-id -i ~/.ssh/id_ed25519.pub rocky@192.168.1.100
ssh-copy-id -i ~/.ssh/id_ed25519.pub rocky@192.168.1.101
```

### Phase 4-5: Configure Ansible (20 minutes)
```bash
# Create project structure
cd /mnt/ansible-data
git init rpi-cluster
cd rpi-cluster
mkdir -p {inventory,playbooks,roles/{common,k3s,monitoring,cloudflare},group_vars,host_vars,logs,backup}

# Copy inventory file
# (Use qnap-ansible-inventory-hosts.ini provided)

# Create ansible.cfg
# (See 02_ANSIBLE_SETUP_QNAP.md Step 6)

# Copy playbook files
# (See 02_ANSIBLE_SETUP_QNAP.md Steps 7-9)
```

### Phase 6-8: Deploy (30-45 minutes)
```bash
cd /mnt/ansible-data/rpi-cluster

# Test connectivity
ansible all -m ping

# Run playbooks
ansible-playbook playbooks/01-common.yml -v
ansible-playbook playbooks/02-k3s.yml -v

# Verify
/usr/local/bin/k3s kubectl get nodes
```

---

## 🔑 Essential Commands

```bash
# Navigation
cd /mnt/ansible-data/rpi-cluster

# Test connectivity
ansible all -m ping

# List all hosts
ansible all --list-hosts

# Run playbook
ansible-playbook playbooks/01-common.yml -v

# Check K3s cluster
/usr/local/bin/k3s kubectl get nodes
/usr/local/bin/k3s kubectl get pods -A

# View logs
tail -f ansible.log
tail -f /mnt/ansible-data/logs/*.log

# Check status
./status.sh  # (Create this: see docs)

# Add new node
ansible-playbook playbooks/01-common.yml --limit new-node-name
ansible-playbook playbooks/02-k3s.yml --limit new-node-name
```

---

## 📁 File Locations

```
/mnt/ansible-data/rpi-cluster/
├── inventory/hosts.ini         ← Update with your node IPs
├── ansible.cfg                 ← Main configuration
├── playbooks/
│   ├── 00-ping.yml
│   ├── 01-common.yml
│   └── 02-k3s.yml
├── group_vars/
│   ├── all.yml
│   ├── masters.yml
│   └── workers.yml
├── logs/                       ← Check for issues
└── backup/                     ← Backups go here
```

---

## 🔧 Configuration Customization

### Change Master IP
Edit `inventory/hosts.ini`:
```ini
[masters]
rpi-master ansible_host=YOUR.IP.HERE
```

### Change Worker IPs
```ini
[workers]
rpi-worker-1 ansible_host=YOUR.IP.HERE
rpi-worker-2 ansible_host=YOUR.IP.HERE
```

### Change Timezone
Edit `group_vars/all.yml`:
```yaml
system_timezone: "YOUR/TIMEZONE"
```

---

## 🐛 Common Issues

| Issue | Solution |
|-------|----------|
| SSH Connection Refused | Check SSH running: `ansible all -m systemd -a "name=sshd"` |
| K3s Install Fails | Check firewall: `ansible all -m firewalld -a "service=https state=enabled"` |
| Inventory Errors | Test: `ansible-inventory --list` |
| Playbook Won't Run | Check syntax: `ansible-playbook playbooks/01-common.yml --syntax-check` |
| Nodes Not Joining | Verify token: Check `/var/lib/rancher/k3s/server/node-token` on master |

---

## 📊 Post-Deployment Checklist

- [ ] All nodes show in `ansible all --list-hosts`
- [ ] All nodes respond to `ansible all -m ping`
- [ ] K3s master ready: `/usr/local/bin/k3s kubectl get nodes`
- [ ] All workers joined cluster
- [ ] SSH keys working without password
- [ ] Logs accessible: `/mnt/ansible-data/logs/`
- [ ] Backups directory ready: `/mnt/ansible-data/backup/`
- [ ] Status script working: `./status.sh`

---

## 🚀 Scaling (Add More Nodes)

```bash
# 1. Prepare new Raspberry Pi with Rocky Linux
# 2. Update inventory/hosts.ini
# 3. SSH keys
ssh-copy-id -i ~/.ssh/id_ed25519.pub rocky@NEW.IP

# 4. Run playbooks on new node
ansible-playbook playbooks/01-common.yml --limit new-node-name
ansible-playbook playbooks/02-k3s.yml --limit new-node-name

# 5. Verify
/usr/local/bin/k3s kubectl get nodes
```

---

## ⏰ Automation Setup (Optional)

```bash
# Install cron jobs
crontab /path/to/ansible-jobs.cron

# Verify installed
crontab -l

# Daily automation includes:
# - Hourly: Connectivity checks
# - Daily: Health checks, updates, backups
# - Weekly: Full maintenance
# - Monthly: Cleanup
```

See **QNAP_SCALING_AUTOMATION.md** for complete automation setup.

---

## 📖 Full Documentation

| Document | Purpose | When to Read |
|----------|---------|--------------|
| QNAP_SUMMARY.md | Overview of everything | First |
| QNAP_DEPLOYMENT_GUIDE.md | Step-by-step deployment | During setup |
| 02_ANSIBLE_SETUP_QNAP.md | Deep technical details | For troubleshooting |
| QNAP_SCALING_AUTOMATION.md | Advanced operations | When scaling |

---

## 🎓 Learning Resources

- **Ansible Docs**: https://docs.ansible.com/ansible/latest/
- **K3s Documentation**: https://docs.k3s.io/
- **Rocky Linux**: https://rockylinux.org/documentation/
- **Our Previous Setup**: 01_DEBIAN_INSTALLATION.md, 03_K3S_DEPLOYMENT.md

---

## 💾 Backup Strategy

**Automatic** (with cron setup):
- Daily configuration backups
- K3s etcd snapshots
- Playbook version control (git)

**Location**: `/mnt/ansible-data/backup/`

**Retention**: 30 days (configurable)

---

## 📞 When You Get Stuck

1. **Check logs**: `tail -f /mnt/ansible-data/logs/ansible.log`
2. **Run with verbose**: `ansible-playbook playbooks/01-common.yml -vvv`
3. **Verify connectivity**: `ansible all -m ping -vvv`
4. **Check specific host**: `ansible rpi-master -m setup | head`
5. **SSH directly**: `ssh -v rocky@IP-ADDRESS`

---

## ✅ Success Indicators

When deployment is complete, you'll have:

1. ✅ Ansible installed on QNAP
2. ✅ SSH keys distributed to all nodes
3. ✅ Inventory configured with all node IPs
4. ✅ All playbooks in place
5. ✅ K3s master running
6. ✅ K3s workers joined cluster
7. ✅ kubectl accessible from QNAP
8. ✅ Logs being collected
9. ✅ Backup directories ready
10. ✅ Ready to add more nodes

---

## 🎯 Next Steps After Deployment

1. **Monitor**: Check health daily
2. **Backup**: Verify backups are running
3. **Document**: Update any custom configs
4. **Plan**: When ready, add more nodes
5. **Deploy Apps**: Start running applications
6. **Monitor**: Set up Prometheus/Grafana

---

## 🔐 Security Notes

- SSH keys are Ed25519 (modern, secure)
- Password authentication disabled in SSH
- Firewall enabled on all nodes
- SELinux in permissive mode (can be strict later)
- Sudo passwordless for Ansible (secure in private network)

---

## 📋 Version Info

- Created: April 9, 2026
- Platform: QNAP TS-473a
- OS: Rocky Linux (all nodes)
- Ansible: 2.10+ (installed via pip)
- K3s: Latest (configured in playbooks)

---

**Ready to deploy? Start with QNAP_DEPLOYMENT_GUIDE.md**

Questions? Check the troubleshooting section in 02_ANSIBLE_SETUP_QNAP.md

Good luck! 🚀
