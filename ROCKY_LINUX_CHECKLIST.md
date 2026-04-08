# Rocky Linux Deployment Checklist

## 📋 Pre-Deployment

### Hardware Preparation
- [ ] Raspberry Pi 3B+, 4, 400, 5, or 500 available
- [ ] microSD card(s) 64GB+ per Pi
- [ ] USB-C power supplies (2.5A+ each)
- [ ] Ethernet cables for stability
- [ ] Card reader for flashing
- [ ] Control machine (Linux/Mac/WSL2)

### Software Preparation
- [ ] Balena Etcher installed
- [ ] Python 3.8+ installed
- [ ] Ansible 2.10+ installed
- [ ] kubectl installed
- [ ] SSH key pair generated: `ssh-keygen -t ed25519`

### Network Planning
- [ ] IP addresses planned (e.g., 192.168.1.100+)
- [ ] Static IP configuration ready
- [ ] Cloudflare account (free)
- [ ] Domain name (free or purchased)

---

## 🔴 Phase 1: OS Installation (Rocky Linux)

### Download & Verify
- [ ] Downloaded Rocky Linux 9 ARM64 image from: https://rockylinux.org/download/
- [ ] File extracted (gunzip or xz -d)
- [ ] File size verified (~400-600MB)
- [ ] Checksum verified (if available)

### Flash to SD Cards
- [ ] Opened Balena Etcher
- [ ] Selected Rocky Linux image
- [ ] Selected correct microSD card
- [ ] Flashed successfully (5-10 minutes each)
- [ ] Ejected microSD card

### Initial Boot
- [ ] Inserted SD card into Raspberry Pi
- [ ] Connected power supply
- [ ] Connected Ethernet cable
- [ ] Waited 1-2 minutes for first boot

### SSH Access
- [ ] Found Pi IP address (router or nmap)
- [ ] SSH access working: `ssh rocky@<ip>`
- [ ] (or root if available on first boot)

---

## 🟡 Phase 2: System Configuration (dnf & firewalld)

### On Each Raspberry Pi

**Hostname Configuration**
- [ ] Changed hostname: `sudo hostnamectl set-hostname rpi-master`
- [ ] Updated /etc/hosts
- [ ] Rebooted to apply: `sudo reboot`

**Package Manager (dnf)**
- [ ] Updated packages: `sudo dnf update -y`
- [ ] Verified dnf working: `sudo dnf list installed`

**Firewall (firewalld)**
- [ ] Verified firewalld enabled: `sudo systemctl status firewalld`
- [ ] Checked current rules: `sudo firewall-cmd --list-all`
- [ ] Opened SSH: `sudo firewall-cmd --permanent --add-service=ssh`
- [ ] Reloaded rules: `sudo firewall-cmd --reload`

**Static IP Configuration**
- [ ] Configured static IP with nmcli or editing config files
- [ ] Verified with: `ip addr`
- [ ] Tested ping: `ping 8.8.8.8`

**User Setup**
- [ ] Created rocky user: `sudo useradd -m -G wheel -s /bin/bash rocky`
- [ ] Set password: `sudo passwd rocky`
- [ ] (or verified rocky user exists)

**SSH Key Configuration**
- [ ] Copied SSH public key: `ssh-copy-id -i ~/.ssh/id_ed25519.pub rocky@<pi-ip>`
- [ ] Tested passwordless login: `ssh rocky@<ip>`

---

## 🟢 Phase 3: Ansible Automation

### Control Machine Setup

**Ansible Inventory**
- [ ] Created inventory/hosts.ini with correct IPs
- [ ] Updated ansible_user to: `rocky`
- [ ] Updated ansible_become to: `True`
- [ ] Added ansible_become_method: `sudo`

**Ansible Configuration**
- [ ] Created ansible.cfg with correct settings
- [ ] Set remote_user to: `rocky`
- [ ] Verified inventory: `ansible all --list-hosts`

**Connectivity Test**
- [ ] Ping all hosts: `ansible all -m ping`
- [ ] All hosts returned: `SUCCESS`

**Common Configuration Playbook**
- [ ] Reviewed playbooks/01-common.yml
- [ ] Verified dnf tasks (not apt)
- [ ] Verified firewalld tasks (not ufw)
- [ ] Ran syntax check: `ansible-playbook playbooks/01-common.yml --syntax-check`
- [ ] Ran playbook: `ansible-playbook playbooks/01-common.yml`
- [ ] All tasks completed successfully

**K3s Installation Playbook**
- [ ] Reviewed playbooks/02-k3s.yml
- [ ] Verified firewalld rules (6443, 10250, 10255)
- [ ] Ran on master first: `ansible-playbook playbooks/02-k3s.yml --limit rpi-master`
- [ ] Waited for K3s to start (~30 seconds)
- [ ] Ran on workers: `ansible-playbook playbooks/02-k3s.yml --limit workers`

---

## 🔵 Phase 4: Kubernetes Cluster

### Cluster Verification
- [ ] SSH to master: `ssh rocky@192.168.1.100`
- [ ] Checked cluster status: `sudo k3s kubectl get nodes`
- [ ] All nodes show: `Ready`
- [ ] Checked pods: `sudo k3s kubectl get pods -A`
- [ ] System pods running

### Kubeconfig Setup
- [ ] Copied kubeconfig to control machine
- [ ] Set KUBECONFIG: `export KUBECONFIG=~/.kube/config-rpi`
- [ ] Verified kubectl access: `kubectl get nodes`
- [ ] Verified cluster info: `kubectl cluster-info`

### Test Deployment
- [ ] Deployed test app: `kubectl apply -f k3s-manifests/sample-app.yml`
- [ ] Verified pods: `kubectl get pods -A`
- [ ] Checked service: `kubectl get svc`
- [ ] Tested connectivity: `curl http://app-service`

### Firewall Verification
- [ ] SSH to node: `ssh rocky@<pi-ip>`
- [ ] Listed firewall rules: `sudo firewall-cmd --list-all`
- [ ] Verified K3s ports open: Look for 6443, 10250 in output

---

## 🟣 Phase 5: Monitoring

### Prometheus & Grafana
- [ ] Added Prometheus repo: `helm repo add prometheus-community ...`
- [ ] Installed Prometheus: `helm install prometheus ...`
- [ ] Installed Grafana: `helm install grafana ...`
- [ ] Waited for pods: `kubectl get pods -n monitoring`

### Grafana Access
- [ ] Port forwarded: `kubectl port-forward svc/grafana 3000:80 -n monitoring`
- [ ] Accessed Grafana: http://localhost:3000
- [ ] Logged in: admin / (password from values)
- [ ] Verified Prometheus datasource working

### Dashboards
- [ ] Imported K3s dashboard
- [ ] Imported Node Exporter dashboard
- [ ] Created custom dashboard
- [ ] Verified metrics displayed

---

## 🟡 Phase 6: Networking & DNS

### Cloudflare Setup
- [ ] Created Cloudflare account (free)
- [ ] Added domain to Cloudflare
- [ ] Updated nameservers at registrar
- [ ] Waited for DNS propagation (24-48h)

### Cloudflare Agent
- [ ] Created tunnel in Cloudflare dashboard
- [ ] Got tunnel token
- [ ] Created secret in K3s: `kubectl create secret generic cloudflare-tunnel ...`
- [ ] Deployed agent: `kubectl apply -f k3s-manifests/cloudflare-agent.yml`
- [ ] Verified connection: `kubectl logs -f deployment/cloudflare-agent -n cloudflare`

### DNS & SSL
- [ ] Added DNS records in Cloudflare
- [ ] Set Flexible SSL in Cloudflare dashboard
- [ ] Tested DNS: `nslookup example.com`
- [ ] Tested HTTPS: `curl https://example.com`

---

## 🟢 Phase 7: Backups & Disaster Recovery

### Backup Setup
- [ ] Created backup directory: `mkdir -p ~/backups`
- [ ] Downloaded backup scripts
- [ ] Made scripts executable: `chmod +x scripts/*.sh`
- [ ] Ran backup test: `./scripts/backup-all.sh`

### Automated Backups
- [ ] Added cron entry: `crontab -e`
- [ ] Set daily backup: `0 2 * * * ~/backup-all.sh`
- [ ] Verified cron: `crontab -l`

### Backup Verification
- [ ] Checked backup files: `ls -lh ~/backups/`
- [ ] Verified backup size reasonable
- [ ] Verified timestamp current
- [ ] Tested extraction: `tar -tzf backup.tar.gz | head`

### DR Testing
- [ ] Reviewed 07_BACKUP_RECOVERY.md
- [ ] Tested etcd restore procedure
- [ ] Tested resource restore procedure
- [ ] Documented results

---

## ✅ Post-Deployment Verification

### Everything Working?
- [ ] `kubectl get nodes` shows all nodes Ready
- [ ] `kubectl get pods -A` shows all pods Running
- [ ] Grafana dashboards showing metrics
- [ ] Domain resolves through Cloudflare
- [ ] HTTPS working (no certificate warnings)
- [ ] Backups running automatically
- [ ] SELinux in permissive mode (if needed)

### Performance Check
- [ ] `kubectl top nodes` shows reasonable CPU/Memory
- [ ] `kubectl top pods` shows pod usage
- [ ] No high CPU or memory spikes
- [ ] Network connectivity stable

### Security Check
- [ ] SSH key-based auth working (not password)
- [ ] firewalld rules applied correctly
- [ ] SELinux status understood
- [ ] Firewall allowing K3s traffic
- [ ] DDoS protection enabled (Cloudflare)

---

## 📋 Optional: Scaling & High Availability

### Add More Nodes
- [ ] Flashed Rocky Linux on new microSD cards
- [ ] Configured hostnames and IPs
- [ ] Updated Ansible inventory
- [ ] Ran playbooks on new nodes
- [ ] Verified nodes joined cluster

### High Availability (3+ Masters)
- [ ] Installed keepalived on masters
- [ ] Configured virtual IP (192.168.1.50)
- [ ] Updated kubeconfig to use VIP
- [ ] Tested failover
- [ ] Verified HA working

---

## 🐛 Troubleshooting Section

### If K3s Won't Start
```bash
# Check SELinux
sudo getenforce  # If enforcing, try:
sudo setenforce 0

# Check firewall
sudo firewall-cmd --list-all | grep 6443

# Check logs
sudo journalctl -u k3s -n 50
```

### If Pods Won't Schedule
```bash
# Check node resources
kubectl describe nodes
kubectl top nodes

# Check resource quotas
kubectl get resourcequota -A
```

### If Firewall Issues
```bash
# List all rules
sudo firewall-cmd --list-all

# Add missing port
sudo firewall-cmd --permanent --add-port=XXXX/tcp
sudo firewall-cmd --reload
```

### If Ansible Fails
```bash
# Check syntax
ansible-playbook playbooks/01-common.yml --syntax-check

# Run verbose
ansible-playbook playbooks/01-common.yml -vv

# Check ansible user
ansible all -m debug -a "var=ansible_user"
```

---

## 📊 Success Metrics

| Component | Expected | Status |
|-----------|----------|--------|
| Nodes | All Ready | ✓ |
| Pods | All Running | ✓ |
| Metrics | Showing data | ✓ |
| DNS | Resolving | ✓ |
| HTTPS | Working | ✓ |
| Backups | Daily | ✓ |
| Firewall | Blocking non-essential | ✓ |

---

## 📞 Quick Support Links

### Rocky Linux
- Docs: https://docs.rockylinux.org/
- Forum: https://forums.rockylinux.org/

### K3s
- Docs: https://docs.k3s.io
- GitHub: https://github.com/k3s-io/k3s

### This Project
- Guide: ROCKY_LINUX_MIGRATION_GUIDE.md
- Quick Ref: QUICK_REFERENCE.md
- Updates: UPDATES_SUMMARY.md

---

## ✨ Final Notes

✅ **You've successfully deployed a production-ready Rocky Linux K3s cluster!**

### What you have:
- Enterprise-ready OS (Rocky Linux 9)
- Kubernetes orchestration (K3s)
- Metrics collection (Prometheus)
- Dashboards (Grafana)
- DDoS protection (Cloudflare)
- Automated backups
- Disaster recovery plan
- Automated scaling capability

### Next Steps:
1. Deploy your applications
2. Monitor cluster health
3. Test disaster recovery monthly
4. Update systems regularly
5. Expand cluster as needed

---

**Status:** Deployment Complete ✅
**Date:** 2026-04-07
**OS:** Rocky Linux 9 ARM64
**Cluster:** Ready for Production

🎉 **Happy Clustering!** 🚀
