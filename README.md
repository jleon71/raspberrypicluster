# Raspberry Pi 4 K3s Cluster Infrastructure - Complete Guide (Rocky Linux)

## Quick Start

Welcome to your complete infrastructure automation framework! This guide provides everything you need to build, deploy, and manage a Raspberry Pi 4 cluster running Rocky Linux 9, K3s (Kubernetes), Prometheus, Grafana, and Cloudflare Agent - all **completely free**.

**Note:** This is the Rocky Linux version. For Debian version, see the legacy guides.

---

## 📋 What's Included

### Complete Documentation (7 Phases)

1. **00_INFRASTRUCTURE_OVERVIEW.md** - Architecture and cost analysis
2. **01_DEBIAN_INSTALLATION.md** (Legacy) / **START_HERE_ROCKY_LINUX.md** (Current) - OS setup on Raspberry Pi
3. **02_ANSIBLE_SETUP.md** or **02_ANSIBLE_SETUP_QNAP_VIRTUALIZATION.md** - Infrastructure automation
4. **03_K3S_DEPLOYMENT.md** - Kubernetes cluster configuration
5. **04_PROMETHEUS_GRAFANA.md** - Monitoring and visualization
6. **05_CLOUDFLARE_AGENT.md** - DNS, security, and edge connectivity
7. **06_SCALING_GUIDE.md** - Expand to 10+ nodes
8. **07_BACKUP_RECOVERY.md** - Disaster recovery procedures

### Bonus Resources

- **scripts/** - Automation scripts for backups and maintenance
- **ansible/** - Ready-to-use Ansible playbooks
- **k3s-manifests/** - Kubernetes deployment examples
- **helm-values/** - Helm chart configurations

---

## 🚀 Start Here

### For First-Time Setup (1-3 nodes)

Follow the guides in order:

```bash
# Phase 1: Get hardware ready
→ 01_DEBIAN_INSTALLATION.md (1-2 hours)

# Phase 2: Automate everything
→ 02_ANSIBLE_SETUP.md (45 min - 1 hour)

# Phase 3: Deploy Kubernetes
→ 03_K3S_DEPLOYMENT.md (1-2 hours)

# Phase 4: Add monitoring
→ 04_PROMETHEUS_GRAFANA.md (1-2 hours)

# Phase 5: Configure networking
→ 05_CLOUDFLARE_AGENT.md (45 min - 1 hour)

# Total: 5-9 hours for complete setup
```

### Quick Links

| Goal | Document | Time |
|------|----------|------|
| Set up 1 Pi (Rocky) | `01_DEBIAN_INSTALLATION.md` (updated for Rocky) | 1-2h |
| Automate config | `02_ANSIBLE_SETUP.md` (dnf & firewalld) | 1h |
| Deploy K3s | `03_K3S_DEPLOYMENT.md` | 1-2h |
| Add monitoring | `04_PROMETHEUS_GRAFANA.md` | 1-2h |
| Configure DNS | `05_CLOUDFLARE_AGENT.md` | 1h |
| Add more Pis | `06_SCALING_GUIDE.md` | 30m-1h per Pi |
| Protect data | `07_BACKUP_RECOVERY.md` | 45m |

---

## 🎯 Key Features

### Infrastructure Automation
- ✅ **Infrastructure as Code** - All configuration in Git
- ✅ **Ansible Playbooks** - Automated setup and configuration
- ✅ **Idempotent** - Safe to run multiple times
- ✅ **Scalable** - Grow from 1 to 100+ nodes

### High Availability (Optional)
- ✅ **Multi-master K3s** - Resilient control plane
- ✅ **Keepalived** - Virtual IP for failover
- ✅ **etcd snapshots** - Regular backups
- ✅ **Load balancing** - MetalLB for services

### Monitoring & Observability
- ✅ **Prometheus** - Metrics collection
- ✅ **Grafana** - Beautiful dashboards
- ✅ **Node Exporter** - Hardware metrics
- ✅ **Alert Rules** - Automated alerting

### Security & Networking
- ✅ **Cloudflare Agent** - DDoS protection
- ✅ **Automatic HTTPS** - Free SSL/TLS
- ✅ **DNS Management** - Cloudflare DNS
- ✅ **Firewall Rules** - WAF and rate limiting

### Data Protection
- ✅ **Automated Backups** - Daily etcd snapshots
- ✅ **Disaster Recovery** - Tested restore procedures
- ✅ **Encryption** - Encrypted backup storage
- ✅ **Retention Policies** - Automatic cleanup

---

## 📊 Architecture Diagram

```
┌─────────────────────────────────────────────────┐
│            Your Control Machine                 │
│   (Laptop/Desktop with Ansible)                │
└────────────┬────────────────────────────────────┘
             │ SSH & Ansible (dnf, firewalld)
    ┌────────┴──────────────────────┐
    │                               │
┌───▼──────────────────┐   ┌───────▼─────────────┐
│  Raspberry Pi 1      │   │  Raspberry Pi 2+    │
│  (Master/Worker)     │   │  (Workers)          │
│  - Rocky Linux 9     │   │  - Rocky Linux 9    │
│  - K3s Master        │   │  - K3s Agent        │
│  - Prometheus        │   │  - Node Exporter    │
│  - Grafana           │   │  - Cloudflare Agent │
│  - Cloudflare Agent  │   │                     │
└──────────────────────┘   └─────────────────────┘
         │                         │
         └────────────────────┬────┘
                              │
                      ┌───────▼────────┐
                      │  Cloudflare    │
                      │  - DNS         │
                      │  - DDoS Prot.  │
                      │  - Edge Cache  │
                      │  - WAF         │
                      └────────────────┘
```

---

## 💰 Cost Analysis

### Hardware (One-Time)
| Item | Cost | Notes |
|------|------|-------|
| Raspberry Pi 4 (8GB) | $75 | Per node |
| MicroSD Card 64GB | $20 | Per node |
| USB-C Power Supply | $10 | Per node |
| Network Cable | $5 | Per node |
| **Total per Pi** | **~$110** | |
| **For 3 Pis** | **~$330** | |

### Software (Forever Free!)
| Component | Cost | License |
|-----------|------|---------|
| Debian | FREE | Open Source |
| K3s | FREE | Open Source |
| Ansible | FREE | Open Source |
| Prometheus | FREE | Open Source |
| Grafana | FREE | Open Source (no license required) |
| Cloudflare Agent | FREE | Free tier |
| **Total** | **$0** | |

### Estimated Budget

- **Small cluster (3 Pis)**: ~$330 hardware + $0 software = **$330 total**
- **Medium cluster (10 Pis)**: ~$1,100 hardware + $0 software = **$1,100 total**
- **Large cluster (20+ Pis)**: ~$2,200+ hardware + $0 software = **$2,200+ total**

**No recurring subscription costs!**

---

## 🔧 System Requirements

### For Each Raspberry Pi
- Raspberry Pi 4 with 4GB+ RAM (8GB recommended)
- 32GB+ microSD card (64GB+ for production)
- USB-C power supply (2.5A minimum)
- Ethernet cable (WiFi supported but not recommended)

### For Control Machine
- Linux, macOS, or Windows (WSL2)
- Python 3.8+
- Ansible 2.10+
- kubectl
- Git (recommended)

### Network
- Router with DHCP or static IP capability
- 100+ Mbps internet connection recommended
- Access to Cloudflare (free account)

---

## ✨ Deployment Models

### Development/Testing
```yaml
Nodes: 1
Master: localhost
Workers: localhost
Storage: USB drive
Cost: ~$110
TTR: 2-3 hours
```

### Small Production
```yaml
Nodes: 3
Master: 1
Workers: 2
Storage: Local microSD
Cost: ~$330
TTR: 5-9 hours
HA: Optional
```

### Medium Production
```yaml
Nodes: 10
Master: 3 (HA)
Workers: 7
Storage: NAS or cloud backup
Cost: ~$1,100
TTR: 12-24 hours
HA: Recommended
```

---

## 📖 How to Use This Guide

### If You're New to Kubernetes
1. Start with **00_INFRASTRUCTURE_OVERVIEW.md**
2. Understand the architecture
3. Follow guides sequentially
4. Don't skip the testing steps

### If You're Experienced with K3s
1. Review **02_ANSIBLE_SETUP.md** for playbook structure
2. Customize playbooks for your needs
3. Jump to **04_PROMETHEUS_GRAFANA.md** or **05_CLOUDFLARE_AGENT.md**
4. Integrate with your existing infrastructure

### If You Need HA (High Availability)
1. Follow through **03_K3S_DEPLOYMENT.md**
2. Refer to **06_SCALING_GUIDE.md** for HA setup
3. Use keepalived for virtual IP
4. Deploy 3+ master nodes

### If You Need Backups & DR
1. Read **07_BACKUP_RECOVERY.md** first
2. Set up automated backups immediately
3. Test restore procedures monthly
4. Keep backups in multiple locations

---

## 🛠️ File Structure

```
rpi-cluster/
├── README.md                           ← You are here
├── 00_INFRASTRUCTURE_OVERVIEW.md       ← Start here
├── 01_DEBIAN_INSTALLATION.md
├── 02_ANSIBLE_SETUP.md
├── 03_K3S_DEPLOYMENT.md
├── 04_PROMETHEUS_GRAFANA.md
├── 05_CLOUDFLARE_AGENT.md
├── 06_SCALING_GUIDE.md
├── 07_BACKUP_RECOVERY.md
│
├── ansible/
│   ├── inventory/
│   │   └── hosts.ini                 ← Edit with your IPs
│   ├── playbooks/
│   │   ├── 01-common.yml
│   │   └── 02-k3s.yml
│   ├── roles/
│   │   ├── common/
│   │   ├── k3s/
│   │   ├── monitoring/
│   │   └── cloudflare/
│   └── ansible.cfg
│
├── helm-values/
│   ├── prometheus-values.yml
│   └── grafana-values.yml
│
├── k3s-manifests/
│   ├── storage-test.yml
│   ├── test-ingress.yml
│   ├── sample-app.yml
│   ├── prometheus-alerts.yml
│   ├── grafana-ingress.yml
│   ├── cloudflare-agent.yml
│   └── resource-quotas.yml
│
├── scripts/
│   ├── backup-all.sh                 ← Run daily
│   ├── backup-etcd.sh
│   ├── backup-pvcs.sh
│   └── backup-k8s-resources.sh
│
└── docs/
    ├── DISASTER_RECOVERY_RUNBOOK.md
    └── TROUBLESHOOTING.md
```

---

## 🚨 Before You Start

### ✅ Prerequisites Checklist

- [ ] Hardware ordered or available
- [ ] Cloudflare account created (free)
- [ ] Domain registered or planned
- [ ] Control machine ready (Linux/Mac/WSL2)
- [ ] Python 3.8+ installed
- [ ] SSH key pair generated
- [ ] 5-9 hours blocked for initial setup
- [ ] Read this entire README

### ⚠️ Important Notes

1. **No Cost Surprises**: All software is free. Your only costs are hardware.
2. **Not For Production Without Review**: While this guide is comprehensive, adapt it for your specific needs.
3. **Backup Early**: Set up backups before deploying critical applications.
4. **Test Restores**: Regularly test that you can actually restore from backups.
5. **Security**: Change default passwords and enable SSH key authentication.
6. **Network**: Use static IPs for all cluster nodes.
7. **Monitoring**: Set up monitoring BEFORE you need it.

---

## 🆘 Getting Help

### Documentation Links

- **Kubernetes**: https://kubernetes.io/docs/
- **K3s**: https://docs.k3s.io
- **Ansible**: https://docs.ansible.com
- **Prometheus**: https://prometheus.io/docs/
- **Grafana**: https://grafana.com/docs/
- **Cloudflare**: https://developers.cloudflare.com

### Troubleshooting

Each guide includes a "Troubleshooting" section. Common issues:

- **Can't SSH to Pi**: Check IP address and SSH key permissions
- **K3s won't start**: Check cgroups are enabled
- **Nodes not joining**: Verify token and network connectivity
- **Prometheus no data**: Check ServiceMonitor configuration

---

## 📝 License & Attribution

This guide is provided as-is for educational and personal use.

**Open Source Projects Used:**
- K3s (Apache 2.0)
- Prometheus (Apache 2.0)
- Grafana (AGPL 3.0)
- Ansible (GPLv3)
- Cloudflared (Apache 2.0)

---

## 🎓 Learning Path

### If New to Kubernetes
```
1. Read: 00_INFRASTRUCTURE_OVERVIEW.md (30 min)
2. Setup: 01_DEBIAN_INSTALLATION.md (2h)
3. Deploy: 02_ANSIBLE_SETUP.md (1h)
4. Learn: 03_K3S_DEPLOYMENT.md (2h)
5. Deploy: kubectl basics and test apps (1h)
6. Monitor: 04_PROMETHEUS_GRAFANA.md (2h)
7. Secure: 05_CLOUDFLARE_AGENT.md (1h)
8. Protect: 07_BACKUP_RECOVERY.md (1h)
Total: ~11 hours of learning
```

### If Kubernetes Experienced
```
1. Review: 00_INFRASTRUCTURE_OVERVIEW.md (10 min)
2. Adapt: 02_ANSIBLE_SETUP.md to your needs (30 min)
3. Deploy: Run playbooks (2h)
4. Configure: 04_PROMETHEUS_GRAFANA.md (1h)
5. Network: 05_CLOUDFLARE_AGENT.md (30 min)
6. Backup: 07_BACKUP_RECOVERY.md (30 min)
Total: ~4-5 hours
```

---

## 🎯 What's Next?

### After Basic Setup
1. ✅ Deploy your own applications
2. ✅ Set up continuous deployment (CI/CD)
3. ✅ Configure monitoring alerts
4. ✅ Test disaster recovery
5. ✅ Join the Kubernetes community

### To Extend This Guide
1. Add ArgoCD for GitOps
2. Add cert-manager for SSL automation
3. Add Harbor for private container registry
4. Add Istio for service mesh
5. Add Jenkins for CI/CD

---

## 💡 Pro Tips

- **Test Everything First**: Use a single Pi for testing
- **Use Git**: Version control your cluster configuration
- **Document Changes**: Keep notes of customizations
- **Backup Before Changes**: Always backup before major upgrades
- **Monitor Metrics**: Set up alerting for resource usage
- **Update Regularly**: Keep Debian and K3s updated
- **Use Namespaces**: Organize apps by namespace
- **Practice DR**: Test disaster recovery monthly
- **Scale Gradually**: Add nodes one at a time
- **Automate**: Use Ansible for everything repeatable

---

## 🤝 Contributing

Found an error? Want to improve this guide?

1. Test the improvement
2. Document the change
3. Submit feedback
4. Share with community

---

## 📞 Support

This is a community guide. Support comes from:

- **Official Docs**: K3s, Ansible, Prometheus, Grafana
- **Community**: Stack Overflow, GitHub Issues
- **Your Peers**: Local Kubernetes meetups

---

## Last Updated

**2026-04-06**

Version: 1.0 (Complete)

---

## 🎉 Ready to Start?

### Next Step

👉 **Open `00_INFRASTRUCTURE_OVERVIEW.md` to understand the architecture**

Then follow the phases in order. You've got this! 🚀

---

**Questions? Issues? Check the troubleshooting sections in each guide.**

**Good luck building your cluster!**
